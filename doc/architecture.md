# JitProfiler Architecture

Engine-native LuaJIT sampling profiler for STALKER Anomaly.
It has an engine layer and a mod layer.

The engine layer is C, compiled into the demonized exe.
`jit.profile` is LuaJIT 2.1's timer stack sampler, backported into 2.0.4 without GC64 so saves stay compatible.
`jit.allocprof` accounts exact per-allocation bytes on the allocator seam.
`jit.util.gcstat` reads the live GC counters (total, threshold, estimate, debt) for the GC-health strip.
These are raw capability delivered in the engine, not in this mod.

The mod layer is Lua. `JitProfiler.script` is the front-end.
It drives the primitives from the console and aggregates the samples. It also resolves each script to its owning mod.
`jitprofiler_report.script` is the presentation layer.
It renders the computed views into ranked text reports, SpeedScope flamegraphs, and the call-graph neighbors, so the front-end holds only capture and resolution.
It logs through xlibs `xlog`.

## CPU: sampling, not instrumentation
`jit.profile` fires a timer on a dedicated thread and records the running stack and the VM state on every fire, with the JIT compiled.
Overhead stays near-zero.
One capture covers the whole modpack with no per-call wrapper and no module selection.
Interval and stack depth are per-capture arguments to `start_cpu`.

The interval is jittered. Each gap is a random draw from an exponential around the mean, so the sample times are never periodic.
A mod running on a fixed schedule then cannot phase-lock to the sampler, so a capture sees its true share.
The allocation profiler already draws its sample distance the same way, so both samplers are free of fixed-interval aliasing.

A sample that lands outside Lua execution is only delivered at the next bytecode dispatch, so its weight would land on the next Lua frame, usually a binder freshly entered from the engine.
`collect_cpu_sample` therefore reads the sample's VM state and prepends a `[C]`, `[GC]`, or `[JIT]` pseudo-leaf to the captured stack when the state was not Lua.
The booking then splits on the dumped depth.
A pseudo-leaf over a single frame is a fresh entry, so the preceding C time is unattributable trunk work and books to the engine owner.
A pseudo-leaf over 2 or more frames means Lua was mid-execution around a C or GC call, so the weight books to the resuming frame, the one that made the call.
The one approximation: a C call that itself enters Lua (a callback fired from inside an engine API) charges the entered handler, the nearest seam the dump can see.
Either way a script is never credited for trunk time that ran before it was entered.

## Allocation: exact bytes, JIT off
`jit.allocprof` counts every allocation at the allocator seam.
It drains the pending bytes to the current stack on each bytecode instruction, with the JIT off for the session.
Byte totals are exact.
Attribution is instruction-granular.

## Instrumentation: own and total over a script set
The sampler names where the frame sits but cannot enter the engine C beneath a Lua call, so a mod that spends its cost inside an engine routine reads as engine time.
Instrumentation closes that gap by wrapping every function of a chosen SET of scripts.
`add_target` puts a script in the set, `add_mod_targets` adds every script a mod owns, and the set holds many scripts at once.
`start_scan` wraps every function of every set script with a timer over xlibs `xprofiler` on the engine `profile_timer`. `stop_scan` restores every original and writes the report.
The timer spans the whole call, so the engine C the function triggers is inside its number, the axis the JIT-on sampler cannot reach.
Each call pushes a pooled stack frame. On return the frame banks total, its whole span, and own, total minus the time charged by nested wrapped children.
Own is exclusive and correct across nesting and cross-mod calls. Total is inclusive.
Each function shows its call count, own ms, total ms, and us per call, ranked by own, with own ms per frame as the budget.
The wrapper runs the original under pcall.
A Lua error still closes the frame, then re-raises to the caller unchanged.
A recursive function books own correctly, and total stays inclusive by definition.
It is opt-in and mutually exclusive with the CPU and allocation captures, because the wrapper tax and the JIT blinding are the per-call cost the sampler exists to avoid.
The natural loop runs the sampler first, then instruments whichever scripts the `[C]` rows name.
It also records the call graph.
Each wrapped call books its span to a parent-to-child edge, so the report shows where a function's total goes and who drove it.
A whole-mod scan totals the cost per owning mod.

