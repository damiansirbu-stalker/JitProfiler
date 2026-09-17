# JitProfiler Architecture

Engine-native LuaJIT sampling profiler for STALKER Anomaly.
It has an engine layer and a mod layer.

The engine layer is C, compiled into the demonized exe.
`jit.profile` is LuaJIT 2.1's timer stack sampler, backported into 2.0.4 without GC64 so saves stay compatible.
`jit.allocprof` accounts exact per-allocation bytes on the allocator seam.
`jit.util.gcstat` reads the live GC counters (total, threshold, estimate, debt) for the GC-health strip.
These are raw capability delivered in the engine, not in this mod.

The mod layer is Lua. `JitProfiler.script` is the front-end.
It drives the primitives from the console and aggregates the samples. It also resolves each script to its owning mod.
`jitprofiler_report.script` is the presentation layer.
It renders the computed views into ranked text reports, SpeedScope flamegraphs, and the call-graph neighbors, so the front-end holds only capture and resolution.
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

## Instrumentation: own and total over a script set
The sampler names where the frame sits but cannot enter the engine C beneath a Lua call, so a mod that spends its cost inside an engine routine reads as engine time.
Instrumentation closes that gap by wrapping every function of a chosen SET of scripts.
`add_target` puts a script in the set, `add_mod_targets` adds every script a mod owns, and the set holds many scripts at once.
`start_scan` wraps every function of every set script with a timer over xlibs `xprofiler` on the engine `profile_timer`. `stop_scan` restores every original and writes the report.
The timer spans the whole call, so the engine C the function triggers is inside its number, the axis the JIT-on sampler cannot reach.
Each call pushes a pooled stack frame. On return the frame banks total, its whole span, and own, total minus the time charged by nested wrapped children.
Own is exclusive and correct across nesting and cross-mod calls. Total is inclusive.
Each function shows its call count, own ms, total ms, and us per call, ranked by own, with own ms per frame as the budget.
The wrapper runs the original under pcall.
A Lua error still closes the frame, then re-raises to the caller unchanged.
A recursive function books own correctly, and total stays inclusive by definition.
It is opt-in and mutually exclusive with the CPU and allocation captures, because the wrapper tax and the JIT blinding are the per-call cost the sampler exists to avoid.
The natural loop runs the sampler first, then instruments whichever scripts the `[C]` rows name.
It also records the call graph.
Each wrapped call books its span to a parent-to-child edge, so the report shows where a function's total goes and who drove it.
A whole-mod scan totals the cost per owning mod.

## Callbacks: per-handler over the dispatch
Every script callback flows through one chokepoint, `make_callback` in axr_main, which dispatches each registered handler in priority order.
`start_callbacks` reads that file-local `intercepts` table by upvalue and swaps each function handler in place for a timing wrapper, keeping its priority.
`make_callback` is never reimplemented, so dispatch order and semantics stay untouched.
Each wrapper times its handler inclusive of the engine C it triggers, banks own and total on a pooled stack like the scan, and re-raises a handler error unchanged.
`stop_callbacks` swaps every original back and writes the report.
A level change or actor destroy force-restores first, so no wrapper orphans.
`spairs` snapshots the handler keys, so a mid-dispatch restore is safe.
A picker set narrows the capture: picked callback names arm alone, and an empty pick arms every callback.
The report ranks every handler by own ms with its callback and owning mod, plus a per-callback rollup, answering which mod hooks a callback and what each handler costs, engine C included.
It is opt-in, mutually exclusive with the other captures, and goes inert on a build where the intercepts upvalue is not reachable.

