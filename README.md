# JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly

Samples the running Lua stack on the engine's own timer, so the JIT stays on and the overhead stays near zero.
One capture covers a whole modpack with no wrapping and no module selection, and it reports where the Lua time goes, ranks the hot functions and scripts, and writes a SpeedScope flamegraph.

[![ModDB](https://img.shields.io/badge/ModDB-c8102e)](https://www.moddb.com/mods/stalker-anomaly/addons/jitprofiler) [![Nexus Mods](https://img.shields.io/badge/Nexus_Mods-da8e35)](https://www.nexusmods.com/stalkeranomaly/mods/145) [![Releases](https://img.shields.io/badge/Releases-24292f?logo=github&logoColor=white)](https://github.com/damiansirbu-stalker/JitProfiler/releases) [![Bugs, suggestions](https://img.shields.io/badge/Bugs%2C_suggestions-24292f?logo=github&logoColor=white)](https://github.com/damiansirbu-stalker/JitProfiler/issues) [![Ko-fi](https://img.shields.io/badge/Ko--fi-ff5e5b?logo=kofi&logoColor=white)](https://ko-fi.com/damiansirbu)

[![ci](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml/badge.svg)](https://github.com/damiansirbu-stalker/JitProfiler/actions/workflows/ci.yml) [![Project Health](https://img.shields.io/badge/project_health-dashboard-00ced1)](https://damiansirbu-stalker.github.io/JitProfiler/)

Requires: Anomaly 1.5.3, a demonized modded-exes build carrying the profiler primitives, launched with -dbg. Exact versions in [readme.txt](doc/readme.txt).

## My work

- Alife mods: [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01) · [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics) · [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance) · [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- Diegetic mods: [DiegeticControl](https://www.moddb.com/mods/stalker-anomaly/addons/diegeticcontrol) · DiegeticAmbience · DiegeticDread
- Tools: [JitProfiler](https://www.moddb.com/mods/stalker-anomaly/addons/jitprofiler)
- Libraries: [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
- Engines: [X-Ray Monolith](https://github.com/themrdemonized/xray-monolith/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged) · [OpenXRay](https://github.com/OpenXRay/xray-16/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged) · [IX-Ray](https://github.com/ixray-team/ixray-1.6-stcop/commits/develop?author=damiansirbu)
- Dependents: [COMPASS](https://github.com/Crimento/COMPASS) · [Nemesis Spawn System](https://github.com/SirEndet/Nemesis-Spawn-System) · [Word of Mouth](https://www.moddb.com/mods/stalker-anomaly/addons/word-of-mouth)
- Integrations: [Warfare (erepb)](https://www.moddb.com/mods/stalker-anomaly/addons/warfare-alife-overhaul-new) · [Homestead](https://www.moddb.com/mods/stalker-anomaly/addons/homestead-stalker-settlement-builder) · [Stealth Overhaul](https://github.com/Alex-leon1594/Stealth_Overhaul_Reworked) · [TALKER](https://www.moddb.com/mods/stalker-anomaly/addons/talker-ai-powered-npc-dialogue-beta) · [Factions of the Zone](https://github.com/devinhorowitz/factionsofthezone) · [Anomaly Radios Extended](https://www.moddb.com/mods/stalker-anomaly/addons/anomaly-radios-extended-full-remaster-152)
- Collaborations: [xAGNA](https://www.moddb.com/mods/stalker-anomaly/addons/xagna)

## Documentation

- [readme.txt](doc/readme.txt) - usage, output, install
- [changelog](doc/changelog) - version history
- [architecture.md](doc/architecture.md) - design, for modders

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