## Callbacks: per-handler over the dispatch
Every script callback flows through one chokepoint, `make_callback` in axr_main, which dispatches each registered handler in priority order.
`start_callbacks` reads that file-local `intercepts` table by upvalue and swaps each function handler in place for a timing wrapper, keeping its priority.
`make_callback` is never reimplemented, so dispatch order and semantics stay untouched.
Each wrapper times its handler inclusive of the engine C it triggers, banks own and total on a pooled stack like the scan, and re-raises a handler error unchanged.
`stop_callbacks` swaps every original back and writes the report.
A level change or actor destroy force-restores first, so no wrapper orphans.
`spairs` snapshots the handler keys, so a mid-dispatch restore is safe.
A picker set holds the callbacks to time, filled by Add all or an individual pick, and cleared by Remove all.
An empty set arms nothing, so start_callbacks refuses it, the same as an empty instrument set.
The report ranks every handler by own ms with its callback and owning mod, plus a per-callback rollup, answering which mod hooks a callback and what each handler costs, engine C included.
It is opt-in, mutually exclusive with the other captures, and goes inert on a build where the intercepts upvalue is not reachable.

## HITCHES: per-render-frame time
The CPU, allocation, instrumentation, and callbacks modes rank Lua work; none says which FRAMES were slow. HITCHES records the per-render-frame time, names the hitches, and couples a CPU capture in so one button ties the stutter to a cause.
It reads `device().time_delta` on the `actor_on_update` heartbeat, one value per frame. That field is the engine's raw frame delta, unsmoothed and unclamped, unlike `f_time_delta` which the engine low-passes and caps at 100ms. The actor registers in the realtime scheduler list and so updates every frame, so the heartbeat is per-frame and the delta it reads is that frame's time.
The capture appends the delta and nothing else: one field read and one table append per frame. Every statistic is computed once at stop over the full series: median, mean, min, max, p95, p99, the 1% low, and the 0.1% low above a frame-count floor.
A hitch is a frame whose delta clears both gates, more than k times the whole-capture median (default 2.0) AND more than an absolute floor (default 50ms). Both must hold, so jitter on a smooth run and the smooth frames on a slideshow are both excluded. The report ranks the hitches by frame time and carries each one's wall offset, so a hitch window lines up against a separate CPU capture of the same scene.
The rhythm finder buckets the seconds between consecutive hitches and reports the top intervals, naming the background-cadence case where a hitch on a beat is a process cycling. It needs a minimum hitch count, below which it reports none.
The capture couples the jit.profile CPU sampler in the backend: `start_frame` arms both the frame recorder and the CPU sampler through the shared `start_cpu_sampler`, and `stop_frame` stops both (`stop_cpu_sampler`) and writes both `last_capture.frame` and `last_capture.cpu`. One capture, near-zero cost on both axes. The front-end never orchestrates this; it calls `start_frame`/`stop_frame` and reads the two retained captures.
Each CPU sample also banks into a per-frame accumulator, so "what ran" decomposes to the leaf view OVER THE HITCH FRAMES, not the whole session, the real correlation. The samples taken between two heartbeat ticks belong to the frame whose delta is read at the second tick, so they align with no off-by-one. The attribution is exact, not approximate: an over-floor frame parks its sample map as a hitch candidate, and at stop the same `k x median AND floor` gate the hitch list uses selects which candidates merge into the bucket. The floor is a valid pre-filter rather than a proxy, because a hitch must exceed it by definition, so no true hitch is missed and the selected set equals `metrics.hitch_count`. Parking only over-floor frames keeps the memory to the rare slow frames.
The mode is mutually exclusive with allocation, instrumentation, and callbacks, which distort frame timing; it owns the CPU sampler for its own session. The panel shows the metric strip, a per-window timeline, the rhythm, the top CPU own (what ran), and the worst-frame table, and the corner banner carries a live frame-time and fps readout while it records, toggleable from the tab. The timeline is an ImPlot line of frame time with the hitch gate drawn flat when the exe binds ImPlot (through xlibs `ximplot`), and a single-row heat strip otherwise (a blue ramp by magnitude, amber past 1.5x the median, red over the gate); the series downsamples max-preserving so a spike survives either drawing.
The hitch frames' stacks stay on the capture. The HITCHES tab shows ANOMALY COST over those frames alone, and the fixture check judges them.

