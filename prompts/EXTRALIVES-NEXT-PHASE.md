# Extra Lives SWF: Data-Flow, Frames.txt, and Multiplayer Feasibility Investigation

## Mission

Continue the existing Extra Lives 1.170.64 reverse-engineering project from the exact current workspace state. The initial SWF analysis and its documentation already exist. Do not restart the investigation or simply rewrite existing findings.

The goal is to understand the actual relationships between the game's data files and executable code, especially `Frames.txt`, then establish an evidence-based technical plan for possible online multiplayer.

Work incrementally. Preserve progress in small, useful documentation updates rather than attempting an enormous one-shot analysis. Avoid unnecessarily expensive repeated scans, massive terminal output, or dumping entire decompiled classes into the conversation. Use targeted searches and save findings to files.

## 1. Start safely and inspect the workspace

Before changing anything:
- Inspect repository root, branch, Git status, recent commits, current documentation, available decompiler output, tools, and temporary analysis artifacts.
- Locate the original APK, `Game.swf`, `Frames.txt`, `Characters.txt`, `Game-app.xml`, sprite resources, and related data files. Verify the actual paths instead of assuming them.
- Reuse existing analysis and tool outputs wherever valid.
- Record the initial Git status and do not overwrite or delete original assets.
- Keep generated files, caches, temporary extraction output, and experiments separate from original research assets.
- Do not print huge file contents or huge search results to the terminal. Use targeted queries and write detailed results directly to Markdown files.

If the previous session was interrupted, inspect its partial output and Git state first. Recover useful work rather than starting over.

## 2. Investigate Frames.txt completely

This is a mandatory investigation. Locate every relevant copy of `Frames.txt`, identify the canonical file, compare duplicates if necessary, and read the entire canonical file through a suitable script or parser if it is large.

Determine its actual format and purpose from evidence, not from its filename. Inspect:
- Encoding, separators, record structure, fields, value ranges, identifiers, grouping, repeated entries, sentinel values, and unusual records.
- Whether records represent frames, animation mappings, sprite references, timing, character definitions, or something else.
- Which identifiers and fields are meaningful, and which remain unknown.
- Whether values refer to other files, embedded resources, ActionScript symbols, or lookup tables.

Then trace how this data reaches the game:
1. Search the repository, extracted files, decompiled source, scripts, and configuration for `Frames.txt`, filename variants, discovered identifiers, resource paths, and related keys.
2. Identify the loader or embedding mechanism, parser, internal data structure, and all confirmed consumers.
3. Follow callers and callees to determine how values are read, transformed, cached, and used.
4. Identify whether consumers run during initialization or repeatedly during gameplay.
5. Determine whether the data affects gameplay logic, animation, sprite selection, collision, movement, combat, character configuration, or presentation only.
6. If no direct filename reference exists, investigate whether the data is embedded, renamed, generated, or loaded indirectly. If no relationship can be proven, document the searches and limits rather than inventing one.

Create a concise but technically useful field-by-field reference with representative records, code references, dependency flow, unusual cases, and unresolved questions. Do not paste the entire file into the report.

## 3. Investigate Characters.txt and related data

Read and analyze `Characters.txt` and any related configuration/data files using the same evidence-driven method. Identify their schema, parser, consumers, runtime representation, and relationship to `Frames.txt`, sprite assets, animation, character attributes, AI, and gameplay state.

Build a cross-reference between these files. Determine whether identifiers are direct, indirect, positional, or dynamically resolved. Investigate missing, duplicated, invalid, or fallback identifiers where the code provides evidence.

Follow other files into the investigation only when they participate in the same dependency chain.

## 4. Trace the actual ActionScript execution paths

Use existing SWF analysis and decompilation artifacts. Inspect the real code and data references, not only class names or summaries.

Trace:
- `Game.fla.MainTimeline`, entry point, initialization, and scene transitions.
- Resource/data loading and lookup.
- Entity creation, initialization, identity, and mutable state.
- Frame events, timers, update loops, and simulation order.
- Input handling and action dispatch.
- Movement, collision, combat, health, AI, interactions, spawning, and death.
- Animation-state selection, sprite resolution, frame progression, and the meaning of the `A/S/D/F` fields.
- Randomness, time-dependent behavior, frame-rate dependencies, global state, save/load behavior, and state copying.
- The existing local multiplayer implementation and differences from single-player.

Follow actual callers, callees, reads, writes, and state mutations. Identify the important methods and locations that connect data files to runtime behavior. If exact source line numbers are unavailable, provide verified class/method names and the relevant decompiler artifact or search evidence.

## 5. Determine determinism and simulation boundaries

Assess whether the simulation is deterministic, conditionally deterministic, or dependent on local execution details. Investigate:
- Frame-rate dependence and variable update intervals.
- Floating-point calculations and update order.
- RNG usage and whether random state can be captured or synchronized.
- Timers, animation events, wall-clock dependencies, and frame callbacks.
- Entity iteration and collision-resolution order.
- Input timing, simultaneous actions, AI decisions, delayed effects, and state transitions.
- Dependencies between gameplay state, rendering, animation, and static data loading.

For each significant risk, cite concrete code evidence and explain the multiplayer consequence. Do not infer determinism from a single-threaded loop. Do not label a system nondeterministic without evidence or an explicitly marked unresolved hypothesis.

Where static analysis cannot answer a question, specify a reproducible runtime experiment: initial conditions, controlled inputs, state variables to record, comparison method, and what result would confirm or reject the hypothesis.

