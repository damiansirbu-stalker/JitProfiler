# JitProfiler Architecture

Engine-native LuaJIT sampling profiler for STALKER Anomaly.
It has an engine layer and a mod layer.

The engine layer is C, compiled into the demonized exe.
`jit.profile` is LuaJIT 2.1's timer stack sampler, backported into 2.0.4 without GC64 so saves stay compatible.
`jit.allocprof` accounts exact per-allocation bytes on the allocator seam.
Both are raw capability delivered in the engine, not in this mod.

The mod layer is Lua, `JitProfiler.script`, the front-end.
It drives the primitives from the console and aggregates the samples.
It resolves each script to its owning mod, then writes the reports and SpeedScope flamegraphs.
It logs through xlibs `xlog`.

## CPU: sampling, not instrumentation
`jit.profile` fires a timer on a dedicated thread and records the running stack and the VM state on every fire, with the JIT compiled.
Overhead stays near-zero.
One capture covers the whole modpack with no per-call wrapper and no module selection.
Interval and stack depth are per-capture arguments to `start_cpu`.

The interval is jittered. Each gap is a random draw from an exponential around the mean, so the sample times are never periodic.
A mod running on a fixed schedule then cannot phase-lock to the sampler, so a capture sees its true share.
The allocation profiler already draws its sample distance the same way, so both samplers are free of fixed-interval aliasing.

A sample that lands outside Lua execution is only delivered at the next bytecode dispatch, so its weight would land on the next Lua frame, usually a binder freshly entered from the engine.
`collect_cpu_sample` therefore reads the sample's VM state and prepends a `[C]`, `[GC]`, or `[JIT]` pseudo-leaf to the captured stack when the state was not Lua.
The booking then splits on the dumped depth.
A pseudo-leaf over a single frame is a fresh entry, so the preceding C time is unattributable trunk work and books to the engine owner.
A pseudo-leaf over 2 or more frames means Lua was mid-execution around a C or GC call, so the weight books to the resuming frame, the one that made the call.
The one approximation: a C call that itself enters Lua (a callback fired from inside an engine API) charges the entered handler, the nearest seam the dump can see.
Either way a script is never credited for trunk time that ran before it was entered.

## Allocation: exact bytes, JIT off
`jit.allocprof` counts every allocation at the allocator seam.
It drains the pending bytes to the current stack on each bytecode instruction, with the JIT off for the session.
Byte totals are exact.
Attribution is instruction-granular.

## Instrumentation: self and total over a script set
The sampler names where the frame sits but cannot enter the engine C beneath a Lua call, so a mod that spends its cost inside an engine routine reads as engine time.
Instrumentation closes that gap by wrapping every function of a chosen SET of scripts.
`add_target` puts a script in the set, `add_mod_targets` adds every script a mod owns, and the set holds many scripts at once.
`start_scan` wraps every function of every set script with a timer over xlibs `xprofiler` on the engine `profile_timer`, and `stop_scan` restores every original and writes the report.
The timer spans the whole call, so the engine C the function triggers is inside its number, the axis the JIT-on sampler cannot reach.
Each call pushes a pooled stack frame; on return the frame banks total, its whole span, and self, total minus the time charged by nested wrapped children.
Self is exclusive and correct across nesting and cross-mod calls; total is inclusive.
Each function shows its call count, self ms, total ms, and us per call, ranked by self, with self ms per frame as the budget.
The wrapper runs the original under pcall, so a Lua error still closes the frame and re-raises unchanged, and a recursive function books self correctly while total stays inclusive by definition.
It is opt-in and mutually exclusive with the CPU and allocation captures, because the wrapper tax and the JIT blinding are the per-call cost the sampler exists to avoid.
The natural loop runs the sampler first, then instruments whichever scripts the `[C]` rows name.

## Per-mod attribution
GAMMA merges every mod into one `gamedata/scripts`, so the script path never names the mod.
For a mod's own file the resolver reads its real backing path through MO2/USVFS, over `GetFinalPathNameByHandle` on a LuaJIT FFI handle, and takes the `mods\<X>` folder as the owner.
A file no mod owns is classified by name against two baked sets, regenerated from the unpacked Anomaly tree and the demonized overlay.

| name matches | owner |
|---|---|
| stock Anomaly set | anomaly |
| demonized overlay set | modded exes |
| neither set | unknown |

Because the sets ship with the mod, the classification is install-independent and never assumes an unresolved script is vanilla.
A `[C]` frame reads as engine.
The resolve runs at report time and caches per script. An absent FFI drops a mod's own file to unknown, and the name sets still classify the rest.

