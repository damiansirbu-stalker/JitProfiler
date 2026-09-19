# JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly

Samples the running Lua stack on the engine's own timer, so the JIT stays on and the overhead stays near zero.
One capture covers a whole modpack with no wrapping and no module selection, and it reports where the Lua time goes, ranks the hot functions and scripts, and writes a SpeedScope flamegraph.

[Releases](https://github.com/damiansirbu-stalker/JitProfiler/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/JitProfiler/issues)

[![ci](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml/badge.svg)](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml) [![Project Health](https://img.shields.io/badge/project_health-dashboard-00ced1)](https://damiansirbu-stalker.github.io/JitProfiler/)

Requires: Anomaly 1.5.3, a demonized modded-exes build carrying the profiler primitives, launched with -dbg. Exact versions in [readme.txt](doc/readme.txt).

## My work

- [GitHub](https://github.com/orgs/damiansirbu-stalker/repositories)
- [ModDB](https://www.moddb.com/members/damian-sirbu/addons)
- [Nexus](https://www.nexusmods.com/profile/damiansirbu/mods)

My contributions to the engine: [X-Ray Monolith](https://github.com/themrdemonized/xray-monolith)

## Documentation

- [readme.txt](doc/readme.txt) - usage, output, install
- [changelog](doc/changelog) - version history
- [architecture.md](doc/architecture.md) - design, for modders

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
