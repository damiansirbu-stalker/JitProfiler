JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly, by Damian
Version: 1.0.0-snapshot (xlibs 1.8.3, demonized 20250908)
Changelog: https://github.com/damiansirbu-stalker/JitProfiler/blob/main/doc/changelog

My work:
GitHub: https://github.com/orgs/damiansirbu-stalker/repositories
ModDB: https://www.moddb.com/members/damian-sirbu/addons
Nexus: https://www.nexusmods.com/profile/damiansirbu/mods

My contributions:
X-Ray Monolith: https://github.com/themrdemonized/xray-monolith

JitProfiler finds which of your mods eats performance, and it finds it for you.
It samples the running Lua stack on the engine's own timer, so the overhead is near-zero, the JIT stays on, and one capture covers the whole modpack at once.
There is nothing to select and nothing to suspect in advance.
Point it at the slow scene and read the ranking.
The sampling is jittered, so a mod that runs on a schedule cannot dodge the sampler.

It offers CPU sampling, memory allocation profiling, targeted instrumentation, and a callbacks profiler.
CPU sampling shows where the Lua time goes.
Memory profiling shows which code generates the garbage the collector must clear, the real source of Anomaly's stutter and something script-side profilers cannot measure.
Instrumentation wraps a chosen set of scripts and times each function for its own and total wall-clock, including the engine C beneath a call, the one axis the sampler cannot reach.
Framerate drops while a scan runs, because each wrapped call carries a timer. That is expected, and the timings stay exact.
It records the call graph as it runs, so each function shows its callers and its callees with their time, and a whole-mod scan totals the cost per owning mod.
The callbacks profiler wraps every registered handler over the make_callback dispatch and ranks each by own ms with its callback and owning mod.
So it names which mod hooks a given callback and what each handler costs, engine C included.

Engine C stays one bucket by nature.
The sampler catches that Lua entered the engine, but the C++ routine underneath stays invisible to it.
JitProfiler makes that bucket actionable anyway.
It ranks the Lua call sites that enter engine C and the collector, the engine-C entry points, so you know which of your own calls drive the engine cost.
Those entry points are the map for a native C++ profiler.
Point Optick at the engine paths JitProfiler names.
The Lua layer finds the entry, and the native profiler opens what lies beneath it.

The sampler and the allocation counter are native code in the modded exe, beneath the script layer.
That native core is my own work in xray-monolith. I backported the timer sampler and wrote the allocation profiler, so JitProfiler runs from the C up, one author for the whole stack.
That is why the cost stays near-zero, and why it reaches the allocator and the VM state a script tool cannot.
Every capture names the mod that owns each hot script, so you get a mod name to act on.
When several mods override the same script, JitProfiler names the one that wins your load order, the copy the game actually loaded and ran.
It reads this from the MO2 virtual filesystem, so the name always matches what executed.

Requirements:
Anomaly 1.5.3
A modded-exes build with the JitProfiler primitives.
jit.profile and jit.allocprof are in the official 2026.9.12 modded exes.
jit.util.gcstat, which drives the GC-health readout, is not yet.
For the full feature set, use the preview exes from my fork:
https://github.com/damiansirbu-stalker/fork-xray-monolith/releases/tag/2026.9.12-mt-jitprofiler
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
start_cpu(interval_ms, depth) overrides the 5 ms and depth 64 defaults. start_cpu(1) samples more often, start_cpu(5, 32) traces shallower and faster.

The allocation profile shows which code generates GC garbage, with the JIT off for the whole capture.
Expect a hard FPS drop that scales with the modpack's script load. The byte counts stay exact, so capture short at the spot you care about:
```
run_string JitProfiler.start_alloc()
-- play 30-60 seconds
run_string JitProfiler.stop_alloc()
```
Allocation counts exact bytes, so only depth is tunable, as start_alloc(20). A long session auto-writes numbered snapshots. Call JitProfiler.write_snapshot() to force one.
Framerate drops sharply while the allocation profile runs, because the JIT is off for the capture. That is expected, and the byte counts stay exact regardless of framerate.

Fold the baseline (anomaly, modded exes, engine) to focus on your mods:
```
run_string JitProfiler.set_fold_baseline(true)
```
Then stop as usual.

