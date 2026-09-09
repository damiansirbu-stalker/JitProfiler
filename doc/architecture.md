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

## Allocation: exact bytes, JIT off
`jit.allocprof` counts every allocation at the allocator seam.
It drains the pending bytes to the current stack on each bytecode instruction, with the JIT off for the session.
Byte totals are exact.
Attribution is instruction-granular.

## Per-mod attribution
GAMMA merges every mod into one `gamedata/scripts`, so the script path never names the mod.
The resolver reads each hot script's real backing path through MO2/USVFS, over `GetFinalPathNameByHandle` on a LuaJIT FFI handle.
It takes the `mods\<X>` folder as the owner.
A loose base file or a packed `db0` reads as anomaly.
The resolve runs at report time.
Each result caches per script.
An absent FFI degrades every owner to unknown.

The owner is always the mod that wins the load order.
When several mods override one script, only the MO2 priority winner sits on disk behind the virtual path, and that is the copy the game loaded and ran.
The resolver opens the virtual path and MO2 hands back the winner's real file, so the name matches what executed. Priority stays MO2's to decide.
Verified on real 3-way conflicts: visual_memory_manager resolved to the active winner over a lower-priority override and a disabled one, and the same held for xr_combat_ignore and zz_item_artefact.

## Metrics and views
Per unit, JitProfiler reports the pprof flat and cum metrics.
SELF is the flat metric, exclusive, the leaf frame when the sample fired.
TOTAL is the cum metric, inclusive, any frame on the stack counted once per sample, so it does not sum to 100%.
The views are BY MOD self and total, BY LEAF, BY SCRIPT, BY ROOT, FRAMEWORK vs handlers, and the VM-state split for CPU.
The fold toggle drops the anomaly baseline from the owned views.

## Output
`appdata/logs/jitprofiler_{cpu,mem}[_snapN]_<timestamp>.txt` is the ranked text.
`jitprofiler_{cpu,mem}[_snapN]_<timestamp>.folded` is the SpeedScope collapsed-stacks flamegraph.
The timestamp is the capture's wall-clock time, so successive runs never overwrite.
Both are ASCII only.

## In-game panel
`jitprofiler_ui.script` registers a panel and a menu entry through the base ImGui Groups API.
The panel draws in the Main group, so it appears while the F11 ImGui overlay is open.
It reads the retained last CPU and last allocation capture through `get_last_capture`.
The panel drives each capture from its own button, and the two never run at once.
A view selector switches the table between by mod, leaf, script, root, and framework.
A selected frame ranks its callers and callees from the retained leaf-first stacks through `compute_neighbors`.
The CPU tab adds the VM-state strip. A fold-anomaly toggle drops the vanilla baseline from the view.
A corner banner in the Unique group renders every frame and shows only while a capture runs.
The chunk executes twice, so registration and the retained capture anchor to `_G` singletons.

## MCM
`jf_mcm.script` adds a flat MCM page through `xmcm.create_config` for the capture defaults.
`JitProfiler.script` reads `jf_mcm.cfg` at capture time through `get_mcm_number`.
An auto-stop duration above 0 ends a running capture from the `update_capture` heartbeat and writes its report.
A forgotten allocation capture then never holds the JIT off past the limit.
A console argument to `start_cpu` or `start_alloc` still overrides the matching value for that call.
The console `set_fold_anomaly` overrides the report fold for the session, and otherwise the MCM fold applies.
The page also carries the shared version and compatibility footer from `_jitprofiler_deps.platform_functor`.

## Limitations
- Inclusive counts and the flamegraph are bounded by the captured stack depth. A frame beyond the depth is not counted, so raise depth to trace deeper.
- The CPU JIT-compiled share is a floor, because phase-1 sampling is interpreter-anchored and under-counts JIT traces.
- Allocation attribution is instruction-granular, so bytes from a C or GC path are credited to the next Lua frame.
- Per-mod attribution needs an MO2/USVFS install. Elsewhere it degrades to script level.
- CPU and allocation are mutually exclusive. Each refuses to start while the other runs.

## Requires
A demonized build exposing `jit.profile` and `jit.allocprof`, and xlibs for `xlog`.
On a stock exe the mod detects the missing primitives and stubs the commands.