## Per-mod attribution
GAMMA merges every mod into one `gamedata/scripts`, so the script path never names the mod.
The resolver reads each script's real backing path through MO2/USVFS, over `GetFinalPathNameByHandle` on a LuaJIT FFI handle. A `mods\<X>` path takes X as the owner.
A path the open cannot reach is packed in a db, classified by name against two baked sets regenerated from the unpacked Anomaly tree and the demonized overlay.
The match is case-insensitive because the VFS lowercases names.
A loose path outside `mods\` that neither set names is a file no mod packaged, so it takes the folder holding its `gamedata` (the MO2 `overwrite`, or the install root) as `loose: <folder>`.

| resolves to | owner |
|---|---|
| a `mods\<X>` path | mod: X |
| the demonized overlay set | modded exes |
| the stock Anomaly set | anomaly |
| a loose path in neither set | loose: `<folder>` |
| none of the above | unknown |

A frame's file comes from LuaJIT `short_src`, which front-truncates a path past its 60-byte buffer to `...tail`. The resolver strips that marker, so a long-named script keys to its real basename.
Because the sets ship with the mod, the classification is install-independent and never assumes an unresolved script is vanilla.
A `[C]` frame reads as engine. A `[string]` chunk from loadstring reads as loadstring unless a real Lua frame below it owns the cost.
The resolve runs at report time and caches per script. An absent FFI drops resolution to the name sets alone.

The owner is always the mod that wins the load order.
When several mods override one script, only the MO2 priority winner sits on disk behind the virtual path, and that is the copy the game loaded and ran.
A mod serving a stock-named script is tagged as an override of that file, so a near-identical copy is not misread as the mod's own cost.
Verified on real 3-way conflicts: visual_memory_manager resolved to the active winner over a lower-priority override and a disabled one, and the same held for xr_combat_ignore and zz_item_artefact.

## Metrics and views
Per unit, JitProfiler reports two numbers, Own and Total.
Total is inclusive. A mod or script gets credit whenever it appears anywhere in a sample, counted once per sample. It does not sum to 100%.
Own is exclusive, the innermost frame when the sample fired.
Total is the primary ranking, per mod and per script, and Own is the second column.
The views are BY MOD, BY SCRIPT, BY LEAF, BY ROOT, and FRAMEWORK vs handlers.
CPU adds the VM-state split and the engine-C entry points, the Lua call sites that drive the engine bucket and the map for a native profiler like Optick.
Each capture also reports the deepest stack it saw and how often a stack reached the depth cap, so an under-counted Total is visible.
The fold toggle drops the baseline rows (anomaly, modded exes, engine, unknown) from the owned views, leaving only mods.

## Output
`appdata/logs/jitprofiler_{cpu,mem}[_snapN]_<timestamp>.txt` is the ranked text.
The instrumentation and callbacks modes write `jitprofiler_inst_<timestamp>.txt` and `jitprofiler_callbacks_<timestamp>.txt`.
`jitprofiler_{cpu,mem}[_snapN]_<timestamp>.folded` is the SpeedScope collapsed-stacks flamegraph.
Every text report is self-contained.
The sampling reports close with a call graph (each top leaf with its callers toward root and callees toward leaf).
The instrumentation and callbacks reports carry per-call avg, min, and max, so nothing the panel shows lives only in the panel.
The timestamp is the capture's wall-clock time, so successive runs never overwrite.
Both are ASCII only.

## In-game panel
`jitprofiler_ui.script` registers a panel and a menu entry through the base ImGui Groups API.
The panel draws in the Main group, so it appears while the F11 ImGui overlay is open.
The top tabs are CPU, MEM, INSTRUMENT, and CALLBACKS. A running capture locks the others, so no two run at once.
CPU and MEM show the retained last capture through `get_last_capture`.
A view selector switches the table between by mod, by script, by leaf, and by root. Each column header sorts, and a filter box narrows the rows.
The by-script rows carry a per-row button that adds or removes the script from the instrumentation set.
Under INSTRUMENTATION a start and stop control arms the whole set, and an overhead line shows the wrapped-function count.
Two sub-views split the screen.
SELECTED lists the working set, each target removable.
BROWSE is a modlist grouped by mod with a search box, where a + adds a script and a +all on the mod header adds every script that mod owns through add_mod_targets.
The results table ranks each function by own with an own-share bar, plus own ms, total ms, and calls, each column sortable on click.
Avg, min, and max per call show on row hover and in the text report.
A frame-budget bar reads own ms per frame against a 60fps frame.
Number columns right-align through CalcTextSize, so magnitude scans down the column.
Each row's owner is tinted by a stable per-mod colour, hovering shows the full frame, and a selected frame ranks its callers and callees through `compute_neighbors`.
The CPU tab adds the VM-state strip, one bar per state (native, interpreter, C, GC, JIT compile) each explained on hover.
Then the engine-C entry points, the Lua call sites that drive the engine bucket.
The CALLBACKS tab arms `start_callbacks` and ranks every registered handler over the make_callback dispatch by own ms, with its callback and owning mod.
A BROWSE subtab lists every registered callback with its handler count and a picker toggle.
A dim line under a running MEM or wrapping capture notes the FPS drop and that accuracy holds.
The MEM view carries a live GC-health strip from `jit.util.gcstat`, a bar for the heap toward the next collection plus the live estimate and debt, hidden when the exe lacks the getter.
On the multi-thread exe a Parallel GC toggle flips the `lua_parallel_gc` cvar, so a CPU capture reads a clean G share.
The panel renders in white JetBrains Mono when the font is present, so digits line up as a column, over a blue accent scheme. Each cost bar carries a single-hue blue heat shade by share.
A fold-baseline toggle drops the non-mod rows, leaving only mods.
A corner banner in the Unique group renders every frame and shows only while a capture runs.
The chunk executes twice, so registration and the retained capture anchor to `_G` singletons.
Measured: unanchored, the `actor_on_update` heartbeat and the console entry points hold separate state tables and `scan.frames` reads 0. Anchored, they share one and frames track ticks.

## Settings and MCM
Capture settings live in the panel, not in MCM.
`JitProfiler.script` keeps them as session state anchored to `_G`, edited from the panel through `get_setting` and `set_setting`.
They cover interval, depth, the auto-snapshot threshold, auto-stop, and fold.
A console argument to `start_cpu` or `start_alloc` overrides one capture, and nothing is persisted.
An auto-stop duration above 0 ends a running capture from the `update_capture` heartbeat and writes its report, so a forgotten allocation capture never holds the JIT off past the limit.

`jf_mcm.script` is an about page through `xmcm.create_config`.
It carries the description, the usage and SpeedScope notes, the version and compatibility footer from `_jitprofiler_init.get_platform_functor`, and one persistent toggle, `show_imgui`.
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
- Instrumentation assumes strict call nesting. A wrapped function that yields a coroutine mid-call desyncs the timer stack until stop_scan restores it.
- Instrumentation wraps a script's module-table functions.
  A function reached through a saved reference (a registered callback, a stored upvalue, a local alias) still calls the original and is not timed.
  The callbacks mode covers the dispatch case.

## Requires
A demonized build exposing `jit.profile` and `jit.allocprof` (and `jit.util.gcstat` for the GC-health strip), and xlibs for `xlog`.
On a stock exe the mod detects the missing primitives and stubs the commands.
