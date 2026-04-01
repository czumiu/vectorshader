# vectorshader

## Clean-slate architecture proposal (standalone HTML app)

This repository currently contains historical reference snapshots and a debugging SVG corpus.
The recommended next step is to rebuild as **one standalone HTML file** with clear internal boundaries,
using the old versions only as behavioral references.

## What we learned from the reference files

- `v46` is the key visible-fill stability reference (endpoint leak fixes, closed-loop sign behavior).
- `v122` has a useful clip-path insight (single-loop clip winding) but should be treated as a narrow, gated behavior instead of a global rule.
- Later large versions blend UI/debug/render concerns too tightly, which made regressions hard to isolate.
- `debug shapes.svg` should become the canonical regression fixture for fill, clip, gradients, and hole/subpath behavior.

## Product constraints for MVP

MVP flow:
1. Upload SVG.
2. Parse + compile shader.
3. Automatically copy shader text to clipboard when possible.
4. Any setting change re-compiles and re-copies.

Intentional non-goals for MVP:
- No animated SVG support.
- No text (must be converted to paths before import).
- No advanced stroke joins/caps beyond round behavior.
- Blend-layer support can be deferred behind a capability flag.

## Recommended architecture

Keep one `index.html`, but split JS into explicit pipeline modules (by section + namespace), with immutable stage outputs.

### 1) App shell (`AppController`)

Responsibilities:
- Wire UI events (file upload, text paste, setting toggles).
- Trigger compile pipeline with debounce.
- Clipboard side effect (best effort + status).
- Persist last-good compile result and diagnostic state.

Rules:
- No geometry logic.
- No shader string templating.

### 2) Front-end parser (`SvgFrontend`)

Responsibilities:
- Parse SVG DOM.
- Resolve styles/presentation attributes.
- Resolve transforms.
- Normalize primitives into a strict scene IR.

Output type: `SceneIR` (pure data).

### 3) Geometry normalization core (`GeometryCore`) — most important

Responsibilities:
- Convert path commands to canonical segments.
- Build **closed fill contours** explicitly (including synthetic closing edges).
- Build curve-aware boundary representation for distance/AA.
- Build clip-safe contour representation from the same normalized basis.

Output type: `ShapeTopologyIR`.

Invariant examples:
- Every contour used for inside/winding has explicit closure edge.
- Fill and clip path evaluators consume the same normalized contour format.
- Any gated fallback (parity vs winding) records why it was chosen.

### 4) Shader IR builder (`ShaderIRBuilder`)

Responsibilities:
- Convert topology + paint data to backend-neutral shader IR.
- Keep backend-independent primitives (`segmentDistance`, `insideTest`, `paintSample`, `compose`).

Output type: `ShaderIR`.

### 5) Backend emitter (`GodotEmitter`)

Responsibilities:
- Emit Godot canvas_item fragment shader text from `ShaderIR`.
- Avoid any geometry decisions here (emit only).

Output type: `GeneratedShader` string.

Later: add `GLSLEmitter` without touching parser/topology code.

### 6) Diagnostics + regression harness (`Diagnostics`)

Responsibilities:
- Per-stage timing + counts (segments, contours, closures, clip groups).
- Toggleable debug panels that inspect IR snapshots.
- Deterministic golden tests against fixtures (starting with `debug shapes.svg`).

Minimum harness:
- Run compile for fixture set.
- Save generated shader + structured compile report JSON.
- Diff reports between revisions.

## Data contracts (critical for avoiding “house-of-cards” regressions)

Use strict stage contracts:

- `SceneIR` → no backend details.
- `ShapeTopologyIR` → normalized closed contours + edge primitives + clip topology.
- `ShaderIR` → backend-neutral codegen graph.
- `GeneratedShader` → final text only.

Each stage should be pure (input -> output) and unit-testable.
Side effects (UI/clipboard/logging) stay in `AppController`.

## Suggested implementation order (slow + intentional)

1. Skeleton standalone `index.html` with module namespaces and compile button flow.
2. Implement parser + path normalization only; print structured IR.
3. Implement fill inside-testing + closure assertions.
4. Implement edge distance + AA.
5. Implement clipping using same topology core.
6. Implement Godot emission.
7. Add auto-clipboard on success + settings recompile/copy loop.
8. Add fixture runner and baseline outputs for `debug shapes.svg`.

## Sanity checks to run continuously

- **Closure check:** every contour used in winding/parity must end with explicit closing edge.
- **Dual evaluator check (debug mode):** compare parity and winding classifications for suspicious contours and log divergences.
- **Emitter truth check:** hash generated shader blocks and diff by stage to ensure edits change active code paths.
- **Round-trip check:** changing a UI optimization setting must only affect expected shader sections.

## Practical guidance from historical versions

- Borrow the visible-fill robustness ideas from `v46` first.
- Borrow the single-loop clip winding behavior from `v122` only behind strict gating + diagnostics.
- Do not start with grid acceleration or large UI/debug feature sets.
- Prefer smaller core rewrite over patching one giant historical file.
