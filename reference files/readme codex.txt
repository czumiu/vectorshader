Project purpose

This project is a portable HTML/CSS/JS tool that converts SVG artwork into a Godot canvas_item shader that reproduces the SVG as closely as possible inside Godot.

The real goal is not “support all of SVG perfectly.” The goal is:

take practical SVG artwork
preprocess as much as possible at conversion time
emit a Godot shader that draws the result at runtime
keep the output visually faithful, especially for:
fills
clipping
gradients
strokes
anti-aliasing
avoid brute-force triangle tessellation as the main solution, because smooth curve behavior is important

note: because of sdf math, all strokes will have round caps for ends and joint styles. 

This tool is used on real SVGs, not toy examples. The hard cases involve:

many subpaths
holes
clipping
arcs
quadratics
open-ish paths that are treated as filled by the solver
endpoint leaks
horizontal line artifacts
complex layered clip interactions

The project has already gone through many versions. The code is large and messy. The right approach is not to blindly patch random branches. The right approach is to understand the fill/clip pipeline and make the logic coherent.

Core philosophy of the renderer

The architecture that historically worked best is this:

1. Fill topology / inside-testing should be robust

The solver often needs to close paths for fill processing even if the source SVG path is open or ambiguous. This is because inside/outside logic requires closed regions.

2. Boundary appearance should remain curve-aware

Even when fill topology is flattened or simplified for robustness, edge distance should still preserve smooth curves where possible.

That means the practical compromise is:

inside/sign/topology: robust closed contour logic
edge AA / boundary distance: curve-aware distance where possible

This distinction matters a lot.

A repeated failure mode in this project was mixing:

one representation for topology
another representation for edge distance
and another representation for clip masks

That created regressions and circular debugging.

What the app currently does conceptually

The app reads SVG content, parses shapes, and emits a Godot shader.

The pipeline conceptually contains these parts:

A. Parse SVG
read SVG elements
resolve transforms
resolve style attributes
interpret fills, strokes, gradients, clip paths
B. Convert primitives into internal shapes

This includes at least:

rect
circle
ellipse
path

Paths are the hard part.

C. Build path representations

A path may produce:

original path segments
fill segments
sign segments
flattened contours
fill contours
stroke segments
topology debug structures

These representations have drifted apart over time. Codex needs to reduce that incoherence, not increase it.

D. Emit shader code

The shader computes:

fill alpha
stroke alpha
clip alpha
gradient color
final composition
E. Preview/debug in the HTML app

The preview/debug layer has historically been useful, but it has also started crashing independently of the render logic. Codex should treat debug code as secondary, not as the truth.

What matters most

The important subsystem is the path fill/clip core.

Not the UI.
Not the export button.
Not the grids.
Not the gradients first.

The difficult part is answering these questions consistently:

Is point p inside this path?
How far is point p from the path boundary?
If this path is used as a clip, does it use the same underlying logic as visible fill?
Are contours actually closed in the emitted logic, not just “conceptually closed”?

That is the real heart of the project.

What has already been learned

These are the important lessons from debugging.

1. “Path is closed” is not enough

A contour can be “treated as closed for fill,” but the emitted sign/winding basis can still miss the actual final closing segment.

That caused endpoint leaks and thin horizontal lines.

2. Parity and winding are not interchangeable in all contexts

For some clip cases, the old parity path failed and winding fixed it.
But applying winding too broadly also broke other cases.

So the rule is not:

use winding everywhere

The rule is:

use the correct inside test for the specific class of contour
and gate it properly
3. Clip bugs and visible fill bugs are not always the same bug

A lot of time was wasted because:

some test SVGs failed in visible fill
others failed in clipping
others failed only in AA
and the same visual symptom could come from different branches

Codex must isolate:

visible fill
clip fill
edge AA
final composition
4. The clip object and the clipped object are different

A major source of confusion:

in some test cases, the “cheese” shape was the clipped object
the actual clip shape was a separate single-loop path

Fixing the cheese visible fill did nothing for the clip bug in those cases.

5. The emitted shader is the ground truth

Many apparently meaningful code changes changed nothing because the emitted shader path stayed the same.

Codex should compare emitted shader logic, not assume a source edit necessarily changed the active branch.

Important landmark versions

These are the known milestones. They matter more than many intermediate versions.

