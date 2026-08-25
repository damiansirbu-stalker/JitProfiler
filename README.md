# JitProfiler: engine-native LuaJIT sampling profiler for STALKER Anomaly

Samples the running Lua stack on the engine's own timer, so the JIT stays on and the overhead stays near zero.
One capture covers a whole modpack with no wrapping and no module selection, and it reports where the Lua time goes, ranks the hot functions and scripts, and writes a SpeedScope flamegraph.

[Releases](https://github.com/damiansirbu-stalker/JitProfiler/releases) | [Bugs, suggestions](https://github.com/damiansirbu-stalker/JitProfiler/issues)

Requires: Anomaly 1.5.3, a demonized modded-exes build carrying the profiler primitives, launched with -dbg. Exact versions in [readme.txt](doc/readme.txt).

## Alife Collection

- [AlifeAmbience](https://github.com/damiansirbu-stalker/AlifeAmbience)
- [AlifeBalance](https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance)
- [AlifeCompanions](https://github.com/damiansirbu-stalker/AlifeCompanions)
- [AlifeDiegetic](https://www.moddb.com/mods/stalker-anomaly/addons/diegetic-audio-control-100)
- [AlifeGuard](https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001)
- [AlifePlus](https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01)
- [AlifeSpooks](https://github.com/damiansirbu-stalker/AlifeSpooks)
- [AlifeTactics](https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics)
- [FurnitureFuel](https://github.com/damiansirbu-stalker/FurnitureFuel)
- [JitProfiler](https://github.com/damiansirbu-stalker/JitProfiler)
- [TestZone](https://github.com/damiansirbu-stalker/TestZone)
- [xlibs](https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)

## Documentation

- [readme.txt](doc/readme.txt) - usage, output, install
- [changelog](doc/changelog) - version history
- [architecture.md](doc/architecture.md) - design, for modders

## License

PolyForm Perimeter License. See [LICENSE](LICENSE).