In-game panel:
An ImGui panel does everything without the console. Open the ImGui overlay (default F11), then pick JitProfiler in the menu bar.
The panel carries a CPU tab, a MEM tab, an INSTRUMENT tab, and a CALLBACKS tab, and a running capture locks the others.
CPU and MEM hold the sampling captures with the ranked views.
BY MOD and BY SCRIPT show Own and Total side by side, every column sortable, while BY LEAF and BY ROOT drill into frames, and selecting a row reads its callers and callees.
The CPU tab adds the VM-state split and the engine-C entry-points list.
The MEM tab adds a live GC-health strip with the heap toward the next collection, the live estimate, the debt, and the collection rate.
INSTRUMENT holds the targeted mode.
Add scripts from the BROWSE modlist (a + per script, or +all on a mod header to instrument the whole mod) or the + on a by-script row.
Then run and read the per-function own and total time, sortable by any column, with avg, min, and max on hover and a frame-budget bar.
Select a function to read its callers and callees with their time.
CALLBACKS arms the callbacks profiler and lists every registered handler ranked by own ms, with its callback and owning mod.
Its BROWSE subtab lists every registered callback with a picker toggle. Picked callbacks arm alone, and none picked arms all.
On the multi-thread exe a Parallel GC toggle gives a clean CPU garbage-collector read.
A small corner banner shows while a capture runs.

Configuration (MCM):
The JitProfiler MCM page holds the capture defaults. These are the sample interval and stack depth, plus the auto-snapshot threshold and the report fold.
The panel and the console commands use these defaults, and a console argument to start_cpu or start_alloc overrides the matching value for that capture.
The MCM also has an auto-stop duration. Set it above 0 and a running capture stops itself after that many seconds, a safety limit for the allocation profile.

The reports go to appdata/logs/:
```
jitprofiler_cpu_<timestamp>.txt     ranked text report
jitprofiler_cpu_<timestamp>.folded  flamegraph for speedscope.app
jitprofiler_mem_<timestamp>.txt     ranked text report
jitprofiler_mem_<timestamp>.folded  flamegraph
jitprofiler_inst_<timestamp>.txt    instrumentation report (own, total, avg, min, max, per-mod subtotals, call graph)
jitprofiler_callbacks_<timestamp>.txt  callbacks report (per handler: callback, owner, own, total; per-callback rollup)
```

The report opens with a VM-state split, how much Lua time is JIT-compiled, interpreted, in C/engine calls, in the garbage collector, and in the JIT compiler.
A GC line follows with the heap, the live estimate, the debt, and the collections per second.
The GC share flags allocation pressure without a separate run.
Then the engine-C entry points name the Lua call sites that drive the engine bucket.
The ranked views follow, each carrying Own (where the code ran) and Total (on the stack anywhere): BY MOD, BY SCRIPT, BY LEAF, BY ROOT, and framework vs handlers.
The allocation report carries the same views by bytes.

The .folded files open at speedscope.app, a flamegraph viewer that runs in your browser.
Drag a .folded file onto the page to explore the stacks.
Nothing is uploaded, and the file never leaves your machine.

On a stock exe without the primitives, the commands print which build is needed and do nothing else.

How It's Built:

The C core lives in xray-monolith, a backport of LuaJIT's jit.profile timer sampler into the 2.0.4 exe without GC64, so saves stay compatible.
The jit.allocprof allocation counter runs on top, both native in the exe beneath the script layer.
It samples on the engine's own timer with the JIT on, so overhead stays near-zero. One capture covers the whole modpack, with nothing to select in advance.
The sampler jitters its interval on an exponential draw, so a scheduled mod cannot phase-lock and dodge it.
It reads what a script profiler cannot: the VM-state split, and the exact allocation bytes at the allocator seam that are the real source of GC stutter.
Per-mod attribution runs through the MO2 virtual filesystem, resolving each merged script to the load-order winner that ran, verified on real conflicts.
The design follows the best in class: SpeedScope flamegraphs, an in-game ImGui panel, and the engine-C entry points mapped for a native profiler like Optick.
Every commit runs the full pipeline locally and in CI: luacheck, a Selene build compiled for STALKER with flags the public build lacks, and a load test that runs every script against engine stubs.
Rule layers then check crash safety, hotpath cost, engine correctness, complexity, architecture contracts, security, and the docs.
It depends on no other mod, not even my own. The only shared layers are X-Ray and xlibs.

[Screenshot: JitProfiler under a live CPU and allocation capture]
Project Health: https://damiansirbu-stalker.github.io/JitProfiler/

Credits:
The engine core is my own contribution to xray-monolith, the jit.profile sampler backported from LuaJIT into the modded exe plus the jit.allocprof allocation profiler on top of it.
LuaJIT and jit.profile are by Mike Pall.
Built for the themrdemonized modded exes (themrdemonized/xray-monolith).

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "JitProfiler by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.

Diagnostics and reporting:
Report at https://github.com/damiansirbu-stalker/JitProfiler/issues/new/choose or the EFP, Anomaly, and Zona Discord. Include repro steps, engine build, modlist, load order, xray.log, and the debug log.
