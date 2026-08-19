# JitProfiler: engine-native LuaJIT profiler for STALKER Anomaly

JitProfiler samples the running Lua stack on the engine's own timer — the JIT stays on, overhead is near-zero, and one capture covers the whole modpack with no wrapping or module selection. It reports where the Lua time goes (JIT-compiled / interpreter / C / GC), ranks the hot functions, scripts, and callback roots, and writes a SpeedScope flamegraph. It consumes two primitives backported into the demonized engine: jit.profile (CPU) and jit.allocprof (allocation).

[Releases](https://github.com/damiansirbu-stalker/JitProfiler/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/JitProfiler/issues)

Requires: Anomaly 1.5.3, a demonized modded-exes build with the profiler primitives, launched with -dbg. Details in [readme.txt](doc/readme.txt).

## Documentation
- [readme.txt](doc/readme.txt) — usage, output, install
- [changelog](doc/changelog) — version history
- [architecture.md](doc/architecture.md) — design, for modders

## License
PolyForm Perimeter License. See [LICENSE](LICENSE).