v34
vectors became smooth
clipping kind of worked
some leaking was fixed, not all
v41
gradients became correct
multiple stops supported
opacity support for gradients supported
v46
horizontal lines leaking from endpoints were gone
this is a major landmark for fill correctness
historically important reference for robust visible fill behavior
v74
grid acceptance worked
speedup for flat art became usable
performance milestone, not the core fill/clip solution
v86
collar case fixed
likely path/topology-specific insight
v122
clipping was somewhat fixed
but the horizontal line implementation was broken again
important because it proved a real clip-specific insight, even though the overall version regressed

These versions should be studied as landmarks, not treated as whole final answers.

What specific bugs were proven
Proven bug: single-loop clip path parity route was wrong

For at least one class of SVG, the clip shape was:

a single-loop path
not the cheese object
and using the old parity route caused failure

A version that switched that single-loop clip interior to winding fixed the clip tests.

That insight is real and should not be lost.

But it must be narrowly gated, not applied to every path clip.

Proven bug: visible fill winding basis was open in emitted shader

For another failure, the emitted shader used winding on a contour whose sign basis was missing the explicit closing edge.

That caused thin horizontal leaks from endpoints.

A fix that made winding use closed fillContours loops instead of an open sign chain removed those lines.

That insight is also real.

Known test cases and what they mean

Codex should treat these as canonical regression tests.


debug shapes.svg is a utility that has the following shapes that test:

cheese-like clipped object behavior
gradients and opacity
subpath/hole handling
visible fill correctness under clip
distinguishing the clipped object from the actual clip shape
isolating the actual clip path bug
proving that the real clip shape was a single-loop path
revealing horizontal endpoint leaks
showing cases where one broad fix regressed another case
stress-testing whether a “fix” is truly general or just overfit


What should be preserved from previous thinking

These are the strategic insights Codex should respect.

Visible fill strategy

Historically best compromise:

robust contour/sign topology
curve-aware boundary distance

Do not throw that away casually.

Clip strategy

Clip handling must not silently diverge from visible fill handling.

But clip handling also cannot blindly reuse visible fill logic if the path class differs.

The important thing is:

use the same core representation where possible
gate special cases carefully
Do not trust “helper exists” to mean “branch is used”

Many wasted versions changed the wrong function or wrong branch.

Codex must inspect:

which branch actually emits the active shader code
and compare generated shader blocks between versions
The likely architectural problem

The current giant file has too many overlapping strategies.

The real issue is no longer one line. It is that there are too many partially conflicting representations:

raw path segments
sign segments
fill segments
fill contours
flattened contours
topology debug contours
clip shape simplifications
primitive fast paths
special-case path branches

The file is overgrown. Codex should not assume the latest version is the best base.

What Codex should do
Preferred mission

Do not try to perfect the whole app immediately.

Instead:

1. Reconstruct a clean path fill/clip core

Focus only on:

path normalization
contour closure
inside test
edge distance
clip evaluation
2. Keep the shell/UI mostly intact

Do not rewrite:

gradients first
grid UI first
export shell first
unrelated debug UI first
3. Build a clean shared internal model

For a path, the core should expose something like:

normalized subpaths
closed contour loops used for fill
boundary geometry used for edge distance
clip-safe representation

Visible fill and clip should both use this core.

4. Add a real regression harness

At minimum, Codex should establish tests or scripted comparisons for the debug shapes


It does not need pixel-perfect test snapshots on day one, but it must at least make comparison easy.

Strong warnings for Codex
Do not blindly patch the whole 100kb file

This project already proved that patch-stacking causes:

regressions
dead branches
helper drift
missing dependencies
fixes that do nothing because the active codegen path did not change
Do not assume the visible path shown in debug is the clip path

That was a major historical confusion.

Do not treat “single-loop path” as always safe

That broke other files.
Single-loop clip winding should be gated by actual contour quality / closure, not by contour count alone.

Do not mix parity and winding by blending results

If both are used, use them as:

a stability check
a branch gate
a diagnostic
not as a pixelwise average
Do not trust conceptual closure alone

If a contour is intended to be closed for fill, the emitted basis used for inside testing must contain the explicit closing edge.

What a good end state looks like

The ideal outcome is:

heavy clipping files do not explode shader size gratuitously
debug tools do not crash independently of rendering
visible fill and clip are explainable through one coherent core
Concrete starting suggestion for Codex

Start by doing this:

Identify the smallest shared core that decides:
inside
edge distance
contour closure
Compare the landmark versions:
v46 for endpoint leak fix
v122 for single-loop clip insight
current/broken latest for what regressed
Build or refactor toward:
v46-style visible fill robustness
v122-style narrow clip fix
without importing the entire later experimental branch
Prefer a targeted subsystem rewrite over more random patching if necessary.