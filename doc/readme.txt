JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly, by Damian
Version: next (xlibs 1.8.3, demonized 20250908)
GitHub: https://github.com/damiansirbu-stalker/JitProfiler
Changelog: https://github.com/damiansirbu-stalker/JitProfiler/blob/main/doc/changelog
Report bugs and suggestions at https://github.com/damiansirbu-stalker/JitProfiler/issues

Alife Collection:
AlifeAmbience: https://github.com/damiansirbu-stalker/AlifeAmbience
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeCompanions: https://github.com/damiansirbu-stalker/AlifeCompanions
AlifeDiegetic: https://www.moddb.com/mods/stalker-anomaly/addons/diegetic-audio-control-100
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeSpooks: https://github.com/damiansirbu-stalker/AlifeSpooks
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics
FurnitureFuel: https://github.com/damiansirbu-stalker/FurnitureFuel
JitProfiler: https://github.com/damiansirbu-stalker/JitProfiler
TestZone: https://github.com/damiansirbu-stalker/TestZone
xlibs: https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001

JitProfiler finds which of your mods eats performance, and it finds it for you.
It samples the running Lua stack on the engine's own timer, so the overhead is near-zero, the JIT stays on, and one capture covers the whole modpack at once.
There is nothing to select and nothing to suspect in advance.
Point it at the slow scene and read the ranking.
The sampling is jittered, so a mod that runs on a schedule cannot dodge the sampler.

It profiles CPU and memory.
CPU sampling shows where the Lua time goes.
Memory profiling shows which code generates the garbage the collector must clear, the real source of Anomaly's stutter and something script-side profilers cannot measure.

The sampler and the allocation counter are native code in the modded exe, beneath the script layer.
That native core is my own work in xray-monolith. I backported the timer sampler and wrote the allocation profiler, so JitProfiler runs from the C up, one author for the whole stack.
That is why the cost stays near-zero, and why it reaches the allocator and the VM state a script tool cannot.
Every capture names the mod that owns each hot script, so you get a mod name to act on.
When several mods override the same script, JitProfiler names the one that wins your load order, the copy the game actually loaded and ran.
It reads this from the MO2 virtual filesystem, so the name always matches what executed.

Requirements:
Anomaly 1.5.3
A demonized modded-exes build with the JitProfiler primitives (jit.profile, jit.allocprof).
xlibs (used for logging).
Launch with -dbg (MO2 launch arguments) so the console accepts run_string.

Install (MO2):
1. Install xlibs and JitProfiler. Load order does not matter.

Usage (in-game console, ~):

The CPU profile shows which code eats Lua time, with the JIT on:
```
run_string JitProfiler.start_cpu()
-- play the scene 30-60 seconds
run_string JitProfiler.stop_cpu()
```
start_cpu(interval_ms, depth) overrides the 5 ms and depth 12 defaults. start_cpu(1) samples more often, start_cpu(5, 20) traces deeper.

The allocation profile shows which code generates GC garbage, with the JIT off and slower:
```
run_string JitProfiler.start_alloc()
-- play 30-60 seconds
run_string JitProfiler.stop_alloc()
```
Allocation counts exact bytes, so only depth is tunable, as start_alloc(20). A long session auto-writes numbered snapshots. Call JitProfiler.write_snapshot() to force one.

Fold the vanilla baseline to focus on your mods:
```
run_string JitProfiler.set_fold_anomaly(true)
```
Then stop as usual.

In-game panel:
An ImGui panel does the same without the console. Open the ImGui overlay (default F11), then pick JitProfiler in the menu bar.
Start or stop the CPU or memory capture from the panel.
Switch the ranked view (by mod, leaf, script, root, framework), and select a row to read the callers and callees of that frame.
A small corner banner shows while a capture runs.

Configuration (MCM):
The JitProfiler MCM page holds the capture defaults for the sample interval, stack depth, report rows, auto-snapshot threshold, and the report fold.
The panel and the console commands use these defaults, and a console argument to start_cpu or start_alloc overrides the matching value for that capture.
The MCM also has an auto-stop duration. Set it above 0 and a running capture stops itself after that many seconds, a safety limit for the allocation profile.

The reports go to appdata/logs/:
```
jitprofiler_cpu_<timestamp>.txt     ranked text report
jitprofiler_cpu_<timestamp>.folded  flamegraph for speedscope.app
jitprofiler_mem_<timestamp>.txt     ranked text report
jitprofiler_mem_<timestamp>.folded  flamegraph
```

The report opens with a VM-state split, showing how much Lua time is JIT-compiled, interpreted, in C/engine calls, in the garbage collector, and in the JIT compiler.
The GC share flags allocation pressure without a separate run.
Then the ranked views: BY MOD self (where the code ran) and total (every mod on the stack), BY LEAF (hot function), BY SCRIPT, BY ROOT (outermost frame driving the cost), and framework vs handlers.
The allocation report carries the same views by bytes.

The .folded files open at speedscope.app, a flamegraph viewer that runs in your browser.
Drag a .folded file onto the page to explore the stacks.
Nothing is uploaded, and the file never leaves your machine.

On a stock exe without the primitives, the commands print which build is needed and do nothing else.

Credits:
The engine core is my own contribution to xray-monolith, the jit.profile sampler backported from LuaJIT into the modded exe plus the jit.allocprof allocation profiler on top of it.
LuaJIT and jit.profile are by Mike Pall.
Built for the themrdemonized modded exes (themrdemonized/xray-monolith).

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "JitProfiler by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in the LICENSE file and on GitHub.
