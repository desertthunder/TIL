---
title: Snap Priority in Vector Editors
description: Finding and ranking snap candidates during interactive transforms
tags:
  - vector-graphics
  - algorithms
  - geometry
  - user-interfaces
---

Snapping in a mature vector editor is a constraint-selection problem. During each drag,
the editor may find plausible matches between several source features on the selection and
hundreds of target features on the canvas. It must choose one transformation that matches
the user's apparent intent, remains responsive, and does not flicker between nearly equal
answers.

The nearest target is not always the best target. A node is usually more precise than a
path passing nearby, the corner the user grabbed may matter more than another corner that
can move a shorter distance, and an intersection can remove two degrees of freedom while
a line removes only one. Inkscape's `SnapManager` accordingly describes its job as finding
the best target, “which is not necessarily also the nearest target.”[^1]

## Sources, Targets, and Transforms

A snap query starts with the active tool and transform. The source features might include
path nodes, segment midpoints, bounding-box corners and edge midpoints, object centers,
rotation centers, or text baselines. Targets include the corresponding object features as
well as paths, guides, grids, page edges, alignment axes, equal-spacing positions, and
intersections.

Each source-target pair proposes a change in transform parameters:

- Point-to-point snapping proposes a translation from the source to the target.
- Point-to-line or point-to-curve snapping projects the source onto the nearest valid
  position and removes one degree of freedom.
- Two compatible line constraints can produce a fully constrained intersection.
- Scaling converts the proposed point position into scale factors around the fixed
  origin. Rotation converts it into an angle around the rotation center.

The transform has to be solved before candidates are compared. Applying an independently
chosen horizontal snap and vertical snap can produce an impossible scale or rotation.
Inkscape models translation, scale, stretch, skew, and rotation separately. Each transform
turns a snapped feature back into the transform parameters that the editor can apply.[^2]

A useful candidate record therefore contains more than a position:

```text
source feature and target feature
proposed transform and resulting point
distance and activation tolerance
number of constrained degrees of freedom
whether it came from an intersection or an active constraint
pointer-to-source distance
stable IDs for deterministic ties and visual feedback
```

## Candidate Generation

Candidate generation should be broad enough to find meaningful alternatives, then cheap
enough to run on every pointer update.

1. Derive source features permitted by the active tool and transform. Do not offer a
   bounding-box corner for a transform whose geometry cannot move that corner in the
   assumed way.
2. Convert a fixed screen-space capture radius into document coordinates using the zoom.
   A ten-pixel target should feel like ten pixels at every zoom level.
3. Query a spatial index with the swept or transformed selection bounds expanded by that
   radius. Exclude the moving selection and targets disabled by document or user settings.
4. Run exact geometry only on the returned objects: nearest points on curves, projections
   onto lines, feature coincidences, and intersections among nearby constraints.
5. Discard candidates outside their tolerance or current viewport before ranking them.

Inkscape follows this broad-phase/narrow-phase shape even though its current traversal is
not a general spatial index. It expands the moving bounding box by the object tolerance,
ignores the dragged objects, collects only intersecting object candidates, and limits the
larger alignment candidate set when it becomes expensive. Its configured tolerance is
divided by zoom before document-space comparisons.[^1]

Keep target families separate through generation. Point, curve, grid-line, guide-line,
alignment, and distribution solvers need different geometry and may yield composite
results. Inkscape first keeps the closest result from each family, derives eligible
intersections, and only then compares the finalists.[^1]

## Conflicts

A lexicographic comparator is easier to reason about than one giant weighted score. Apply
hard eligibility rules first, semantic priority next, and distance only among candidates
that are otherwise comparable. One practical ordering is:

1. Reject disabled, hidden, off-screen, self-referential, or out-of-tolerance targets.
2. Honor temporary overrides and “always snap” modes.
3. Prefer a fully constrained point or intersection over a one-axis line snap when the
   tool allows both.
4. Prefer the source feature that the pointer indicates.
5. Compare normalized transform displacement, then the second constraint distance for
   intersections.
6. Break ties by target type and stable object/feature ID.

This ordering should be a documented product choice rather than an accidental consequence
of iteration order. It also needs exceptions. A constraint introduced by holding Shift or
Ctrl is mandatory, but a candidate that merely projects onto that constraint should lose
to a real geometric target on the constraint. Coincident node and intersection targets
may share a location. Choosing the node gives the indicator a more concrete explanation.

Inkscape's comparator illustrates these policies. “Always snap” candidates cannot lose to
ordinary candidates. A fully constrained result beats a partly constrained line result.
At the same location, a node beats an intersection. Equal-distance results compare their
second snap distance and then prefer a free snap over a constrained projection.[^3]

When transforming a selection, there is another conflict: which source feature should
represent the user's intent? Inkscape records both source-to-target distance and the
source's original distance from the pointer. Its configurable weight interpolates between
the shortest snap and the source nearest the pointer. “Only snap the closest node”[^2] makes
pointer distance decisive.[^3] This avoids a distant corner hijacking a drag merely
because it happens to align perfectly.

If a scalar score is used, normalize unlike quantities first. Screen distance, angular
change, relative scale change, and semantic priority do not share units. A tuple such as

```text
(always_snap, fully_constrained, semantic_rank,
 pointer_intent_rank, normalized_transform_error, stable_id)
```

makes those comparisons visible. Do not encode hard rules as small numeric bonuses: a
large enough distance can unexpectedly overpower them.

## Stability

Ranking each pointer event independently causes chatter near a boundary. Retain the
current winner while it remains inside a slightly larger release radius, and require a
new candidate to beat it by a margin before switching. This hysteresis creates two
thresholds: one to acquire a snap and another to release it.

The retained state must refer to feature IDs, not stale coordinates. Recompute its
geometry after zoom, object, or transform changes. Clear it when the tool, modifier keys,
selection, or enabled target families change. Candidate order and tie-breaking must also
be deterministic so identical document states produce identical snaps.

Visual feedback is part of conflict resolution. Highlight the source, target, guide, or
equal-spacing relation that won. Figma, for example, separates snapping to vector
geometry, object centers and outer points, and the pixel grid, displays a red guide, and
provides a modifier to disable snapping temporarily.[^4] The indication lets users adjust
their pointer or override the system instead of guessing why an object stopped.

Snap ranking is well suited to [property-based testing](/notes/2026/08/21/prop-test/).
Useful properties include invariance under translating the entire scene, independence
from candidate enumeration order, symmetry under axis reflection, zoom-invariant
screen-space tolerance, and the rule that disabled or out-of-range targets never win.
Generated near-ties are especially valuable because most visible snapping bugs live at
priority boundaries.

[^1]: Inkscape, [`SnapManager` implementation](https://gitlab.com/inkscape/inkscape/-/raw/master/src/snap.cpp) and [class documentation](https://inkscape.gitlab.io/inkscape/doxygen/classSnapManager.html).

[^2]: Inkscape, [pure-transform snapping implementation](https://gitlab.com/inkscape/inkscape/-/raw/master/src/pure-transform.cpp).

[^3]: Inkscape, [`SnappedPoint::isOtherSnapBetter`](https://gitlab.com/inkscape/inkscape/-/raw/master/src/snapped-point.cpp).

[^4]: Figma, [“Adjust alignment, rotation, position, and dimensions”](https://help.figma.com/hc/en-us/articles/360039956914-Adjust-alignment-rotation-position-and-dimensions).