## 6. Build a candidate logical game-state model

Separate:
- Authoritative gameplay state.
- Derived/reconstructible state.
- Animation and presentation state.
- Client-local state.
- Static definitions loaded from data files.
- Persistent/save state.

Trace proposed fields to actual code. Explain which fields change, where they change, who consumes them, whether they need synchronization, and what can happen if instances disagree. Explicitly determine whether `Frames.txt` contributes to gameplay state, presentation state, or both, with evidence.

Do not invent unsupported fields or claim the proposed schema is already implemented.

## 7. Evaluate multiplayer architecture against the actual code

Compare:
A. Deterministic lockstep.
B. Server-authoritative simulation.
C. Authoritative state replication with local interpolation/presentation.
D. A hybrid architecture, if justified.

For each approach, discuss evidence-based advantages, concrete blockers, code/runtime dependencies, required modifications, latency and divergence risks, security implications, and likely failure modes. Specifically assess whether simulation logic can be separated from Flash/AIR rendering and whether a headless server-side simulation is realistic.

Use the original local multiplayer mode as evidence, not as proof that online multiplayer will be easy. Trace how local players, controls, entities, and shared state are represented, and identify assumptions that break across separate devices/processes.

Recommend an architecture only after comparing it with the observed implementation.

## 8. Design a minimal prototype, but do not implement networking yet

Prepare a realistic plan for two possible prototype scopes:
- Prototype A: controlled movement and position/state synchronization.
- Prototype B: collision or combat synchronization.

For each, specify scope, exclusions, runtime/server requirements, authoritative state, input/state protocol, update frequency or tick strategy, timestamping, interpolation, handling of delayed/missing updates, logging, divergence detection, and measurable acceptance criteria.

Explain whether each prototype should use the original runtime, a modified build, or a separate simulation, and why. Identify the riskiest assumption that each prototype tests. Do not start implementing networking until the investigation and plan are documented.

## 9. Documentation deliverables

Create or update these English Markdown files under `docs/extralives-swf-analysis/`:
- `FRAMES-TXT-ANALYSIS.md`
- `CHARACTERS-TXT-ANALYSIS.md`
- `DATA-FLOW-AND-DEPENDENCIES.md`
- `DETERMINISM-AND-GAME-LOOP.md`
- `LOGICAL-GAME-STATE-MODEL.md`
- `ONLINE-MULTIPLAYER-ARCHITECTURE.md`
- `ONLINE-MULTIPLAYER-PROTOTYPE.md`

Update existing `README.md`, `GAME-ARCHITECTURE.md`, `CODEBASE-AND-CLASS-INDEX.md`, `ASSETS-AND-DATA-FORMATS.md`, `NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md`, and `EVIDENCE-AND-OPEN-QUESTIONS.md` only where new evidence warrants it.

Documentation must:
- Distinguish verified facts, reasoned inferences, hypotheses, and runtime-only questions.
- Cite exact file paths, classes, methods, fields, and data records where available.
- Explain the full chain from data file to parser/loader to runtime consumer to behavior.
- Avoid unsupported certainty and generic claims without technical explanation.
- Be useful to another developer or coding agent continuing the work.
- Clearly list what was and was not investigated.

## 10. Work in bounded stages to avoid losing progress

Do not attempt to produce all deliverables in one enormous uninterrupted operation.

Use this sequence:
1. Inspect the workspace and write a short private/progress note identifying artifacts and current state.
2. Analyze `Frames.txt` and save its findings.
3. Trace `Characters.txt` and cross-file dependencies; save findings.
4. Trace execution paths and determinism.
5. Build the logical state model and architecture comparison.
6. Write the prototype plan.
7. Cross-check documentation, then update summaries and open questions.

After each stage, save useful findings to the intended Markdown files so a crash or interruption does not lose all progress. Keep tool output concise. Avoid giant terminal dumps, huge context responses, unnecessary parallel heavy jobs, or repeating completed scans. If a task is too large for the current session, finish and validate the current stage, document the exact next step, and stop cleanly rather than risking the whole session.

## 11. Verification and GitHub

Before committing:
- Verify claims about `Frames.txt` and `Characters.txt` against the files and confirmed consumers.
- Check class/method names, paths, identifiers, and cross-document consistency.
- Ensure original APK, SWF, and extracted assets are unchanged.
- Exclude binaries, caches, credentials, unrelated files, and temporary outputs.
- Review the full diff and validate Markdown links/structure.
- Commit only intended research documentation changes with a descriptive commit message.
- Push to `origin main` and verify the resulting commit and changed files on GitHub.

Never claim that a push or runtime test succeeded without confirmation. If tooling, decompilation, runtime access, or authentication blocks a step, record the exact blocker and next actionable step.

## Final response

Report in Persian:
1. What was actually inspected, especially `Frames.txt` and `Characters.txt`.
2. What those files contain and their verified relationships to code, sprites, animation, and gameplay.
3. The most important traced code paths and dependencies.
4. Evidence-based conclusions about determinism and multiplayer feasibility.
5. Recommended architecture and the smallest useful prototype.
6. Remaining uncertainties and exact runtime experiments required.
7. Commit SHA and GitHub links only after verifying the push.

Prioritize verified technical evidence and actionable conclusions over document volume. The objective is to understand how the original game works and what would actually be required for online multiplayer, not merely to create more Markdown.