# JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly

Samples the running Lua stack on the engine's own timer, so the JIT stays on and the overhead stays near zero.
One capture covers a whole modpack with no wrapping and no module selection, and it reports where the Lua time goes, ranks the hot functions and scripts, and writes a SpeedScope flamegraph.

[Releases](https://github.com/damiansirbu-stalker/JitProfiler/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/JitProfiler/issues)

[![ci](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml/badge.svg)](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml) [![Project Health](https://img.shields.io/badge/project_health-dashboard-00ced1)](https://damiansirbu-stalker.github.io/JitProfiler/)

Requires: Anomaly 1.5.3, a demonized modded-exes build carrying the profiler primitives, launched with -dbg. Exact versions in [readme.txt](doc/readme.txt).

## My work

- Alife mods: [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01) · [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics) · [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance) · [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- Diegetic mods: [DiegeticControl](https://www.moddb.com/mods/stalker-anomaly/addons/diegeticcontrol) · DiegeticAmbience · DiegeticDread
- Tools: [JitProfiler](https://www.moddb.com/mods/stalker-anomaly/addons/jitprofiler)
- Libraries: [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
- Engines: [X-Ray Monolith](https://github.com/themrdemonized/xray-monolith/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged) · [OpenXRay](https://github.com/OpenXRay/xray-16/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged)
- Integrations: [Word of Mouth](https://github.com/joshcoppola/word_of_mouth) · [Warfare (erepb)](https://www.moddb.com/mods/stalker-anomaly/addons/warfare-alife-overhaul-new) · [Stealth Overhaul](https://github.com/Alex-leon1594/Stealth_Overhaul_Reworked) · [COMPASS](https://github.com/Crimento/COMPASS)
- Collaborations: [xAGNA](https://www.moddb.com/mods/stalker-anomaly/addons/xagna)

## Documentation

- [readme.txt](doc/readme.txt) - usage, output, install
- [changelog](doc/changelog) - version history
- [architecture.md](doc/architecture.md) - design, for modders

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
