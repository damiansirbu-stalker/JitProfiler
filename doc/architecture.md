# JitProfiler Architecture

Engine-native LuaJIT sampling profiler for STALKER Anomaly. Two decoupled layers.

1. Engine primitives (C, compiled into the exe): `jit.profile`, LuaJIT 2.1's timer-driven stack sampler backported into demonized's LuaJIT 2.0.4 without GC64 (so saves stay compatible, dodging the GC64 savefile break that has the full 2.1 upgrade on hold); and `jit.allocprof`, per-function allocation attribution on the allocator seam. These are the raw capability, delivered in the demonized engine, not in this mod.

2. This mod (Lua): `JitProfiler.script`, the front-end. It drives the primitives from the console, aggregates the samples, and writes the text reports and SpeedScope flamegraphs. It contains no engine code and depends on no other mod.

## Sampling, not instrumentation
`jit.profile` fires a timer on a dedicated thread and records the running stack plus the VM state at each tick, with the JIT still compiled. Overhead is near-zero, and the whole modpack is covered in one capture with no per-call wrapper and no module selection.

## Views (post-processed from the captured stacks)
- VM state: `N` JIT-compiled, `I` interpreter, `C` engine call, `G` garbage collector, `J` JIT compiler. The share of each shows where the Lua time actually goes; a high `G` is GC pressure.
- By leaf: the hot function (`name@script:line`).
- By script: summed per file.
- By root: the outermost frame, i.e. the callback or loop driving the cost.
- Framework vs handlers: callback-dispatch machinery (`make_callback`, `SendScriptCallback`, `spairs`) split from real work.
The allocation report carries the same stack views weighted by bytes.

## Output
`appdata/logs/JitProfiler_{cpu,alloc}_report.txt` (ranked text) and `_{cpu,alloc}.folded` (SpeedScope). ASCII only.

## Requires
A demonized build exposing `jit.profile` + `jit.allocprof`. On a stock exe the mod detects their absence and no-ops with a message.
