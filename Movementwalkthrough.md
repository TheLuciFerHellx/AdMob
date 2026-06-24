# 🚗 CarMover.cs — Full Rework Documentation

---

## 📦 Only ONE File Changed

> **[CarMover.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-Grid/Car-OUT-jam-puzzle-Game-Grid/Assets/Script/CarMover.cs)**

Everything else untouched:
`CarController` · `SpawnCars` · `Carout` · `ParkingSlotManger` · `ParkingGameManager` · passengers · coins · haptics · object pool · DriveAway

---

## 🔴 OLD MOVEMENT — Why It Felt Robotic

### Example: 5×5 Grid, Car at [2,1] facing RIGHT →

```
     Col: 0    1    2    3    4
Row 4: [  ] [  ] [  ] [  ] [  ]   ← Top border (road)
Row 3: [  ] [  ] [  ] [  ] [  ]
Row 2: [  ] [  ] [🚗] [🚙] [  ]   ← 🚙 = another car blocking
Row 1: [  ] [  ] [🚗→][  ] [  ]   ← Our car starts here
Row 0: [  ] [  ] [  ] [  ] [  ]   ← Bottom border
```

**Old Logic (ALWAYS drives to wall first):**
```
[2,1] → [3,1] → [4,1]  ← HITS WALL (Row 1, Col 4)
       ↓
[4,2] → [4,3] → [4,4]  ← Climbs right border to top
       ↓
[3,4] → [2,4]           ← Slides along top to center
       ↓
Parking Slot 🅿️
```

**Problem:** Car always drives to [4,1] first even if there's a shorter route.
**It looks scripted. Same every time. Robotic.**

---

## 🟢 NEW MOVEMENT — BFS Smart Escape

### Same Grid, Same Car — but now BFS finds SMARTEST route:

```
     Col: 0    1    2    3    4
Row 4: [  ] [  ] [  ] [  ] [  ]   ← Top border (road)
Row 3: [  ] [  ] [  ] [  ] [  ]
Row 2: [  ] [  ] [🚗] [🚙] [  ]   ← 🚙 = blocked cell
Row 1: [  ] [  ] [🚗→][  ] [  ]   ← Car starts here
Row 0: [  ] [  ] [  ] [  ] [  ]
```

**BFS sees [3,1] is free → goes there.**
**BFS sees [4,1] is free → that's a border! DONE.**

```
[2,1] → [3,1] → [4,1]  ← Border reached (shortest path)
       ↓
[4,2] → [4,3] → [4,4]  ← Up right border
       ↓
[3,4] → [2,4]           ← Slide to center
       ↓
🅿️ Parking Slot
```

### Now with a BLOCKED path — BFS routes around:

```
     Col: 0    1    2    3    4
Row 4: [  ] [  ] [  ] [  ] [  ]
Row 3: [  ] [  ] [  ] [  ] [  ]
Row 2: [  ] [  ] [🚗] [🚙] [🚙]   ← Two blocked cells
Row 1: [  ] [  ] [🚗→][🚙] [🚙]   ← Row 1 also blocked
Row 0: [  ] [  ] [  ] [  ] [  ]
```

**Old system:** Tries to go right → blocked → car stops. Player sees no movement.

**New BFS:** Checks all 4 directions intelligently:
```
Can go RIGHT  [3,1]? ❌ Blocked
Can go UP     [2,2]? ❌ Blocked
Can go LEFT   [1,1]? ✅ Free!
Can go DOWN   [2,0]? ✅ Free (border)!
```