## Per-mod attribution
Mods are merged into one `gamedata/scripts`, so the script path never names the mod.
The resolver reads each script's real backing path through MO2/USVFS, over `GetFinalPathNameByHandle` on a LuaJIT FFI handle. A `mods\<X>` path takes X as the owner.
A path the open cannot reach is packed in a db, classified by name against two baked sets regenerated from the unpacked Anomaly tree and the demonized overlay.
The match is case-insensitive because the VFS lowercases names.
A loose path outside `mods\` that neither set names is a file no mod packaged, so it takes the folder holding its `gamedata` (the MO2 `overwrite`, or the install root) as `loose: <folder>`.

| resolves to | owner |
|---|---|
| a `mods\<X>` path | mod: X |
| the demonized overlay set | modded exes |
| the stock Anomaly set | anomaly |
| a loose path in neither set | loose: `<folder>` |
| none of the above | unknown |

A frame's file comes from LuaJIT `short_src`, which front-truncates a path past its 60-byte buffer to `...tail`. The resolver strips that marker, so a long-named script keys to its real basename.
Because the sets ship with the mod, the classification is install-independent and never assumes an unresolved script is vanilla.
A `[C]` frame reads as engine. A `[string]` chunk from loadstring reads as loadstring unless a real Lua frame below it owns the cost.
The resolve runs at report time and caches per script. An absent FFI drops resolution to the name sets alone.

The owner is always the mod that wins the load order.
When several mods override one script, only the MO2 priority winner sits on disk behind the virtual path, and that is the copy the game loaded and ran.
A mod serving a stock-named script is tagged as an override of that file, so a near-identical copy is not misread as the mod's own cost.
Verified on real 3-way conflicts: visual_memory_manager resolved to the active winner over a lower-priority override and a disabled one, and the same held for xr_combat_ignore and zz_item_artefact.

## Metrics and views
Per unit, JitProfiler reports two numbers, Own and Total.
Total is inclusive. A mod or script gets credit whenever it appears anywhere in a sample, counted once per sample. It does not sum to 100%.
Own is exclusive, the innermost frame when the sample fired.
Own is the primary number, per mod and per script, and Total is the second column. A high Total over a low Own marks a dispatcher.
The views are BY MOD, BY SCRIPT, BY LEAF, BY ROOT, and FRAMEWORK vs handlers.
CPU adds the VM-state split and the engine-C entry points, the Lua call sites that drive the engine bucket and the map for a native profiler like Optick.
Each capture also reports the deepest stack it saw and how often a stack reached the depth cap, so an under-counted Total is visible.
The fold toggle drops the baseline rows (anomaly, modded exes, engine, unknown) from the owned views, leaving only mods.
At Stop, `compute_anomaly_cost()` takes each CPU, MEM or hitch-frame stack whose Self frame is not mod-owned and reads its caller chain. The chain is every mod on the stack, nearest the leaf first.
The Self frame comes from `get_stack_anchor()`, the same rule as BY MOD Own. Here a mod is an MO2 mod or a loose install folder. Baseline is what remains.
COST is the one table every tab shows (`compute_cost_rows()`). Each sample is booked to exactly one row. The Cost column therefore adds up to 100%.
A sample whose Self frame is a mod's own code is Own on that function. A sample in vanilla or engine code is Caused on the first mod frame toward the root that was called to do a job.
"Called to do a job" means the frame's caller is a vanilla dispatcher, the engine itself, or mod code.
The dispatchers are `make_callback`, `SendScriptCallback`, `ProcessEventQueue` and the `xr_logic` condlist and event functions, matched by name and by live definition site.
A mod frame whose caller is plain vanilla code is a patch, a wrapper or a replacement. It keeps Own and is passed over for Caused. The vanilla row's hover names it under "via".
Functions under 80% of all stacks hook the callback system. They are passed over the same way and tagged.
With no mod entry on the stack, the vanilla or engine function takes the sample as Own. Anomaly's own heavy code shows as itself at full weight and never on a mod.
The rows can be summed by Function, Script or Mod. A row's hover lists what it caused and what it passed through. The panel, the .txt, the json and the saved page carry the same rows.
This rule errs in one direction only. A dispatcher missing from the list, or a patch that calls heavier vanilla than the original did, makes a mod look cheaper than it is.

## Anonymize
A shared capture would print every third-party mod name beside its cost, which no author can be asked to share.
The Anonymize toggle masks them at the display.
`compute_masked_name` replaces every letter and digit with #, keeping spaces, separators, and the extension, so the length and word shape still read.
The masking is uniform across the panel, the flamegraph, and the json.
The masking is obfuscation, not encryption. The word lengths and separators survive, so a reader can still match a known mod list against the shapes.
It hides names from a casual look at a shared capture. A deliberate de-anonymization still succeeds.
It is print-only. The real name still drives owner resolution, the sort, the colour, and the instrumentation set.
Only the rendered string changes, so nothing downstream reads a masked key.
Only mod-owned names obfuscate.
Baseline owners, the `[C]` `[GC]` `[JIT]` markers, separators, digits, the extension, and a line suffix stay literal, so the report stays readable and the code locations stay honest.
The keep-set exempts the names an author keeps readable.
Each entry is a Lua pattern of at least 3 literal characters, matched case-insensitively from the start of the owner name or the script file, so a broad entry cannot exempt another author's name.
It is the one persisted piece of panel state, a newline file at the writable appdata root edited from the footer, empty by default.
The toggle rewrites the live panel and every export together, so a screenshot, a text report, and the json are safe at the source.

## Save page
A capture becomes a self-contained web page you save to disk, from the footer's Save page control.
JitProfiler writes the retained captures as one JSON and injects it into a shipped viewer page.
It writes the result to the appdata logs folder as jitprofiler_page.html.
The viewer is a static page that draws the in-game panel from the JSON in the browser, so the saved page reads exactly like the tool.
JitProfiler forces the mask on the saved copy regardless of the live toggle.
Every mod name is masked except the ones the keep-list matches, so keeping only your own names leaves every other author's name masked.
A status line under the Save button lists the names the keep-set leaves readable, before anything is written.
JitProfiler makes no network request. Sharing the page is a manual step you own: commit the saved file to your repo as pages/jitprofiler.html and push.

## Output
`appdata/logs/jitprofiler_{cpu,mem}[_snapN]_<timestamp>.txt` is the ranked text.
The instrumentation and callbacks modes write `jitprofiler_inst_<timestamp>.txt` and `jitprofiler_callbacks_<timestamp>.txt`.
The frame mode writes `jitprofiler_frame_<timestamp>.txt`: the metric block, the rhythm, and the ranked hitch list.
`jitprofiler_{cpu,mem}[_snapN]_<timestamp>.folded` is the SpeedScope collapsed-stacks flamegraph.
`jitprofiler_<kind>_<timestamp>.json` is the machine-readable capture, the ranked views plus meta.
A minimal in-module writer encodes it, because Anomaly ships no JSON library. It honours the Anonymize toggle at write time.
The json carries the same ranked views the panel shows, so any HTML site can render it into a page that matches the in-game view.
Every text report is self-contained.
The sampling reports close with a call graph (each top leaf with its callers toward root and callees toward leaf).
The instrumentation and callbacks reports carry per-call avg, min, and max, so nothing the panel shows lives only in the panel.
The timestamp is the capture's wall-clock time, so successive runs never overwrite.
Both are ASCII only.

## In-game panel
`jitprofiler_ui.script` registers the panel and a menu entry through the base ImGui Groups API and draws in the Main group, so it appears while the F11 ImGui overlay is open.
The panel opens at 90% of the live screen, centered, sized on appear from the `vid_mode` cvar so it never assumes a resolution, then resizes freely.
CPU and MEM sampling render in `jitprofiler_ui.script`. `jitprofiler_ui_trace.script` holds the INSTRUMENT and CALLBACKS tabs, and `jitprofiler_ui_config.script` the footer, with the Settings, the Anonymize keep-set, and the Save page control.
Both reach one set of render helpers and the single panel filter through `jitprofiler_ui.SHARED`, so the split keeps one copy of the drawing code and the filter state.
The top tabs are CPU, MEM, INSTRUMENT, CALLBACKS, and HITCHES. A running capture locks the others, so no two run at once.
CPU and MEM show the retained last capture through `get_last_capture`.
A Group by selector sums the COST rows per function, script or mod. Each column header sorts and explains itself on hover, and a filter box narrows the rows. A click on a Mod row filters to that mod and drops to Function.
Each results table fills the height left below it and scrolls within that, sized by `get_fill_size` off `GetContentRegionAvail`, so the analysis grows with the window.
The pickers above it, the SELECTED set and the BROWSE modlist and the callbacks list, fold or render inline, so the panel's own scrollbar carries them without a nested box.
The by-script rows carry a per-row button that adds or removes the script from the instrumentation set.
Under INSTRUMENTATION a start and stop control arms the whole set, and an overhead line shows the wrapped-function count.
Two sub-views split the screen.
SELECTED folds the working set under a header, each target removable.
BROWSE is a modlist grouped by mod with a search box. A + adds one script.
Inside a mod's fold, a +Add all / -Remove all row instruments or clears that whole mod through add_mod_targets and remove_mod_targets.
A Clear button by the set count empties the set.
The search matches the mod name or the script name. A matching mod shows all its scripts, and hits render with their group open, collapse ignored.
The results table ranks each function by own with an own-share bar, plus own ms, total ms, and calls, each column sortable on click.
Avg, min, and max per call are sortable columns, carried through the text report, the json, and the saved viewer page.
A frame-budget bar reads own ms per frame against a 60fps frame.
Number columns right-align through CalcTextSize, so magnitude scans down the column.
Each row's owner is tinted by a stable per-mod colour, hovering shows the full frame, and a selected frame ranks its callers and callees through `compute_neighbors`.
An Advanced fold, closed by default, holds the fold-baseline and parallel-GC switches, the VM-state strip (one bar per state, each explained on hover), and every engine-C entry point in a scroll box.
The CALLBACKS tab arms `start_callbacks` and ranks each timed handler over the make_callback dispatch by own ms, with its callback and owning mod.
A BROWSE subtab lists every registered callback with its handler count and a picker toggle.
A dim line under a running MEM or wrapping capture notes the FPS drop and that accuracy holds.
The MEM view carries a live GC-health strip from `jit.util.gcstat`, a bar for the heap toward the next collection plus the live estimate and debt, hidden when the exe lacks the getter.
On the multi-thread exe a Parallel GC toggle flips the `lua_parallel_gc` cvar, so a CPU capture reads a clean G share.
The panel renders in white JetBrains Mono when the font is present, so digits line up as a column, over a blue accent scheme. Each cost bar carries a single-hue blue heat shade by share.
A fold-baseline toggle drops the non-mod rows, leaving only mods.
A corner banner in the Unique group renders every frame and shows only while a capture runs.
The chunk executes twice, so registration and the retained capture anchor to `_G` singletons.
Measured: unanchored, the `actor_on_update` heartbeat and the console entry points hold separate state tables and `scan.frames` reads 0. Anchored, they share one and frames track ticks.

## Settings and MCM
Capture settings live in the panel, not in MCM.
`JitProfiler.script` keeps them as session state anchored to `_G`, edited from the panel through `get_setting` and `set_setting`.
They cover interval, depth, the auto-snapshot threshold, auto-stop, fold, and the Anonymize toggle.
A console argument to `start_cpu` or `start_alloc` overrides one capture. Only the Anonymize keep-set persists (see Anonymize). The rest is session state.
An auto-stop duration above 0 ends a running capture from the `update_capture` heartbeat and writes its report, so a forgotten allocation capture never holds the JIT off past the limit.

`jf_mcm.script` is an about page through `xmcm.create_config`.
It carries the description, the usage and SpeedScope notes, the version and compatibility footer from `_jitprofiler_init.get_platform_functor`, and one persistent toggle, `show_imgui`.
The panel, menu, and banner read `show_imgui` through a cached flag and draw nothing while it is off, so JitProfiler leaves the ImGui menu bar.

## Limitations
- Inclusive counts and the flamegraph are bounded by the captured stack depth. A frame beyond the depth is not counted, so raise depth to trace deeper.
- Engine C time is measured as one bucket, because the sampler cannot see which C routine ran.
  The `[C]` pseudo-leaf shows the Lua stack it re-entered from, and nothing more.
  Cost a mod adds through data (populations, per-NPC config) surfaces as engine time, never under the mod's name.
- The CPU JIT-compiled share is a floor, because phase-1 sampling is interpreter-anchored and under-counts JIT traces.
- Allocation attribution is instruction-granular, so bytes from a C or GC path are credited to the next Lua frame.
- Per-mod attribution needs an MO2/USVFS install. Elsewhere it degrades to script level.
- CPU and allocation are mutually exclusive. Each refuses to start while the other runs.
- Instrumentation assumes strict call nesting. A wrapped function that yields a coroutine mid-call desyncs the timer stack until stop_scan restores it.
- Instrumentation Own is exact under recursion, but Total inflates.
  A directly recursive wrapped function re-adds each level's span, so its inclusive number double-counts the nested self-calls. Read Own for a recursive function.
- Instrumentation wraps a script's module-table functions.
  A function reached through a saved reference (a registered callback, a stored upvalue, a local alias) still calls the original and is not timed.
  A script written mostly as local functions dispatched through callbacks or time-events exposes little on its module table, so a scan of it times almost nothing; a scan that records no called row says so in the panel and points to CPU sampling or the callbacks mode.
  The callbacks mode covers the dispatch case.

- A sample taken in engine C over a single Lua frame books to engine, because a single frame is read as a fresh entry from the engine.
  The same holds for a coroutine body, which is also a single frame. Its C time books to engine until the sampler learns the thread identity from the engine.
- FRAME measures gameplay frames only: `actor_on_update` does not fire in a menu or a full load screen, so a paused or loading stretch is absent from the series, which is correct for a stutter hunt. A pause reads as near-zero deltas, not a hitch, because the engine freezes the timer.

## Requires
A demonized build exposing `jit.profile` and `jit.allocprof` (and `jit.util.gcstat` for the GC-health strip), and xlibs for `xlog`.
On a stock exe the mod detects the missing primitives and stubs the commands.