The owner is always the mod that wins the load order.
When several mods override one script, only the MO2 priority winner sits on disk behind the virtual path, and that is the copy the game loaded and ran.
A mod serving a stock-named script is tagged as an override of that file, so a near-identical copy is not misread as the mod's own cost.
Verified on real 3-way conflicts: visual_memory_manager resolved to the active winner over a lower-priority override and a disabled one, and the same held for xr_combat_ignore and zz_item_artefact.

## Metrics and views
Per unit, JitProfiler reports two numbers, Own and Total.
Total is inclusive. A mod or script gets credit whenever it appears anywhere in a sample, counted once per sample. It does not sum to 100%.
Own is exclusive, the innermost frame when the sample fired.
Total is the primary ranking, per mod and per script, and Own is the second column.
The views are BY MOD, BY SCRIPT, BY LEAF, BY ROOT, FRAMEWORK vs handlers, and the VM-state split for CPU.
Each capture also reports the deepest stack it saw and how often a stack reached the depth cap, so an under-counted Total is visible.
The fold toggle drops the baseline rows (anomaly, modded exes, engine, unknown) from the owned views, leaving only mods.

## Output
`appdata/logs/jitprofiler_{cpu,mem}[_snapN]_<timestamp>.txt` is the ranked text.
`jitprofiler_{cpu,mem}[_snapN]_<timestamp>.folded` is the SpeedScope collapsed-stacks flamegraph.
The timestamp is the capture's wall-clock time, so successive runs never overwrite.
Both are ASCII only.

## In-game panel
`jitprofiler_ui.script` registers a panel and a menu entry through the base ImGui Groups API.
The panel draws in the Main group, so it appears while the F11 ImGui overlay is open.
Two top tabs split the panel, SAMPLING and INSTRUMENTATION.
Under SAMPLING a CPU/MEM toggle shows the retained last capture through `get_last_capture`, and the two never run at once.
A view selector switches the table between by mod, by script, by leaf, and by root; each column header sorts and a filter box narrows the rows.
The by-script rows carry a per-row button that adds or removes the script from the instrumentation set.
Under INSTRUMENTATION a start and stop control arms the whole set, an overhead line shows the wrapped-function count, and a checkbox tree grouped by mod with a search box picks the targets.
The results table ranks each function by self, with total, calls, and us per call.
Each row's owner is tinted by a stable per-mod colour, hovering shows the full frame, and a selected frame ranks its callers and callees through `compute_neighbors`.
The CPU tab adds the VM-state strip, one bar per state (native, interpreter, C, GC, JIT compile), each explained on hover.
Number columns render in white JetBrains Mono so digits line up as a column, and each cost bar carries a single-hue amber heat shade by share.
A fold-baseline toggle drops the non-mod rows, leaving only mods.
A corner banner in the Unique group renders every frame and shows only while a capture runs.
The chunk executes twice, so registration and the retained capture anchor to `_G` singletons.

## Settings and MCM
Capture settings live in the panel, not in MCM.
`JitProfiler.script` keeps them as session state anchored to `_G`, edited from the panel through `get_setting` and `set_setting`.
They cover interval, depth, report rows, the auto-snapshot threshold, auto-stop, and fold.
A console argument to `start_cpu` or `start_alloc` overrides one capture, and nothing is persisted.
An auto-stop duration above 0 ends a running capture from the `update_capture` heartbeat and writes its report, so a forgotten allocation capture never holds the JIT off past the limit.

`jf_mcm.script` is an about page through `xmcm.create_config`.
It carries the description, the usage and SpeedScope notes, the version and compatibility footer from `_jitprofiler_deps.platform_functor`, and one persistent toggle, `show_imgui`.
The panel, menu, and banner read `show_imgui` through a cached flag and draw nothing while it is off, so JitProfiler leaves the ImGui menu bar.

## Limitations
- Inclusive counts and the flamegraph are bounded by the captured stack depth. A frame beyond the depth is not counted, so raise depth to trace deeper.
- Engine C time is measured as one bucket, because the sampler cannot see which C routine ran.
  The `[C]` pseudo-leaf shows the Lua stack it re-entered from, and nothing more.
  Cost a mod adds through data (populations, per-NPC config) surfaces as engine time, never under the mod's name.
- The CPU JIT-compiled share is a floor, because phase-1 sampling is interpreter-anchored and under-counts JIT traces.
- Allocation attribution is instruction-granular, so bytes from a C or GC path are credited to the next Lua frame.
- Per-mod attribution needs an MO2/USVFS install. Elsewhere it degrades to script level.
- CPU and allocation are mutually exclusive. Each refuses to start while the other runs.

## Requires
A demonized build exposing `jit.profile` and `jit.allocprof`, and xlibs for `xlog`.
On a stock exe the mod detects the missing primitives and stubs the commands.
