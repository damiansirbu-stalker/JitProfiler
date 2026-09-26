Version: 1.0.0-snapshot (xlibs 1.8.3, demonized 20250908)
Changelog: https://github.com/damiansirbu-stalker/JitProfiler/blob/main/doc/changelog
Health: https://damiansirbu-stalker.github.io/JitProfiler/health/
JitProfiler: https://damiansirbu-stalker.github.io/JitProfiler/jitprofiler/
Bugs: https://github.com/damiansirbu-stalker/JitProfiler/issues
Recommended exe: https://github.com/damiansirbu-stalker/fork-xray-monolith/releases/tag/2026.9.12-mt-jitprofiler
Russian / На русском: https://github.com/damiansirbu-stalker/JitProfiler/blob/main/doc/readme_ru.txt

My work:
GitHub: https://github.com/orgs/damiansirbu-stalker/repositories
ModDB: https://www.moddb.com/members/damian-sirbu/addons
Nexus: https://www.nexusmods.com/profile/damiansirbu/mods

My contributions:
X-Ray Monolith: https://github.com/themrdemonized/xray-monolith

[ Hero image: jitprofiler-hero.gif - the profiler runs a capture ]

Early release. Some of the X-Ray crash fixes and optimizations this build relies on are not yet in the latest demonized exe, so run it with the exe from the fork below:
https://github.com/damiansirbu-stalker/fork-xray-monolith/releases/tag/2026.9.12-mt-jitprofiler
It is the latest demonized MT build (2026.9.12) with a few X-Ray crash fixes and optimizations on top, nothing else.

Stop guessing, measure.

JitProfiler is a Lua profiler for Anomaly. It samples the script layer while you play and names the mod behind each cost.
Start a capture and play. Stop it and read the report, every mod and script ranked by cost, each resolved through the MO2 load order to the copy that ran.
CPU ranks Lua time, MEM ranks allocation. Sampling keeps the JIT on and costs almost nothing, so one capture covers the whole modpack with no module to pick.
When the ranking names a mod, instrument its scripts for exact per-call timing.
The sampler and the allocation counter are native code in xray-monolith, this project's own contribution to the engine, so it reads the allocation bytes and the VM state a script-side profiler cannot.

Sampling - CPU and MEM. Leave it running while you play, then read the ranking.

CPU - the ranked breakdown of Lua time, by mod and by script.
It samples the running stack on the engine's own timer at 5 ms with the JIT on, so the overhead is near-zero.
The interval is jittered on an exponential draw, so a mod on a schedule cannot phase-lock to it.
It splits the time by VM state and lists the engine-C entry points, your Lua call sites that drive the engine cost and the map for a native profiler like Optick.

MEM - the ranked breakdown of allocation, the garbage the collector must clear.
It counts exact bytes at the allocator seam with the JIT off, the stutter source a script-side tool cannot measure.
The framerate drops during the capture, so keep it short. The byte counts stay exact regardless.
A live GC-health strip reads the heap toward the next collection, the live estimate, the debt, and the collection rate.

Instrumentation - INSTRUMENT and CALLBACKS. Use these on the mod that sampling named.

INSTRUMENT - own and total wall-clock per function over the scripts you pick, engine C included.
It wraps every function of the set and times each call, with the call count and the average, minimum, and maximum per call.
It records the call graph and totals the cost per owning mod, so you see where a function's time goes.
Each call carries a timer, so the framerate drops while a scan runs. The numbers stay exact.
Build the set from the modlist browser or straight from a CPU row.

CALLBACKS - what each event handler costs and which mod hooks a callback.
It wraps the handlers you pick over the game's make_callback dispatch and ranks each by own ms with its callback and owning mod, engine C included.
Dispatch order and semantics stay untouched, so the reading matches what actually runs.

PUBLISH - share a capture as a web page without exposing other authors' mods.
One click turns a capture into a live page on your own GitHub, the in-game view drawn in the browser and shareable by a link.
Every third-party mod name is masked in the panel and in every export unless your keep-list frees it, so another author's mod never stands beside a cost.
The token stays in the session and travels only to GitHub over HTTPS, with no server in between.
No other Anomaly profiler publishes from inside the game.

The panel opens over the F11 overlay, JitProfiler in the menu bar, and holds the 5 tabs.
Own counts the innermost frame when the sample fired, Total counts every frame on the stack.
Every column sorts, and a fold toggle hides the baseline so only your mods remain.
Console commands mirror every capture for scripted runs, and the Settings hold the interval, depth, and an auto-stop safety.
Every capture also writes a ranked text report, a flamegraph for speedscope.app, and a machine-readable json to appdata/logs, so nothing the panel shows lives only in the panel.

Requirements:
Anomaly 1.5.3
A modded-exes build with the JitProfiler primitives.
jit.profile and jit.allocprof are in the official 2026.9.12 modded exes.
jit.util.gcstat, which drives the GC-health readout, is not yet.
For the full feature set, use the preview exes from the fork below:
https://github.com/damiansirbu-stalker/fork-xray-monolith/releases/tag/2026.9.12-mt-jitprofiler
xlibs (used for logging).
Launch with -dbg (MO2 launch arguments) to use the console commands.

Compatibility:
Coexists with everything. A developer profiler with no gameplay of its own. It samples only while you run a capture and stays dormant otherwise.

How It's Built:

The system has a C core and a Lua layer.

The C core in xray-monolith is a backport and adaptation of LuaJIT's jit.profile timer sampler into the 2.0.4 exe without GC64, so saves stay compatible.
The jit.allocprof allocation counter runs on top, both native in the exe beneath the script layer.
Per-mod attribution runs through the MO2 virtual filesystem, resolving each merged script to the load-order winner that ran, verified on real conflicts.

The Lua layer is original, learned through reverse-engineering X-Ray and custom engine changes.
It favors the engine's own mechanisms and minimal intervention, and adds no per-frame cost while idle.
It carries tracing and monitoring from the ground up, every flow timed off the log level.
The mod avoids writing engine values, holding its own state in parallel, so save corruption is impossible.
Every commit runs the full pipeline locally and in CI: luacheck, a Selene build compiled for STALKER with flags the public build lacks, and a load test that runs every script against engine stubs.
Rule layers then check crash safety, hotpath cost, engine correctness, complexity, architecture contracts, security, and the docs.
It depends on no other mod, not even the author's own. The only shared layers are X-Ray and xlibs.

That pipeline runs on every commit and publishes what it finds. The header links a live health page and a JitProfiler capture of the mod's real CPU and allocation cost.

Credits:
LuaJIT and jit.profile are by Mike Pall.
Built for the themrdemonized modded exes (themrdemonized/xray-monolith).

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "JitProfiler by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  The full license is in the LICENSE file and on GitHub.

Diagnostics and reporting:
Every release goes through careful engineering and testing, but bugs can still slip through.
To report one, reproduce with debug logging on, and the world log where the mod has one.
First rule this mod out: reproduce with it off, then on. The cleanest test is this mod alone on vanilla and xlibs.
Send the traces on the Anomaly Discord, or file a defect on GitHub with the same information.
Attach xray.log, the mod log, the engine build, the modlist, and the load order.
For deep technical details and mechanisms, check the architecture docs on GitHub.

Tags: engine-native, performance, save-safe, profiler, luajit, sampling-profiler, cpu-profiling, memory-profiling, gc, flamegraph, callbacks, instrumentation, mod-attribution, low-overhead, optick
