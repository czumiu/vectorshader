# vectorshader

vectorshader is a standalone HTML app that compiles practical SVG artwork into a Godot `canvas_item` fragment shader.

It is not a full SVG renderer. It is a bounded compiler aimed at browser-faithful visible output for common vector art, with honest fallbacks where exact support is not worth the cost or risk.

## What it is for

Use vectorshader when you want to:

- take static SVG artwork,
- preserve the visible 2D result as closely as practical,
- emit a single Godot shader,
- and avoid hand-rebuilding the art as shader code.

## Product stance

vectorshader prefers:

- browser-visible correctness over editor-specific behavior,
- strong primitive recovery where confidence is high,
- compile-time work over runtime work,
- honest approximation over fake exactness,
- and bounded scope over full SVG compliance.

Inkscape is useful for authoring and reduction, but when browsers disagree with Inkscape, browser output is the authority.

## Current stable behavior

The current stable branch has the following practical behavior:

- aspect ratio output modes are implemented,
- defs reachability cull is implemented,
- clip-path handling is broadly hardened,
- `clipPathUnits="objectBoundingBox"` works,
- masks work at a practical luminance-first baseline,
- invert-alpha mask edge cases have a narrow duct-tape fix,
- `userSpaceOnUse` linear gradients are evaluated in inverse space,
- degenerate `objectBoundingBox` gradients suppress paint,
- single-segment linecap handling is corrected,
- donut / hole fallback behavior is fixed,
- invalid clip permissiveness has been tightened,
- mask batching is conservative to avoid cross-group interference.

## Current supported input, in practice

Core visible geometry:

- `<rect>`
- `<circle>`
- `<ellipse>`
- `<line>`
- `<polyline>`
- `<polygon>`
- `<path>`
- `<g>`

Practical structural/resource support:

- `<defs>` when referenced by visible output
- `<use>`
- `<symbol>` through `<use>`
- linear gradients
- radial gradients
- clip paths
- masks

## Current intentional non-goals

Not part of the current product guarantee:

- patterns
- text rendering
- filters as a general supported feature
- markers
- full CSS cascade
- blend modes beyond normal source-over
- full SVG compliance
- perfect arbitrary boolean/path fill exactness
- universally exact mixed-path stroke semantics in every case

If text must survive visually, convert it to paths first.

## Important project-specific rules

### Browser is ground truth

When browser rendering and Inkscape rendering disagree, vectorshader follows browser behavior.

### Visible correctness beats mathematical purity

A branch does not need to be a perfect signed distance field everywhere. If the visible boundary and compositing result are right, that is acceptable.

### Recovery should fail closed

If a generic path can safely be promoted into a stronger primitive, do it. If that promotion is ambiguous, keep it generic.

### Approximation is allowed

General path fill and mixed-path stroke handling may be approximate as long as the visible result stays convincing and stable.

### Scope is bounded

The project should not drift into “full SVG renderer” ambitions. Unsupported features should stay unsupported until they are stable enough to promote.

## Architecture, in one file

Even though the app lives in one standalone HTML file, it still has conceptual layers:

1. SVG frontend parsing
2. defs/resource registry
3. geometry normalization and topology recovery
4. render preparation
5. Godot shader emission
6. UI/debug shell

Keeping those boundaries clear is important for future edits.

## Workflow

Typical workflow:

1. paste or load SVG into the app
2. inspect browser preview
3. compile shader
4. compare Godot output against browser output
5. reduce failures into small diagnostics
6. patch one subsystem at a time

## Testing policy

Handcrafted diagnostics matter more than giant random files when debugging.

Keep a permanent regression suite. Current must-keep tests include:

- the six-case pack,
- the corrected ribbon gradient diagnostic,
- invert-alpha mask edge cases,
- degenerate `objectBoundingBox` gradient cases,
- masked sibling regressions,
- nested `use + clip + mask` cases.

## Editing rules

Before changing code, ask:

- Is this a real browser-faithfulness bug?
- Is this already allowed by an approximation rule?
- Is the feature in scope?
- Will this destabilize clips, masks, gradients, or fallback behavior?
- Is this a semantic fix, or an optimization disguised as one?

## Versioning rule

Use `vX` or `vX.Y`.

- a fully landed new feature increments the major version,
- a patch or fix increments the minor version,
- do not switch to a three-part semver scheme unless the project owner explicitly changes that rule.

## Current recommended next steps

- keep one clean stable merged build,
- keep one separate debug-comments build,
- keep the regression suite beside the stable build,
- update the spec whenever a real subsystem is promoted or deliberately constrained.