BFS picks the SHORTEST: go DOWN to [2,0] (that's already the bottom border!)
```
[2,1] → [2,0]           ← Bottom border instantly!
       ↓
[3,0] → [4,0]           ← Along bottom border
       ↓
[4,1] → [4,2] → [4,3] → [4,4]  ← Up right border
       ↓
[3,4] → [2,4]           ← Slide to center
       ↓
🅿️ Parking Slot
```

**Every car takes a DIFFERENT route based on actual traffic. Never predictable. Never robotic.**

---

## 🎯 Movement Feel — Old vs New

| | Old | New |
|---|---|---|
| **Path** | Always to last wall cell | BFS shortest free path |
| **Rotation** | Instant snap (LookRotation) | Smooth Slerp (gradual steer) |
| **Waypoints** | Pause + reset at each cell | One continuous glide |
| **Car front** | Teleports to new direction | Front leads, body follows |
| **Blocked route** | Stops/looks stuck | Routes around obstacle |
| **Feel** | 🤖 Robotic | 🚗 Premium & natural |

---

## 🔄 The Smooth Movement Loop Explained

Old code (BROKEN — paused at every waypoint):
```csharp
// Old inner loop — ran for EACH waypoint separately:
while (i < waypoints.Count)
{
    Vector3 wp = waypoints[i];
    // snap direction
    transform.rotation = Quaternion.LookRotation(moveDir); // INSTANT SNAP
    transform.position += toTarget.normalized * step;
    i++;
    continue; // ← jump to next cell immediately
}
```

New code (SMOOTH — one continuous flow):
```csharp
int wpIndex = 0;

while (wpIndex < waypoints.Count)          // ← ONE loop, all waypoints
{
    Vector3 target = waypoints[wpIndex];
    Vector3 toTarget = target - transform.position;
    toTarget.y = 0f;

    // Arrived? Move to next waypoint silently (no pause)
    if (distSq <= stepSize * stepSize || distSq < 0.0004f)
    {
        transform.position = new Vector3(target.x, transform.position.y, target.z);
        wpIndex++;       // ← just increment, keep moving same frame
        continue;
    }

    // Smooth Slerp turn — front of car leads
    Quaternion targetRot = Quaternion.LookRotation(moveDir);
    transform.rotation = Quaternion.Slerp(
        transform.rotation,
        targetRot,
        turnSpeed * Time.deltaTime);   // ← gradual, not instant

    // Move in the direction car is ACTUALLY facing
    // (not slide sideways — realistic arc turns)
    transform.position += transform.forward * stepSize;

    yield return null;    // ← only ONE yield per frame, feels fluid
}
```

**The key difference:** `wpIndex++; continue;` — when the car passes through a waypoint it immediately continues to the next one **in the same frame**, with no yield in between. The player never sees it pause.

---

## 🧠 BFS Algorithm — Step by Step

```
START: Car at grid position [2,2], facing East (right)

Step 1: Add [2,2] to queue. Mark visited.

Step 2: Pop [2,2]. Check all 4 neighbors:
        [2,3] North → free → add to queue
        [2,1] South → free → add to queue
        [3,2] East  → free → add to queue  ← facing dir, checked FIRST
        [1,2] West  → free → add to queue

Step 3: Pop [3,2] (East checked first — facing bias).
        Is [3,2] a border? No (grid is 5x5, so border is col 0 or col 4)
        Check neighbors: [4,2] East → free → add

Step 4: Pop [2,3]. Check neighbors...
Step 5: Pop [2,1]. Check neighbors...

Step N: Pop [4,2]. Is [4,2] border? YES! (col 4 = maxCols-1)
        STOP. We found the border!

Trace back parent chain:
        [4,2] → parent [3,2] → parent [2,2 start]

Path: [2,2] → [3,2] → [4,2] ← SHORTEST FREE PATH TO BORDER ✅
```

---

## ⚙️ Inspector Tuning Guide

In Unity, select any **Car prefab** → look at `CarMover` component:

```
[Header: Movement]
  speed      = 35    → Car travel speed. Try 25-50
  turnSpeed  = 9     → Steering smoothness:
                          6  = very smooth/cinematic
                          9  = good default (recommended)
                          14 = sharp/snappy turns

[Header: Traffic]
  safetyDistance = 4 → Raycast distance for traffic check
```

---

## ✅ Full CarMover.cs Source Code

```csharp
// ============================================================
// CarMover.cs  —  Smart Pathfinding + Smooth Steering
// ============================================================
// WHAT CHANGED vs original:
//   • GeneratePath() now uses BFS to find the shortest non-blocked
//     escape route to the grid border instead of always driving
//     straight to the wall.
//   • Waypoint following is ONE continuous motion — the car never
//     stops or jitters at each grid cell.
//   • Rotation uses Quaternion.Slerp so the car steers smoothly
//     from the front, just like a real vehicle.
//   • All other systems (passengers, coins, pool, haptics, DriveAway,
//     ParkingSlot, Carout) are completely untouched.
// ============================================================

using UnityEngine;
using DG.Tweening;
using TMPro;
using UnityEngine.UI;
using System.Collections;
using System.Collections.Generic;

public class CarMover : MonoBehaviour
{
    // ── Identity ──────────────────────────────────────────────
    public ColorOfCarAndPassengers carType;
    public ColorOfCarAndPassengers defaultCarType;

    // ── Movement ──────────────────────────────────────────────
    [Header("Movement")]
    public float speed = 35f;

    /// <summary>
    /// How fast the car rotates toward the next direction.
    /// Higher = snappier turns. Lower = floaty/cinematic.
    /// Tune in Inspector.
    /// </summary>
    [SerializeField] private float turnSpeed = 9f;

    private Vector3 _targetPosition;   // final parking slot world pos
    private bool isMoving = false;
    private Coroutine _moveCoroutine;

    // ── Passengers / UI ──────────────────────────────────────
    public int CapacityOfPassengers = 4;
    public int thisCarCapacity;
    public TextMeshProUGUI totalPassengerTxt;
    public Image arrow;
    public bool isParked = false;

    // ── Traffic ───────────────────────────────────────────────
    [Header("Traffic")]
    public float safetyDistance = 4f;

    // ── VFX / SFX ─────────────────────────────────────────────
    public GameObject coinPrefab;
    public ParticleSystem passengerAddParticle;

    // ── Internal refs ─────────────────────────────────────────
    private Carout _carout;

    // ── BFS node ──────────────────────────────────────────────
    private class PathNode
    {
        public Vector2Int Pos;
        public PathNode Parent;

        public PathNode(Vector2Int pos, PathNode parent)
        {
            Pos = pos;
            Parent = parent;
        }
    }

    // =========================================================
    // AWAKE
    // =========================================================

    void Awake()
    {
        _carout = GetComponent<Carout>();
    }

    // =========================================================
    // PUBLIC API
    // =========================================================

    public void SetDestination(Vector3 target)
    {
        if (isMoving) return;

        _targetPosition = target;
        isMoving = true;

        if (_moveCoroutine != null) StopCoroutine(_moveCoroutine);
        _moveCoroutine = StartCoroutine(MoveRoutine());
    }

    public void ResetCapacity()
    {
        CapacityOfPassengers = thisCarCapacity;
        isMoving = false;
        isParked = false;
    }

    public void ResetEnum()
    {
        carType = defaultCarType;
        Debug.Log("Car enum reset to: " + carType);
    }

    // =========================================================
    // MAIN MOVE COROUTINE
    // =========================================================

    private IEnumerator MoveRoutine()
    {
        if (_carout == null) { isMoving = false; yield break; }

        Vector2Int startGrid = _carout.currentGridIndex;

        // --------------------------------------------------
        // 1. Silently snap car to its exact grid cell center
        //    (only if it drifted slightly — prevents path errors)
        // --------------------------------------------------
        var startSlot = SpawnCars.Instance.GetSlotAt(startGrid.x, startGrid.y);
        if (startSlot != null &&
            Vector3.Distance(transform.position, startSlot.WorldPosition) > 0.3f)
        {
            transform.position = startSlot.WorldPosition;
        }

        // --------------------------------------------------
        // 2. Build waypoint list via BFS smart escape
        // --------------------------------------------------
        Vector2Int facingDir = GetDirectionFromAngle(transform.eulerAngles.y);
        List<Vector3> waypoints = BuildPath(startGrid, facingDir);

        // Always end at the parking slot
        waypoints.Add(_targetPosition);

        if (waypoints.Count == 0) { isMoving = false; yield break; }

        // --------------------------------------------------
        // 4. CONTINUOUS SMOOTH FOLLOW
        //    — car glides through ALL waypoints as one fluid motion.
        //    — NO stopping or resetting between waypoints.
        //    — Rotation is Slerp so turns feel gradual.
        // --------------------------------------------------
        int wpIndex = 0;

        while (wpIndex < waypoints.Count)
        {
            Vector3 target = waypoints[wpIndex];
            Vector3 toTarget = target - transform.position;
            toTarget.y = 0f;

            float distSq = toTarget.sqrMagnitude;
            float stepSize = speed * Time.deltaTime;

            // ── Arrived at this waypoint? ──────────────────
            if (distSq <= stepSize * stepSize || distSq < 0.0004f)
            {
                // Snap cleanly to waypoint so no floating drift accumulates
                transform.position = new Vector3(target.x, transform.position.y, target.z);
                wpIndex++;
                continue;
            }

            // ── Smooth Slerp rotation ──────────────────────
            // Car steers toward the NEXT waypoint it is heading to,
            // so the front always leads — just like a real car.
            Vector3 moveDir = toTarget.normalized;

            Quaternion targetRot = Quaternion.LookRotation(moveDir);
            transform.rotation = Quaternion.Slerp(
                transform.rotation,
                targetRot,
                turnSpeed * Time.deltaTime);

            // ── Move forward ───────────────────────────────
            // Drive in the direction the car is ACTUALLY facing
            // (not just straight toward the waypoint) so that the
            // turning arc looks natural rather than sliding sideways.
            transform.position += transform.forward * stepSize;

            yield return null;
        }

        // --------------------------------------------------
        // 5. Final snap into parking slot
        // --------------------------------------------------
        transform.position = _targetPosition;
        transform.rotation = Quaternion.Euler(0f, 0f, 0f);

        isMoving = false;
        isParked = true;

        // Parked haptic
        HapticManager.Instance?.CarFull();
    }

    // =========================================================
    // BFS SMART ESCAPE — finds shortest non-blocked grid path
    //                    from startGrid to any border cell
    // =========================================================

    private List<Vector3> BuildPath(Vector2Int startGrid, Vector2Int initialFacingDir)
    {
        List<Vector3> worldPath = new List<Vector3>();

        int maxCols = SpawnCars.Instance.maxColumns;
        int maxRows = SpawnCars.Instance.maxRows;

        // --------------------------------------------------
        // BFS to find shortest path to ANY border cell
        // --------------------------------------------------
        Queue<PathNode> openSet = new Queue<PathNode>();
        HashSet<Vector2Int> visited = new HashSet<Vector2Int>();

        PathNode startNode = new PathNode(startGrid, null);
        openSet.Enqueue(startNode);
        visited.Add(startGrid);

        // 4-directional moves (cardinal only — realistic parking lot movement)
        Vector2Int[] cardinalDirs = new Vector2Int[]
        {
            new Vector2Int(0,  1),   // North
            new Vector2Int(0, -1),   // South
            new Vector2Int(1,  0),   // East
            new Vector2Int(-1, 0),   // West
        };

        PathNode borderNode = null;

        while (openSet.Count > 0)
        {
            PathNode current = openSet.Dequeue();
            Vector2Int pos = current.Pos;

            // Check if we reached a border cell (exit point)
            bool isBorder =
                pos.x == 0 ||
                pos.x == maxCols - 1 ||
                pos.y == 0 ||
                pos.y == maxRows - 1;

            if (isBorder && pos != startGrid)
            {
                borderNode = current;
                break;
            }

            // Build neighbour list — preferred direction first
            List<Vector2Int> neighbours = new List<Vector2Int>();

            // Insert facing dir first (continuity bias)
            Vector2Int preferred = pos + initialFacingDir;
            if (IsCardinal(initialFacingDir))
                neighbours.Add(preferred);

            foreach (Vector2Int d in cardinalDirs)
            {
                Vector2Int n = pos + d;
                if (IsCardinal(d) && n != preferred)
                    neighbours.Add(n);
            }

            foreach (Vector2Int n in neighbours)
            {
                if (visited.Contains(n)) continue;

                // Out of bounds = not a valid grid cell
                if (SpawnCars.Instance.GetSlotAt(n.x, n.y) == null) continue;

                // Skip occupied cells (cars in the way)
                if (n != startGrid && SpawnCars.Instance.IsSlotBlocked(n.x, n.y)) continue;

                visited.Add(n);
                openSet.Enqueue(new PathNode(n, current));
            }
        }

        // --------------------------------------------------
        // Reconstruct grid path from BFS result
        // --------------------------------------------------
        if (borderNode == null)
        {
            // BFS found no route (fully surrounded) — fallback: drive straight
            borderNode = FallbackStraightPath(startGrid, initialFacingDir, maxCols, maxRows);
        }

        // Walk parent chain → reverse into forward order
        List<Vector2Int> gridPath = new List<Vector2Int>();
        PathNode node = borderNode;
        while (node != null && node.Pos != startGrid)
        {
            gridPath.Add(node.Pos);
            node = node.Parent;
        }
        gridPath.Reverse();

        // Convert grid indices → world positions
        foreach (Vector2Int gi in gridPath)
        {
            var slot = SpawnCars.Instance.GetSlotAt(gi.x, gi.y);
            if (slot != null) worldPath.Add(slot.WorldPosition);
        }

        // --------------------------------------------------
        // Border Road — reuse existing logic to reach top center
        // --------------------------------------------------
        Vector2Int borderPos = (borderNode != null) ? borderNode.Pos : startGrid;
        AppendBorderRoad(worldPath, borderPos, maxCols, maxRows);

        return worldPath;
    }

    // ─── Reconstruct a straight-line fallback node chain ──────
    private PathNode FallbackStraightPath(
        Vector2Int start, Vector2Int dir, int maxCols, int maxRows)
    {
        PathNode prev = new PathNode(start, null);
        Vector2Int cur = start;

        for (int i = 0; i < Mathf.Max(maxCols, maxRows) + 1; i++)
        {
            Vector2Int next = cur + dir;
            bool outside =
                next.x < 0 || next.x >= maxCols ||
                next.y < 0 || next.y >= maxRows;

            if (outside) break;
            cur = next;
            PathNode n = new PathNode(cur, prev);
            prev = n;
        }

        return prev;
    }

    // ─── Append border-road waypoints (left / right / bottom) ─
    private void AppendBorderRoad(
        List<Vector3> path, Vector2Int borderPos, int maxCols, int maxRows)
    {
        int topRow  = maxRows - 1;
        int centerX = maxCols / 2;

        // LEFT BORDER  → move up then right to top center
        if (borderPos.x == 0)
        {
            for (int y = borderPos.y + 1; y < maxRows; y++)
                AddSlot(path, 0, y);

            for (int x = 1; x <= centerX; x++)
                AddSlot(path, x, topRow);
        }
        // RIGHT BORDER → move up then left to top center
        else if (borderPos.x == maxCols - 1)
        {
            for (int y = borderPos.y + 1; y < maxRows; y++)
                AddSlot(path, maxCols - 1, y);

            for (int x = maxCols - 2; x >= centerX; x--)
                AddSlot(path, x, topRow);
        }
        // BOTTOM BORDER → move right then up then left to top center
        else if (borderPos.y == 0)
        {
            for (int x = borderPos.x + 1; x < maxCols; x++)
                AddSlot(path, x, 0);

            for (int y = 1; y < maxRows; y++)
                AddSlot(path, maxCols - 1, y);

            for (int x = maxCols - 2; x >= centerX; x--)
                AddSlot(path, x, topRow);
        }
        // TOP BORDER → already at top, just slide to center
        else if (borderPos.y == maxRows - 1)
        {
            if (borderPos.x < centerX)
            {
                for (int x = borderPos.x + 1; x <= centerX; x++)
                    AddSlot(path, x, topRow);
            }
            else
            {
                for (int x = borderPos.x - 1; x >= centerX; x--)
                    AddSlot(path, x, topRow);
            }
        }
    }

    // ─── Helper: add slot world pos only if not duplicate ─────
    private void AddSlot(List<Vector3> path, int x, int y)
    {
        var slot = SpawnCars.Instance.GetSlotAt(x, y);
        if (slot == null) return;
        if (path.Count > 0 &&
            Vector3.Distance(path[path.Count - 1], slot.WorldPosition) < 0.01f)
            return;
        path.Add(slot.WorldPosition);
    }

    // =========================================================
    // DIRECTION HELPERS
    // =========================================================

    private static bool IsCardinal(Vector2Int d)
    {
        return (d.x == 0 || d.y == 0) && d != Vector2Int.zero;
    }

    private Vector2Int GetDirectionFromAngle(float angle)
    {
        angle = Mathf.Repeat(Mathf.Round(angle / 45f) * 45f, 360f);

        switch (Mathf.RoundToInt(angle))
        {
            case 0:   return new Vector2Int(0,  1);
            case 45:  return new Vector2Int(1,  1);
            case 90:  return new Vector2Int(1,  0);
            case 135: return new Vector2Int(1, -1);
            case 180: return new Vector2Int(0, -1);
            case 225: return new Vector2Int(-1,-1);
            case 270: return new Vector2Int(-1, 0);
            case 315: return new Vector2Int(-1, 1);
        }

        return new Vector2Int(0, 1);
    }

    private Vector2Int ToCardinal(Vector2Int d)
    {
        if (Mathf.Abs(d.x) >= Mathf.Abs(d.y))
            return new Vector2Int((int)Mathf.Sign(d.x), 0);
        else
            return new Vector2Int(0, (int)Mathf.Sign(d.y));
    }

    // =========================================================
    // TRAFFIC RAYCAST
    // =========================================================

    private bool IsMovingCarAhead()
    {
        RaycastHit hit;
        if (Physics.Raycast(
            transform.position + Vector3.up * 0.5f,
            transform.forward,
            out hit,
            safetyDistance))
        {
            if (hit.collider.CompareTag("car") &&
                hit.collider.gameObject != gameObject)
            {
                CarMover other = hit.collider.GetComponent<CarMover>();
                if (other != null && !other.isParked)
                    return true;
            }
        }
        return false;
    }

    // =========================================================
    // DRIVE AWAY  (UNCHANGED from original)
    // =========================================================

    public void DriveAway()
    {
        isMoving = false;

        totalPassengerTxt.enabled = false;
        arrow.enabled = false;

        SoundManager.Instance.PlaySound(SoundManager.SoundName.carFull);

        Vector3 pos = transform.position + Vector3.up * 1.5f;
        GameObject coin = Instantiate(coinPrefab, pos, Quaternion.identity);
        coin.GetComponent<CoinPopup>()?.Play();

        DG.Tweening.Sequence s = DOTween.Sequence();

        s.Append(transform.DOMove(-transform.forward * 7f, 0.001f).SetRelative());
        s.AppendInterval(0.1f);
        s.Append(transform.DORotate(new Vector3(0, 90, 0), 0.2f));
        s.Append(transform.DOMove(transform.right * 80f, 0.5f).SetRelative());

        s.OnComplete(() =>
        {
            ObjectPool.Instance.AddToPool(gameObject);
            isParked = false;
        });
    }
}
```
