# Traffic Jam Game -- Vehicle & Grid Design Guide

## Goal

Create a polished Traffic Escape / Match Right Bus style puzzle while
keeping a hidden logical grid.

------------------------------------------------------------------------

# Grid

-   **Cell Size:** `4`
-   **Pivot:** Center of vehicle
-   Keep the gameplay grid hidden.

------------------------------------------------------------------------

# Vehicle Occupancy

  Vehicle                       Cells
  --------------------------- -------
  Small Car                         1
  Medium Car                        2
  Long Bus                          3
  Extra Long Bus (Optional)         4

------------------------------------------------------------------------

# Recommended Dimensions (Cell Size = 4)

  Vehicle        Length   Width   Height
  ------------ -------- ------- --------
  Small             3.6     3.5      3.4
  Medium            7.6     3.5      3.4
  Long Bus         11.6     3.5      3.4
  Extra Long       15.6     3.5      3.4

> Keep width and height almost identical for all vehicles. Only length
> changes.

------------------------------------------------------------------------

# Prefab Rules

All prefabs should have **Scale = (1,1,1)**.

Do **not** create different sizes by scaling.

Instead create different meshes/prefabs with different lengths.

## Prefab Hierarchy

``` text
CarRoot
├── BoxCollider
├── Rigidbody (optional)
├── CarMover
├── CarOut
├── CarData
├── PassengerManager
└── Visual
    ├── Body
    ├── Wheels
    ├── Windows
    ├── Shadow
    └── Arrow
```

Move **CarRoot**. Never move the gameplay using the Visual object.

------------------------------------------------------------------------

# Occupancy

## Small

    [X]

Pivot at center.

## Medium

    [X][X]

Pivot between the two cells.

## Long Bus

    [X][X][X]

Pivot on the center cell.

## Extra Long

    [X][X][X][X]

Pivot between the two middle cells.

------------------------------------------------------------------------

# Visual Polish

-   Vehicle width fills about **90%** of a cell.
-   Tiny visual gap (0.15--0.25 units).
-   Optional mesh offset: ±0.08 units.
-   Optional mesh rotation: ±3°.
-   Use soft shadows.
-   Camera should let the puzzle fill roughly 75% of screen width.

------------------------------------------------------------------------

# Level Design

Prefer dense irregular shapes instead of perfect rectangles.

Good:

``` text
 □□□□□
□□□□□□
 □□□□□
```

Avoid:

``` text
□□□□□
□□□□□
□□□□□
```

------------------------------------------------------------------------

# Important

Do NOT change:

-   Grid logic
-   Occupancy system
-   A\* / movement
-   Center-based spawning

Improve only:

-   Mesh size
-   Camera
-   Shadows
-   Level layouts
-   Visual polish
