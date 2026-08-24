JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly, by Damian
Version: next
GitHub: https://github.com/damiansirbu-stalker/JitProfiler
Changelog: https://github.com/damiansirbu-stalker/JitProfiler/blob/main/doc/changelog
Bugs, suggestions: https://github.com/damiansirbu-stalker/JitProfiler/issues

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

JitProfiler samples the running Lua stack on the engine's own timer. The JIT stays on, overhead is near-zero, and one capture covers the whole modpack with no wrapping and no module selection. It is sampling, not instrumentation. It reads two primitives backported into the demonized engine: jit.profile (CPU) and jit.allocprof (allocation).

Requirements:
Anomaly 1.5.3
A demonized modded-exes build with the JitProfiler primitives (jit.profile, jit.allocprof) baked in.
Launch with -dbg (MO2 launch arguments) so the console accepts run_string.

Install (MO2):
1. Install JitProfiler. Load order does not matter.

Usage (in-game console, ~):
CPU profile (which code eats Lua time; the JIT stays on):
  run_string JitProfiler.start()
  ...play the scene 30-60s...
  run_string JitProfiler.stop()

Allocation profile (which code generates GC garbage; JIT off, slower):
  run_string JitProfiler.astart()
  ...play 30-60s...
  run_string JitProfiler.astop()

Output (appdata/logs/):
  JitProfiler_cpu_report.txt     ranked text report
  JitProfiler_cpu.folded         flamegraph for https://www.speedscope.app
  JitProfiler_alloc_report.txt
  JitProfiler_alloc.folded

The report opens with a VM-state split: how much Lua time is JIT-compiled, interpreted, in C/engine calls, in the garbage collector, and in the JIT compiler. The GC share flags allocation pressure without a separate run. Then ranked views: by leaf (hot function), by script, by root (the outermost frame driving the cost), and framework vs handlers. The allocation report carries the same views by bytes.

On a stock exe without the primitives, the commands print which build is needed and do nothing else.

Credits:
LuaJIT and jit.profile by Mike Pall. Profiler primitives backported into the demonized modded exes (themrdemonized/xray-monolith).

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "JitProfiler by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.
