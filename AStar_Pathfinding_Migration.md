# A* Pathfinding Migration Plan

This document outlines the design, architecture, and step-by-step implementation for migrating the parking puzzle game from a hardcoded vehicle exit path system to a dynamic A* pathfinding system with smooth spline-based movement and rotation.

---

## 1. Project Analysis

### Current Architecture
Currently, vehicles exit the parking grid using a hardcoded sequence of grid actions defined in `GeneratePath()` inside [CarMover.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/CarMover.cs). This function calculates waypoints by moving directly to the borders and then executing hardcoded turns towards the exit slot.

Blockage detection is handled in `CheckBlockageForCar()` inside [SpawnCars.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/SpawnCars.cs), which projects a raycast/grid check in the vehicle's forward direction to see if another vehicle blocks its immediate path.

### Why Scripts Must Change
1. **[CarMover.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/CarMover.cs)**:
   - Must remove the hardcoded `GeneratePath()` system entirely.
   - Must query `AStarPathfinder` for pathfinding within the grid.
   - Must integrate with `PathSmoother` to convert discrete grid coordinates into a smooth spline.
   - Must update its movement and rotation loops to follow the curve tangent smoothly using `Quaternion.Slerp`.

2. **[Carout.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/Carout.cs)**:
   - Blockage detection must be updated. Instead of checking if a single grid cell in front is blocked, it must verify whether a complete A* path to the exit exists.
   - If a path exists, it is not blocked; otherwise, it is blocked (triggering the tackle/nudge animation).

3. **[SpawnCars.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/SpawnCars.cs)**:
   - `GridSlot` must be updated to track whether it is an `IsExit` cell.
   - Grid initialization must mark exit cells (by default, the cell at `(maxColumns / 2, maxRows - 1)`).

---

## 2. New Architecture

The pathfinding system utilizes a custom, allocation-free A* pathfinder. It searches the grid cells to find the shortest path from the vehicle's position to any marked exit cell.

### Key Rules Implemented
- **8-Directional Neighbors**: Up, Down, Left, Right, Top-Left, Top-Right, Bottom-Left, Bottom-Right.
- **Costs**: Straight movement = 10, Diagonal movement = 14.
- **Corner Cutting Prevention**: Diagonal movements are forbidden if either the adjacent horizontal or vertical cell is blocked.
- **Allocation-Free Nodes**: To prevent garbage collection spikes on mobile, the A* system uses a pre-allocated grid of `PathNode` structures. It resets these nodes before each search instead of instantiating new objects.
- **Path Smoothing**: Once the path is generated, the discrete grid waypoints are converted into a smooth curved path using a **Catmull-Rom spline** interpolator.
- **Continuous Tangent Rotation**: Vehicles face the movement direction by calculating the tangent of the spline and smoothly slerping towards it.

### Class Architecture
- **`CarSize`**: Enum defining vehicle cell sizes (Small = 1, Medium = 2, Large = 3, Bus = 4).
- **`AStarPathfinder`**: Singleton class managing path node grids, cost calculations, open/closed lists, and A* search.
- **`PathSmoother`**: Mathematical helper class that interpolates a discrete point list into a smooth Catmull-Rom spline.

---

## 3. New Scripts

These scripts are completely new and must be created in the `Assets/Script/` directory.

### `CarSize.cs`
Create this file at [CarSize.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/CarSize.cs):
```csharp
public enum CarSize
{
    Small = 1,
    Medium = 2,
    Large = 3,
    Bus = 4
}
```

### `AStarPathfinder.cs`
Create this file at [AStarPathfinder.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/AStarPathfinder.cs):
```csharp
using System.Collections.Generic;
using UnityEngine;

public class AStarPathfinder : MonoBehaviour
{
    public static AStarPathfinder Instance { get; private set; }

    private class PathNode
    {
        public Vector2Int gridIndex;
        public int gCost;
        public int hCost;
        public int fCost => gCost + hCost;
        public PathNode parent;

        public void Reset()
        {
            gCost = int.MaxValue;
            hCost = 0;
            parent = null;
        }
    }

    private PathNode[,] nodeGrid;
    private int gridCols = 0;
    private int gridRows = 0;

    // Reusable lists to prevent garbage collection allocations at runtime
    private List<PathNode> openList = new List<PathNode>();
    private bool[,] closedGrid;
    private bool[,] openGrid;
    private List<Vector2Int> neighborsBuffer = new List<Vector2Int>(8);

    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }

    private void PrepareGrid(int cols, int rows)
    {
        if (nodeGrid == null || gridCols != cols || gridRows != rows)
        {
            gridCols = cols;
            gridRows = rows;
            nodeGrid = new PathNode[cols, rows];
            closedGrid = new bool[cols, rows];
            openGrid = new bool[cols, rows];

            for (int x = 0; x < cols; x++)
            {
                for (int y = 0; y < rows; y++)
                {
                    nodeGrid[x, y] = new PathNode { gridIndex = new Vector2Int(x, y) };
                }
            }
        }

        for (int x = 0; x < cols; x++)
        {
            for (int y = 0; y < rows; y++)
            {
                nodeGrid[x, y].Reset();
            }
        }
    }

    public List<Vector2Int> FindPath(Vector2Int startCoords, List<Vector2Int> ignoreOccupiedCells)
    {
        if (SpawnCars.Instance == null) return null;

        int cols = SpawnCars.Instance.maxColumns;
        int rows = SpawnCars.Instance.maxRows;

        PrepareGrid(cols, rows);

        // Reset grids
        openList.Clear();
        System.Array.Clear(closedGrid, 0, closedGrid.Length);
        System.Array.Clear(openGrid, 0, openGrid.Length);

        // Find exits
        List<Vector2Int> exitCells = GetExitCells(cols, rows);
        if (exitCells.Count == 0)
        {
            Debug.LogError("A* Pathfinder: No exit cells defined on the grid!");
            return null;
        }

        // Start node setup
        PathNode startNode = nodeGrid[startCoords.x, startCoords.y];
        startNode.gCost = 0;
        startNode.hCost = GetMinDistanceToExits(startCoords, exitCells);
        
        openList.Add(startNode);
        openGrid[startCoords.x, startCoords.y] = true;

        PathNode targetNode = null;

        while (openList.Count > 0)
        {
            // Pick lowest F cost node
            PathNode currentNode = openList[0];
            for (int i = 1; i < openList.Count; i++)
            {
                if (openList[i].fCost < currentNode.fCost || 
                    (openList[i].fCost == currentNode.fCost && openList[i].hCost < currentNode.hCost))
                {
                    currentNode = openList[i];
                }
            }

            openList.Remove(currentNode);
            openGrid[currentNode.gridIndex.x, currentNode.gridIndex.y] = false;
            closedGrid[currentNode.gridIndex.x, currentNode.gridIndex.y] = true;

            // Target reached check
            var slot = SpawnCars.Instance.GetSlotAt(currentNode.gridIndex.x, currentNode.gridIndex.y);
            if (slot != null && slot.IsExit)
            {
                targetNode = currentNode;
                break;
            }

            // Neighbors lookup
            List<Vector2Int> neighbors = GetNeighbors(currentNode.gridIndex);
            for (int i = 0; i < neighbors.Count; i++)
            {
                Vector2Int neighborCoords = neighbors[i];

                if (closedGrid[neighborCoords.x, neighborCoords.y])
                    continue;

                // Walkable checks
                bool isOccupied = SpawnCars.Instance.IsSlotBlocked(neighborCoords.x, neighborCoords.y);
                if (isOccupied && ignoreOccupiedCells != null && ignoreOccupiedCells.Contains(neighborCoords))
                {
                    isOccupied = false;
                }

                if (isOccupied)
                    continue;

                // Corner cutting prevention
                if (IsDiagonalMove(currentNode.gridIndex, neighborCoords))
                {
                    if (IsCornerCutBlocked(currentNode.gridIndex, neighborCoords, ignoreOccupiedCells))
                    {
                        continue;
                    }
                }

                // Cost updates
                int moveCost = GetDistance(currentNode.gridIndex, neighborCoords);
                int newCostToNeighbor = currentNode.gCost + moveCost;
                PathNode neighborNode = nodeGrid[neighborCoords.x, neighborCoords.y];

                if (newCostToNeighbor < neighborNode.gCost || !openGrid[neighborCoords.x, neighborCoords.y])
                {
                    neighborNode.gCost = newCostToNeighbor;
                    neighborNode.hCost = GetMinDistanceToExits(neighborCoords, exitCells);
                    neighborNode.parent = currentNode;

                    if (!openGrid[neighborCoords.x, neighborCoords.y])
                    {
                        openList.Add(neighborNode);
                        openGrid[neighborCoords.x, neighborCoords.y] = true;
                    }
                }
            }
        }

        if (targetNode != null)
        {
            return RetracePath(startNode, targetNode);
        }

        return null; // Blocked / No path exists
    }

    private List<Vector2Int> GetNeighbors(Vector2Int position)
    {
        neighborsBuffer.Clear();

        for (int x = -1; x <= 1; x++)
        {
            for (int y = -1; y <= 1; y++)
            {
                if (x == 0 && y == 0)
                    continue;

                int checkX = position.x + x;
                int checkY = position.y + y;

                if (checkX >= 0 && checkX < gridCols && checkY >= 0 && checkY < gridRows)
                {
                    if (SpawnCars.Instance.GetSlotAt(checkX, checkY) != null)
                    {
                        neighborsBuffer.Add(new Vector2Int(checkX, checkY));
                    }
                }
            }
        }

        return neighborsBuffer;
    }

    private bool IsDiagonalMove(Vector2Int a, Vector2Int b)
    {
        return a.x != b.x && a.y != b.y;
    }

    private bool IsCornerCutBlocked(Vector2Int from, Vector2Int to, List<Vector2Int> ignoreCells)
    {
        int dx = to.x - from.x;
        int dy = to.y - from.y;

        Vector2Int hNeighbor = new Vector2Int(from.x + dx, from.y);
        Vector2Int vNeighbor = new Vector2Int(from.x, from.y + dy);

        return IsCellBlocked(hNeighbor, ignoreCells) || IsCellBlocked(vNeighbor, ignoreCells);
    }

    private bool IsCellBlocked(Vector2Int coords, List<Vector2Int> ignoreCells)
    {
        if (coords.x < 0 || coords.x >= gridCols || coords.y < 0 || coords.y >= gridRows)
            return true;

        if (SpawnCars.Instance.GetSlotAt(coords.x, coords.y) == null)
            return true;

        bool occupied = SpawnCars.Instance.IsSlotBlocked(coords.x, coords.y);
        if (occupied && ignoreCells != null && ignoreCells.Contains(coords))
        {
            occupied = false;
        }

        return occupied;
    }

    private int GetDistance(Vector2Int a, Vector2Int b)
    {
        int dstX = Mathf.Abs(a.x - b.x);
        int dstY = Mathf.Abs(a.y - b.y);

        if (dstX > dstY)
            return 14 * dstY + 10 * (dstX - dstY);
        return 14 * dstX + 10 * (dstY - dstX);
    }

    private int GetMinDistanceToExits(Vector2Int node, List<Vector2Int> exits)
    {
        int min = int.MaxValue;
        for (int i = 0; i < exits.Count; i++)
        {
            int d = GetDistance(node, exits[i]);
            if (d < min) min = d;
        }
        return min;
    }

    private List<Vector2Int> GetExitCells(int cols, int rows)
    {
        List<Vector2Int> exits = new List<Vector2Int>();
        for (int x = 0; x < cols; x++)
        {
            for (int y = 0; y < rows; y++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(x, y);
                if (slot != null && slot.IsExit)
                {
                    exits.Add(slot.GridIndex);
                }
            }
        }
        return exits;
    }

    private List<Vector2Int> RetracePath(PathNode startNode, PathNode endNode)
    {
        List<Vector2Int> path = new List<Vector2Int>();
        PathNode currentNode = endNode;

        while (currentNode != startNode)
        {
            path.Add(currentNode.gridIndex);
            currentNode = currentNode.parent;
        }
        path.Add(startNode.gridIndex);
        path.Reverse();
        return path;
    }
}
```

### `PathSmoother.cs`
Create this file at [PathSmoother.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/PathSmoother.cs):
```csharp
using System.Collections.Generic;
using UnityEngine;

public static class PathSmoother
{
    /// <summary>
    /// Generates a smooth Catmull-Rom spline path from a list of discrete waypoints.
    /// </summary>
    public static List<Vector3> SmoothPath(List<Vector3> path, int pointsPerSegment = 8)
    {
        if (path == null || path.Count < 2)
            return path;

        List<Vector3> smoothPoints = new List<Vector3>();

        // Create control points and pad the ends to compute boundary tangents
        List<Vector3> controlPoints = new List<Vector3>(path.Count + 2);
        
        Vector3 startDirection = (path[1] - path[0]).normalized;
        controlPoints.Add(path[0] - startDirection * 2f);
        
        controlPoints.AddRange(path);
        
        int lastIndex = path.Count - 1;
        Vector3 endDirection = (path[lastIndex] - path[lastIndex - 1]).normalized;
        controlPoints.Add(path[lastIndex] + endDirection * 2f);

        // Interpolate along control segments
        for (int i = 1; i < controlPoints.Count - 2; i++)
        {
            Vector3 p0 = controlPoints[i - 1];
            Vector3 p1 = controlPoints[i];
            Vector3 p2 = controlPoints[i + 1];
            Vector3 p3 = controlPoints[i + 2];

            for (int j = 0; j < pointsPerSegment; j++)
            {
                float t = (float)j / pointsPerSegment;
                smoothPoints.Add(GetCatmullRomPosition(t, p0, p1, p2, p3));
            }
        }

        // Snap precisely to target point
        smoothPoints.Add(path[path.Count - 1]);
        return smoothPoints;
    }

    private static Vector3 GetCatmullRomPosition(float t, Vector3 p0, Vector3 p1, Vector3 p2, Vector3 p3)
    {
        float t2 = t * t;
        float t3 = t2 * t;

        float a = 0.5f * (-t3 + 2f * t2 - t);
        float b = 0.5f * (3f * t3 - 5f * t2 + 2f);
        float c = 0.5f * (-3f * t3 + 4f * t2 + t);
        float d = 0.5f * (t3 - t2);

        return p0 * a + p1 * b + p2 * c + p3 * d;
    }
}
```

---

## 4. Modified Scripts

### `CarMover.cs`
Update this file completely at [CarMover.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/CarMover.cs):
```csharp
using Unity.VisualScripting;
using UnityEngine;
using UnityEngine.Video;
using DG.Tweening;
using TMPro;
using UnityEngine.UI;
using System.Collections;
using System.Collections.Generic;

public class CarMover : MonoBehaviour
{
    public ColorOfCarAndPassengers carType;
    public ColorOfCarAndPassengers defaultCarType;

    private Vector3 targetPosition;

    public float speed = 35f;

    private bool isMoving = false;

    [Header("Car Size")]
    public CarSize carSize = CarSize.Small;

    private Vector3 originalScale;
    [HideInInspector]
    public CarMover originalPrefab;

    public int CapacityOfPassengers = 4;
    public int thisCarCapacity;

    public TextMeshProUGUI totalPassengerTxt;
    public Image arrow;

    public bool isParked = false;

    [Header("Traffic")]
    public float safetyDistance = 4f;

    public GameObject coinPrefab;

    public ParticleSystem passengerAddParticle;
    private Carout carout;

    void Awake()
    {
        carout = GetComponent<Carout>();
        originalScale = transform.localScale;
    }

    public Vector3 GetOriginalScale()
    {
        return originalScale;
    }

    public void SetDestination(Vector3 target)
    {
        if (isMoving) return;

        targetPosition = target;
        isMoving = true;
        StartCoroutine(MoveRoutine());
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

    private IEnumerator MoveRoutine()
    {
        if (carout == null)
        {
            yield break;
        }

        Vector2Int currentGrid = carout.currentGridIndex;

        // Snap to center of starting slot smoothly
        var currentSlot = SpawnCars.Instance.GetSlotAt(currentGrid.x, currentGrid.y);
        if (currentSlot != null)
        {
            while (Vector3.Distance(transform.position, currentSlot.WorldPosition) > 0.02f)
            {
                transform.position = Vector3.MoveTowards(
                    transform.position,
                    currentSlot.WorldPosition,
                    speed * Time.deltaTime);

                yield return null;
            }

            transform.position = currentSlot.WorldPosition;
        }

        // Generate Path using A*
        if (AStarPathfinder.Instance == null)
        {
            Debug.LogError("AStarPathfinder.Instance is null! Cannot plan A* path.");
            isMoving = false;
            yield break;
        }

        List<Vector2Int> gridPath = AStarPathfinder.Instance.FindPath(currentGrid, null);
        List<Vector3> waypoints = new List<Vector3>();

        if (gridPath != null)
        {
            for (int i = 0; i < gridPath.Count; i++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(gridPath[i].x, gridPath[i].y);
                if (slot != null)
                {
                    waypoints.Add(slot.WorldPosition);
                }
            }
        }

        // Append final parking slot outside the grid bounds
        waypoints.Add(targetPosition);

        // Smooth waypoints into a premium spline curve
        List<Vector3> smoothWaypoints = PathSmoother.SmoothPath(waypoints, 8);

        // Initial Rotation Alignment
        if (smoothWaypoints.Count > 0)
        {
            Vector3 firstDir = (smoothWaypoints[0] - transform.position).normalized;
            firstDir.y = 0;

            if (firstDir != Vector3.zero)
            {
                transform.rotation = Quaternion.LookRotation(firstDir);
            }
        }

        // Follow splined path smoothly
        int wpIndex = 0;
        while (wpIndex < smoothWaypoints.Count)
        {
            Vector3 wp = smoothWaypoints[wpIndex];
            Vector3 toTarget = wp - transform.position;
            float step = speed * Time.deltaTime;

            if (toTarget.sqrMagnitude <= step * step)
            {
                transform.position = wp;
                wpIndex++;
                continue;
            }

            Vector3 moveDir = toTarget.normalized;
            transform.position += moveDir * step;

            // Rotation tracking with smooth Slerp interpolation
            if (moveDir != Vector3.zero)
            {
                Quaternion targetRot = Quaternion.LookRotation(moveDir);
                transform.rotation = Quaternion.Slerp(transform.rotation, targetRot, 15f * Time.deltaTime);
            }

            yield return null;
        }

        // Perfect snap to parking spot
        transform.position = targetPosition;
        transform.rotation = Quaternion.Euler(0, 0, 0);

        isMoving = false;
        isParked = true;

        HapticManager.Instance?.CarFull();
    }

    private bool IsMovingCarAhead()
    {
        RaycastHit hit;

        if (Physics.Raycast(
            transform.position + Vector3.up * 0.5f,
            transform.forward,
            out hit,
            safetyDistance))
        {
            if (hit.collider.CompareTag("car"))
            {
                if (hit.collider.gameObject == gameObject)
                    return false;

                CarMover other = hit.collider.GetComponent<CarMover>();
                if (other != null && !other.isParked)
                {
                    return true;
                }
            }
        }

        return false;
    }

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

        // Move back
        s.Append(
            transform.DOMove(
                -transform.forward * 7f,
                0.001f).SetRelative());

        // Wait
        s.AppendInterval(0.1f);

        // Turn right
        s.Append(
            transform.DORotate(
                new Vector3(0, 90, 0),
                0.2f));

        // Drive away
        s.Append(
            transform.DOMove(
                transform.right * 80f,
                0.5f).SetRelative());

        s.OnComplete(() =>
        {
            ObjectPool.Instance.AddToPool(gameObject);
            isParked = false;
        });
    }
}
```

### `Carout.cs`
Update this file completely at [Carout.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/Carout.cs):
```csharp
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;
using UnityEngine;
using UnityEngine.Video;
using DG.Tweening;

public class Carout : MonoBehaviour
{
    public float rayDis = 5f;

    public float dashSpeed = 0.2f;
    public float returnSpeed = 0.5f;

    public bool isBlocked { get; private set; }

    public Vector2Int currentGridIndex; 

    private List<Vector2Int> occupiedCells = new List<Vector2Int>();

    public void SetOccupiedCells(List<Vector2Int> cells)
    {
        occupiedCells = new List<Vector2Int>(cells);
    }

    [Header("Dynamic Tackle Settings")]
    public float tackleCollsionOffset = 4f;
    public Vector2Int blockingGridIndex;

    [SerializeField] private ParticleSystem impactParticle;

    private bool isTackling = false;
    private Sequence tackleSequence;

    public bool CheckForBlockage()
    {
        if (SpawnCars.Instance == null || AStarPathfinder.Instance == null) return false;

        // Verify if a valid A* path to the exit exists, ignoring the car's own occupied body slots.
        List<Vector2Int> path = AStarPathfinder.Instance.FindPath(currentGridIndex, occupiedCells);

        if (path != null && path.Count > 0)
        {
            isBlocked = false;

            // Free all slots occupied by the car body
            if (occupiedCells != null && occupiedCells.Count > 0)
                SpawnCars.Instance.FreeSlots(occupiedCells);
            else
                SpawnCars.Instance.SetSlotOccupation(
                    currentGridIndex.x, currentGridIndex.y, false);

            // Car is leaving the grid
            occupiedCells.Clear();
            occupiedCells.Add(currentGridIndex);
        }
        else
        {
            isBlocked = true;

            // Search ahead to identify the blocking cell for the tackle nudge
            Vector2Int facingDir = SpawnCars.Instance.AngleToGridDir(transform.eulerAngles.y);
            Vector2Int frontCell = currentGridIndex + facingDir;
            
            Vector2Int checkCell = frontCell;
            bool foundOccupied = false;
            for (int i = 0; i < 10; i++)
            {
                if (SpawnCars.Instance.IsSlotBlocked(checkCell.x, checkCell.y))
                {
                    blockingGridIndex = checkCell;
                    foundOccupied = true;
                    break;
                }
                if (SpawnCars.Instance.GetSlotAt(checkCell.x, checkCell.y) == null)
                    break;
                checkCell += facingDir;
            }

            if (!foundOccupied)
            {
                blockingGridIndex = frontCell;
            }
        }

        return isBlocked;
    }

    public void DoTackle()
    {
        if (isTackling)
            return;

        if (tackleSequence != null && tackleSequence.IsActive())
            return;

        isTackling = true;

        Vector3 originalPosition = transform.position;
        Vector3 tacklePosition = originalPosition + (transform.forward * 1.5f);

        if (SpawnCars.Instance != null && isBlocked)
        {
            var blockingSlot = SpawnCars.Instance.GetSlotAt(blockingGridIndex.x, blockingGridIndex.y);
            if (blockingSlot != null)
            {
                Vector3 blockingWorldPos = blockingSlot.WorldPosition;
                Vector3 dir = (blockingWorldPos - originalPosition).normalized;

                float distance = Vector3.Distance(originalPosition, blockingWorldPos);

                if (distance > tackleCollsionOffset)
                {
                    tacklePosition = blockingWorldPos - dir * tackleCollsionOffset;
                }
                else
                {
                    tacklePosition = originalPosition + (dir * (distance * 0.4f));
                }
            }
        }

        tackleSequence = DOTween.Sequence();

        tackleSequence.Append(transform.DOMove(tacklePosition, dashSpeed).SetEase(Ease.OutFlash));

        tackleSequence.AppendCallback(() =>
        {
            if (impactParticle != null)
            {
                impactParticle.Play();
            }
            HapticManager.Instance?.CarHit();
        });

        tackleSequence.Append(transform.DOShakeRotation(0.2f, 10f));
        tackleSequence.Append(transform.DOMove(originalPosition, returnSpeed).SetEase(Ease.InQuad));

        tackleSequence.OnComplete(() =>
        {
            isTackling = false;
        });

        tackleSequence.OnKill(() =>
        {
            isTackling = false;
        });
    }
}
```

### `SpawnCars.cs`
Update this file completely at [SpawnCars.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/SpawnCars.cs):
```csharp
// using System.Collections;
// using System.Collections.Generic;
// using UnityEngine;
// using UnityEngine.Video;

// public class SpawnCars : MonoBehaviour
// {
//     public List<CarMover> carPrefabs;

//     public Vector3 CenterArea;
//     public Vector3 AreaSize;

//     public SpawnPassengers spawnPassengers;

//     // public int howManyCars = 5;

//     public LayerMask carLayer;

//     public List<Transform> carSpawnPoints;

//     public LevelData currentLevelData; 

//     void Awake()
//     {
//         // SpawnCarAsPerLevelNeed();
//     }

//         // public void SpawnCarAsPerLevelNeed()
//         // {
//         //         int carsToSpawn = currentLevelData.carsToSpawn;
//         //         for (int i = 0; i < carsToSpawn; i++)
//         //         {
//         //                 int index = Random.Range(0, carPrefabs.Count);
//         //                 int carSpawnSlots = Random.Range(0, carSpawnPoints.Count);

//         //                 // Define the allowed angles
//         //                 // float[] possibleAngles = { 0f, 30f, 60f, 90f, 180f, 270f, -30f, -60f };
//         //                 // float randomYRotation = possibleAngles[Random.Range(0, possibleAngles.Length)]; // 0 90 180 270
//         //                 // Quaternion spawnRot = Quaternion.Euler(0, randomYRotation, 0);

//         //                 var carMover = Instantiate(carPrefabs[index], carSpawnPoints[carSpawnSlots].position, Quaternion.identity);
//         //                 spawnPassengers.TotalCarsSpawn.Add(carMover);
                                
//         //                 // carSpawnPoints.RemoveAt(carSpawnSlots);
//         //         }
//         // }

//         // public void SpawnCarAsPerLevelNeed()
//         // {
//         // // Safety check: Make sure we have data and spawn points
//         // if (currentLevelData == null || carSpawnPoints.Count == 0) return;

//         // // Create a temporary list so we don't destroy the original list of points
//         // List<Transform> availablePoints = new List<Transform>(carSpawnPoints);

//         // int carsToSpawn = currentLevelData.carsToSpawn;

//         // for (int i = 0; i < carsToSpawn; i++)
//         // {
//         //         // Prevent crash if you try to spawn more cars than you have points
//         //         if (availablePoints.Count == 0) break; 

//         //         int prefabIndex = Random.Range(0, carPrefabs.Count);
//         //         int pointIndex = Random.Range(0, availablePoints.Count);

//         //         var carMover = Instantiate(carPrefabs[prefabIndex], availablePoints[pointIndex].position, Quaternion.identity);
//         //         spawnPassengers.TotalCarsSpawn.Add(carMover);
                        
//         //         // Remove from the TEMP list so two cars don't spawn on top of each other
//         //         availablePoints.RemoveAt(pointIndex);
//         // }
//         // }

//     public void SpawnCarAsPerLevelNeed()
//     {
//         if (currentLevelData == null || carSpawnPoints.Count == 0) return;

//         List<Transform> availablePoints = new List<Transform>(carSpawnPoints);
//         int carsToSpawn = currentLevelData.carsToSpawn;

//         for (int i = 0; i < carsToSpawn; i++)
//         {
//             if (availablePoints.Count == 0) break; 

//             int prefabIndex = Random.Range(0, carPrefabs.Count);
//             // int pointIndex = Random.Range(0, availablePoints.Count); // for random
//             int pointIndex = 0; // for index based

//             // 1. Get reference to the prefab we want
//             CarMover selectedPrefab = carPrefabs[prefabIndex];
            
//             // 2. Try to get from pool, if null then Instantiate
//             CarMover carMover = ObjectPool.Instance.GetCarFromPool(selectedPrefab); 
            
//             if (carMover != null) 
//             {
//                 Debug.Log("car found in pool");
//                 carMover.transform.position = availablePoints[pointIndex].position;
//                 carMover.transform.rotation = Quaternion.identity;
//                 carMover.ResetCapacity();
//                 carMover.ResetEnum();
//                 carMover.gameObject.SetActive(true);
//                 carMover.arrow.enabled = true;
//                 carMover.totalPassengerTxt.enabled= false;
//             } 
//             else 
//             {
//                 Debug.Log("new car added");
//                 carMover = Instantiate(selectedPrefab, availablePoints[pointIndex].position, Quaternion.identity);
//                 // Optional: Add to pool list if your GetFromPool relies on a specific list
//                 // ObjectPool.Instance.pool.Add(carMover.gameObject); 
//             }

//             // 3. Add to the list so level doesn't end early
//             spawnPassengers.TotalCarsSpawn.Add(carMover);
//             availablePoints.RemoveAt(pointIndex);
//         }
//     }


// }

// using System.Collections.Generic;
// using UnityEngine;

// public class SpawnCars : MonoBehaviour 
// {
//     [Header("Prefabs & References")]
//     [SerializeField] public GameObject carSpawnPrefab; 
//     public List<CarMover> carPrefabs;
//     public SpawnPassengers spawnPassengers;
//     public LevelData currentLevelData;

//     [Header("Grid Layout Settings")]
//     public Vector3 CarSpawnGrid; 
    
//     // Based on your car dimensions (X=3, Z=6), 6.5f forms a safe square boundary box
//     // so cars can rotate 0, 90, or -90 degrees without ever clipping.
//     private const float GridSlotSize = 6.5f; 

//     private Vector3 centerArea = new Vector3(-2.4f, 0, -8.5f); 

//     [Header("Runtime Tracked Positions")]
//     public List<GameObject> spawnedPositions = new List<GameObject>();

//     public void SpawnCarAsPerLevelNeed()
//     {
//         SpawnGridAndCars();
//     }

//     public void SpawnGridAndCars()
//     {
//         if (currentLevelData == null || carSpawnPrefab == null || carPrefabs.Count == 0) return;

//         ClearOldPositions();

//         int carsToSpawn = currentLevelData.carsToSpawn;
//         if (carsToSpawn <= 0) return;

//         // 1. Dynamically calculate grid columns based on the total car count.
//         // For 2 cars, it makes a compact layout. For 40 cars, it creates a balanced block.
//         int columns = Mathf.CeilToInt(Mathf.Sqrt(carsToSpawn));
//         int rows = Mathf.CeilToInt((float)carsToSpawn / columns);

//         int spawnedCount = 0;

//         // 2. Build the grid outward from the middle point
//         for (int row = 0; row < rows; row++)
//         {
//             for (int col = 0; col < columns; col++)
//             {
//                 if (spawnedCount >= carsToSpawn) break;

//                 // This math centers the entire block exactly on centerArea.
//                 // It ensures low car counts (like 2) stay clustered tightly in the middle,
//                 // instead of shooting out to the far corners of your Gizmo box.
//                 float offsetX = (col - (columns - 1) / 2f) * GridSlotSize;
//                 float offsetZ = (row - (rows - 1) / 2f) * GridSlotSize;

//                 Vector3 spawnPosition = centerArea + new Vector3(offsetX, 0, offsetZ);

//                 // 3. Select random direction securely: 0, 90, or -90 degrees
//                 float[] possibleAngles = { 0f, 90f};
//                 float randomYRotation = possibleAngles[Random.Range(0, possibleAngles.Length)];
//                 Quaternion randomRotation = Quaternion.Euler(0, randomYRotation, 0);

//                 // 4. Drop the layout position slot indicator
//                 GameObject slotIndicator = Instantiate(carSpawnPrefab, spawnPosition, randomRotation);
//                 spawnedPositions.Add(slotIndicator); 

//                 // 5. Fetch vehicle from pool or construct asset directly
//                 int randomCarIndex = Random.Range(0, carPrefabs.Count);
//                 CarMover chosenPrefab = carPrefabs[randomCarIndex];
//                 CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
                
//                 if (activeCar != null)
//                 {
//                     activeCar.transform.position = spawnPosition;
//                     activeCar.transform.rotation = randomRotation;
//                     activeCar.ResetCapacity();
//                     activeCar.ResetEnum();
//                     activeCar.gameObject.SetActive(true);
//                     activeCar.arrow.enabled = true;
//                     activeCar.totalPassengerTxt.enabled = false;
//                 }
//                 else
//                 {
//                     activeCar = Instantiate(chosenPrefab, spawnPosition, randomRotation);
//                 }

//                 spawnPassengers.TotalCarsSpawn.Add(activeCar);
//                 spawnedCount++;
//             }
//         }
//     }

//     private void ClearOldPositions()
//     {
//         foreach (GameObject pos in spawnedPositions)
//         {
//             if (pos != null) Destroy(pos);
//         }
//         spawnedPositions.Clear();
//     }

//     void OnDrawGizmos()
//     {
//         // Outermost grid size boundaries
//         Gizmos.color = Color.green;
//         Gizmos.DrawWireCube(centerArea, CarSpawnGrid);

//         // Preview the compact, centered grid alignment inside your Scene View
//         if (currentLevelData != null && currentLevelData.carsToSpawn > 0)
//         {
//             Gizmos.color = Color.cyan;
//             int columns = Mathf.CeilToInt(Mathf.Sqrt(currentLevelData.carsToSpawn));
//             int rows = Mathf.CeilToInt((float)currentLevelData.carsToSpawn / columns);

//             for (int r = 0; r < rows; r++)
//             {
//                 for (int c = 0; c < columns; c++)
//                 {
//                     float offsetX = (c - (columns - 1) / 2f) * GridSlotSize;
//                     float offsetZ = (r - (rows - 1) / 2f) * GridSlotSize;
//                     Gizmos.DrawWireCube(centerArea + new Vector3(offsetX, 0, offsetZ), new Vector3(GridSlotSize, 0.2f, GridSlotSize));
//                 }
//             }
//         }
//     }
// }


// using System.Collections.Generic;
// using UnityEngine;

// public class SpawnCars : MonoBehaviour 
// {
//     [Header("Prefabs & References")]
//     [SerializeField] public GameObject carSpawnPrefab; 
//     public List<CarMover> carPrefabs;
//     public SpawnPassengers spawnPassengers;
//     public LevelData currentLevelData;

//     [Header("Grid Layout Settings")]
//     public Vector3 CarSpawnGrid; 
    
//     // Constant outer dimensions for your big map arena grid configuration
//     [SerializeField] private int maxColumns = 5;
//     [SerializeField] private int maxRows = 5;

//     private const float GridSlotSize = 6.5f; 
//     private Vector3 centerArea = new Vector3(-2.4f, 0, -8.5f); 

//     [Header("Runtime Tracked Positions")]
//     public List<GameObject> spawnedPositions = new List<GameObject>();

//     public void SpawnCarAsPerLevelNeed()
//     {
//         SpawnGridAndCars();
//     }

//     public void SpawnGridAndCars()
//     {
//         if (currentLevelData == null || carSpawnPrefab == null || carPrefabs.Count == 0) return;

//         ClearOldPositions();

//         int carsToSpawn = currentLevelData.carsToSpawn;
//         if (carsToSpawn <= 0) return;

//         // 1. Calculate the exact structural center indices of the big predefined grid
//         int centerCol = maxColumns / 2;
//         int centerRow = maxRows / 2;

//         // 2. Map all slots inside the big grid and sort them based on center distance
//         List<Vector2Int> sortedSlots = new List<Vector2Int>();
//         for (int r = 0; r < maxRows; r++)
//         {
//             for (int c = 0; c < maxColumns; c++)
//             {
//                 sortedSlots.Add(new Vector2Int(c, r));
//             }
//         }

//         // Sort: Closest layout grid items to the absolute middle slot bubble up first
//         Vector2Int centerTarget = new Vector2Int(centerCol, centerRow);
//         sortedSlots.Sort((a, b) => 
//             Vector2Int.Distance(a, centerTarget).CompareTo(Vector2Int.Distance(b, centerTarget))
//         );

//         // 3. Process spawning loops strictly using the closest slots up to carsToSpawn count
//         int spawnedCount = 0;
//         for (int i = 0; i < sortedSlots.Count; i++)
//         {
//             if (spawnedCount >= carsToSpawn) break;

//             Vector2Int slot = sortedSlots[i];

//             // Center math ensures slot (centerCol, centerRow) falls perfectly onto centerArea
//             float offsetX = (slot.x - centerCol) * GridSlotSize;
//             float offsetZ = (slot.y - centerRow) * GridSlotSize;
//             Vector3 spawnPosition = centerArea + new Vector3(offsetX, 0, offsetZ);

//             // Select random direction securely: 0 or 90 degrees
//             float[] possibleAngles = { 0f, 90f };
//             float randomYRotation = possibleAngles[Random.Range(0, possibleAngles.Length)];
//             Quaternion randomRotation = Quaternion.Euler(0, randomYRotation, 0);

//             // Drop the layout position slot indicator
//             GameObject slotIndicator = Instantiate(carSpawnPrefab, spawnPosition, randomRotation);
//             spawnedPositions.Add(slotIndicator); 

//             // Fetch vehicle from pool or construct asset directly
//             int randomCarIndex = Random.Range(0, carPrefabs.Count);
//             CarMover chosenPrefab = carPrefabs[randomCarIndex];
//             CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
            
//             if (activeCar != null)
//             {
//                 activeCar.transform.position = spawnPosition;
//                 activeCar.transform.rotation = randomRotation;
//                 activeCar.ResetCapacity();
//                 activeCar.ResetEnum();
//                 activeCar.gameObject.SetActive(true);
//                 activeCar.arrow.enabled = true;
//                 activeCar.totalPassengerTxt.enabled = false;
//             }
//             else
//             {
//                 activeCar = Instantiate(chosenPrefab, spawnPosition, randomRotation);
//             }

//             spawnPassengers.TotalCarsSpawn.Add(activeCar);
//             spawnedCount++;
//         }
//     }

//     private void ClearOldPositions()
//     {
//         foreach (GameObject pos in spawnedPositions)
//         {
//             if (pos != null) Destroy(pos);
//         }
//         spawnedPositions.Clear();
//     }

//     void OnDrawGizmos()
//     {
//         // Outermost boundary visualization
//         Gizmos.color = Color.green;
//         Gizmos.DrawWireCube(centerArea, CarSpawnGrid);

//         // Preview the complete big structural map grid matrix inside your Editor
//         Gizmos.color = Color.cyan;
//         int centerCol = maxColumns / 2;
//         int centerRow = maxRows / 2;

//         for (int r = 0; r < maxRows; r++)
//         {
//             for (int c = 0; c < maxColumns; c++)
//             {
//                 float offsetX = (c - centerCol) * GridSlotSize;
//                 float offsetZ = (r - centerRow) * GridSlotSize;
                
//                 // Keep track of the middle center point visually using Red
//                 if (c == centerCol && r == centerRow) Gizmos.color = Color.red;
//                 else Gizmos.color = Color.cyan;

//                 Gizmos.DrawWireCube(centerArea + new Vector3(offsetX, 0, offsetZ), new Vector3(GridSlotSize - 0.2f, 0.2f, GridSlotSize - 0.2f));
//             }
//         }
//     }
// }


// #region secondCode
// using System.Collections.Generic;
// using UnityEngine;
// using System.Linq; 
// using DG.Tweening;

// public class SpawnCars : MonoBehaviour
// {

//     public static SpawnCars Instance { get; private set; }

//     public class GridSlot
//     {
//         public Vector2Int GridIndex;
//         public Vector3 WorldPosition;
//         public bool IsOccupied;

//         public GridSlot(Vector2Int index, Vector3 pos)
//         {
//             GridIndex = index;
//             WorldPosition = pos;
//             IsOccupied = false;
//         }
//     }

//     [Header("Prefabs & References")]
//     [SerializeField] public GameObject carSpawnPrefab;
//     // public List<CarMover> carPrefabs;

//     [Header("Car Prefabs By Size")]
//     public List<CarMover> smallCarPrefabs;
//     public List<CarMover> mediumCarPrefabs;
//     public List<CarMover> busCarPrefabs;

//     public SpawnPassengers spawnPassengers;
//     public LevelData currentLevelData;

//     [Header("Grid Layout Settings")]
//     // public Vector3 CarSpawnGrid;
//     public int maxColumns = 5;
//     public int maxRows = 5;
//     private const float GridSlotSize = 4.5f;
//     [SerializeField] private Vector3 centerArea = new Vector3(-2.4f, 0, -8.5f);

//     [Header("Runtime Tracked Positions")]
//     public List<GameObject> spawnedPositions = new List<GameObject>();
    
//     private List<GridSlot> gridSlots = new List<GridSlot>();


//     private void Awake()
//     {
//         // Initialize the singleton instance
//         if (Instance == null) Instance = this;
//         else Destroy(gameObject);
//     }

//     // --- GRID ACCESS METHODS FOR EXTERNAL SCRIPTS ---
//     public GridSlot GetSlotAt(int x, int y)
//     {
//         return gridSlots.Find(slot => slot.GridIndex.x == x && slot.GridIndex.y == y);
//     }

//     public bool IsSlotBlocked(int x, int y)
//     {
//         GridSlot slot = GetSlotAt(x, y);
//         if (slot == null) return false; // Grid boundaries block cars
//         return slot.IsOccupied;
//     }

//     public void SetSlotOccupation(int x, int y, bool occupied)
//     {
//         GridSlot slot = GetSlotAt(x, y);
//         if (slot != null)
//         {
//             slot.IsOccupied = occupied;
//         }
//     }

//     public void SpawnCarAsPerLevelNeed()
//     {
//         SpawnGridAndCars();
//     }

//     // public void SpawnGridAndCars()
//     // {
//     //     if (currentLevelData == null || carSpawnPrefab == null || carPrefabs.Count == 0) return;
        
//     //     ClearOldPositions();
        
//     //     int carsToSpawn = currentLevelData.carsToSpawn;
//     //     if (carsToSpawn <= 0) return;

//     //     InitializeGridData();

//     //     Vector2Int centerTarget = new Vector2Int(maxColumns / 2, maxRows / 2);
//     //     var sortedSlots = gridSlots
//     //         .OrderBy(slot => Vector2Int.Distance(slot.GridIndex, centerTarget))
//     //         .ToList();

//     //     int spawnedCount = 0;
//     //     for (int i = 0; i < sortedSlots.Count; i++)
//     //     {
//     //         if (spawnedCount >= carsToSpawn) break;

//     //         GridSlot slot = sortedSlots[i];

//     //         if (slot.IsOccupied) continue; 

//     //         float[] possibleAngles = { 0f, 90f }; //define posiotn in witch Direction car will spawn simply
//     //         float randomYRotation = possibleAngles[Random.Range(0, possibleAngles.Length)];
//     //         Quaternion randomRotation = Quaternion.Euler(0, randomYRotation, 0);

//     //         GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, randomRotation);
//     //         spawnedPositions.Add(slotIndicator);

//     //         int randomCarIndex = Random.Range(0, carPrefabs.Count);
//     //         CarMover chosenPrefab = carPrefabs[randomCarIndex];
            
//     //         CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
            
//     //         if (activeCar != null)
//     //         {
//     //             activeCar.transform.position = slot.WorldPosition;
//     //             activeCar.transform.rotation = randomRotation;
//     //             activeCar.ResetCapacity();
//     //             activeCar.ResetEnum();
//     //             activeCar.gameObject.SetActive(true);
//     //             activeCar.arrow.enabled = true;
//     //             activeCar.totalPassengerTxt.enabled = false;
//     //         }
//     //         else
//     //         {
//     //             activeCar = Instantiate(chosenPrefab, slot.WorldPosition, randomRotation);
//     //         }

//     //         // ASSIGN INITIAL GRID INDEX TO THE CAR
//     //         Carout caroutComp = activeCar.GetComponent<Carout>();
//     //         if (caroutComp != null)
//     //         {
//     //             caroutComp.currentGridIndex = slot.GridIndex;
//     //         }

//     //         spawnPassengers.TotalCarsSpawn.Add(activeCar);
            
//     //         slot.IsOccupied = true; 
//     //         spawnedCount++;
//     //     }
//     // }
//     public void SpawnGridAndCars()
//     {
//         if (currentLevelData == null || carSpawnPrefab == null ) return;
        
//         ClearOldPositions();
        
//         // If there are specific visual car spawns configured in the LevelData, load them!
//         if (currentLevelData.carSpawns != null && currentLevelData.carSpawns.Count > 0)
//         {
//             maxColumns = currentLevelData.gridColumns;
//             maxRows = currentLevelData.gridRows;

//             InitializeGridData();

//             foreach (var spawn in currentLevelData.carSpawns)
//             {
//                 GridSlot slot = GetSlotAt(spawn.gridPosition.x, spawn.gridPosition.y);
//                 if (slot == null) continue;

//                 Quaternion carRotation = Quaternion.Euler(0, spawn.rotationY, 0);

//                 Vector2Int facing=AngleToGridDir(spawn.rotationY);

//                 List<Vector2Int> occupiedCells=
//                 GetCellsForCar(
//                 spawn.gridPosition,
//                 facing,
//                 spawn.carSize);

//                 bool canSpawn=true;

//                 foreach(var cell in occupiedCells)
//                 {
//                     GridSlot gs=GetSlotAt(cell.x,cell.y);

//                     if(gs==null || gs.IsOccupied)
//                     {
//                         canSpawn=false;
//                         break;
//                     }
//                 }

//                 if(!canSpawn)
//                     continue;

//                 // Spawn the rotation indicator (arrow/direction slot indicator)
//                 GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, carRotation);
//                 spawnedPositions.Add(slotIndicator);

//                 // As per user request: spawn cars randomly from the available prefabs list at runtime
//                 // int randomCarIndex = Random.Range(0, carPrefabs.Count);
//                 // CarMover chosenPrefab = carPrefabs[randomCarIndex];

//                 CarMover chosenPrefab=GetPrefabForSize(spawn.carSize);

//                 if(chosenPrefab==null)
//                     continue;

//                 CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
                
//                 if (activeCar != null)
//                 {
//                     activeCar.transform.position = slot.WorldPosition;
//                     activeCar.transform.rotation = carRotation;
//                     activeCar.ResetCapacity();
//                     activeCar.ResetEnum();
//                     activeCar.gameObject.SetActive(true);
//                     activeCar.arrow.enabled = true;
//                     activeCar.totalPassengerTxt.enabled = false;
//                 }
//                 else
//                 {
//                     activeCar = Instantiate(chosenPrefab, slot.WorldPosition, carRotation);
//                     activeCar.originalPrefab = chosenPrefab;
//                 }

//                 // ASSIGN INITIAL GRID INDEX TO THE CAR
//                 Carout caroutComp = activeCar.GetComponent<Carout>();
//                 if (caroutComp != null)
//                 {
//                     caroutComp.currentGridIndex = slot.GridIndex;
//                     caroutComp.SetOccupiedCells(occupiedCells);
//                 }
                
//                 foreach(var cell in occupiedCells)
//                     SetSlotOccupation(cell.x,cell.y,true);

//                 AnimateCarSpawn(activeCar, spawnPassengers.TotalCarsSpawn.Count);//------------
//                 spawnPassengers.TotalCarsSpawn.Add(activeCar);
//                 // slot.IsOccupied = true;
//             }
//         }
//         else
//         {
//             // FALLBACK / LEGACY SYSTEM: Spawn randomly (Levels 1-8 and infinite random generation)
//             maxColumns = 5;
//             maxRows = 5;
//             InitializeGridData();

//             int carsToSpawn = currentLevelData.carsToSpawn;
//             if (carsToSpawn <= 0) return;

//             Vector2Int centerTarget = new Vector2Int(maxColumns / 2, maxRows / 2);
//             var sortedSlots = gridSlots
//                 .OrderBy(slot => Vector2Int.Distance(slot.GridIndex, centerTarget))
//                 .ToList();

//             int spawnedCount = 0;
//             for (int i = 0; i < sortedSlots.Count; i++)
//             {
//                 if (spawnedCount >= carsToSpawn) break;

//                 GridSlot slot = sortedSlots[i];

//                 if (slot.IsOccupied) continue; 

//                 float[] possibleAngles = { 0f, 45f, 90f ,135f, 180f, 225f, 270f, 315f}; 
//                 float randomYRotation = possibleAngles[Random.Range(0, possibleAngles.Length)];
//                 Quaternion randomRotation = Quaternion.Euler(0, randomYRotation, 0);

//                 GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, randomRotation);
//                 spawnedPositions.Add(slotIndicator);

//                 // int randomCarIndex = Random.Range(0, carPrefabs.Count);
//                 // CarMover chosenPrefab = carPrefabs[randomCarIndex];

//                 CarMover chosenPrefab=GetPrefabForSize(CarSize.Small);

//                 if(chosenPrefab==null)
//                     return;
                
//                 CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
                
//                 if (activeCar != null)
//                 {
//                     activeCar.transform.position = slot.WorldPosition;
//                     activeCar.transform.rotation = randomRotation;
//                     activeCar.ResetCapacity();
//                     activeCar.ResetEnum();
//                     activeCar.gameObject.SetActive(true);
//                     activeCar.arrow.enabled = true;
//                     activeCar.totalPassengerTxt.enabled = false;
//                 }
//                 else
//                 {
//                     activeCar = Instantiate(chosenPrefab, slot.WorldPosition, randomRotation);
//                     activeCar.originalPrefab = chosenPrefab;
//                 }

//                 Carout caroutComp = activeCar.GetComponent<Carout>();
//                 if (caroutComp != null)
//                 {
//                     caroutComp.currentGridIndex = slot.GridIndex;
//                 }

//                 AnimateCarSpawn(activeCar, spawnedCount);
//                 spawnPassengers.TotalCarsSpawn.Add(activeCar);
//                 slot.IsOccupied = true; 
//                 spawnedCount++;
//             }
//         }
//     }

//     public bool CheckBlockageForCar(Vector2Int currentGridIndex, Vector3 eulerAngles, out Vector2Int targetGridIndex)
//     {
//         // Vector2Int forwardDirection = Vector2Int.zero;
        
//         // // Round to nearest 90 degrees to avoid rotation precision floating-point bugs
//         // float currentYRotation = Mathf.Repeat(Mathf.Round(eulerAngles.y / 90f) * 90f, 360f);

//         // if (Mathf.Approximately(currentYRotation, 0f))       forwardDirection = new Vector2Int(0, 1);   // North
//         // else if (Mathf.Approximately(currentYRotation, 90f))  forwardDirection = new Vector2Int(1, 0);   // East
//         // else if (Mathf.Approximately(currentYRotation, 180f)) forwardDirection = new Vector2Int(0, -1);  // South
//         // else if (Mathf.Approximately(currentYRotation, 270f)) forwardDirection = new Vector2Int(-1, 0);  // West

//         // targetGridIndex = currentGridIndex + forwardDirection;
        
//          // 1. Calculate 8-way direction
//         float angle = Mathf.Repeat(Mathf.Round(eulerAngles.y / 45f) * 45f, 360f);
//         int roundedAngle = Mathf.RoundToInt(angle);
        
//         Vector2Int stepDir = Vector2Int.zero;
//         switch (roundedAngle)
//         {
//             case 0:   stepDir = new Vector2Int(0, 1);   break; // Up
//             case 45:  stepDir = new Vector2Int(1, 1);   break; // Up-Right
//             case 90:  stepDir = new Vector2Int(1, 0);   break; // Right
//             case 135: stepDir = new Vector2Int(1, -1);  break; // Down-Right
//             case 180: stepDir = new Vector2Int(0, -1);  break; // Down
//             case 225: stepDir = new Vector2Int(-1, -1); break; // Down-Left
//             case 270: stepDir = new Vector2Int(-1, 0);  break; // Left
//             case 315: stepDir = new Vector2Int(-1, 1);  break; // Up-Left
//         }

//         // 2. Loop to check the entire path in that direction
//         Vector2Int nextCheck = currentGridIndex + stepDir;
//         targetGridIndex = currentGridIndex; // Default fallback

//         // Loop continues until it hits an obstacle or the edge of the grid
//         while (true)
//         {
//             // Check if the current slot in the path is blocked using your method
//             if (IsSlotBlocked(nextCheck.x, nextCheck.y))
//             {
//                 targetGridIndex = nextCheck;
//                 return IsSlotBlocked(targetGridIndex.x, targetGridIndex.y);
//                 // return true; // Path is blocked
//             }

//             // If GetSlotAt returns null (edge of grid), stop and return clear
//             if (GetSlotAt(nextCheck.x, nextCheck.y) == null)
//             {
//                 targetGridIndex = nextCheck - stepDir; // Last valid spot
//                 return false; // Path is clear to the exit
//             }

//             nextCheck += stepDir;
//         }

//     }
//     //====================================================================

//     public Vector2Int AngleToGridDir(float rotationY)
//     {
//         float angle = Mathf.Repeat(Mathf.Round(rotationY / 45f) * 45f, 360f);

//         switch (Mathf.RoundToInt(angle))
//         {
//             case 0: return new Vector2Int(0,1);
//             case 45: return new Vector2Int(1,1);
//             case 90: return new Vector2Int(1,0);
//             case 135: return new Vector2Int(1,-1);
//             case 180: return new Vector2Int(0,-1);
//             case 225: return new Vector2Int(-1,-1);
//             case 270: return new Vector2Int(-1,0);
//             case 315: return new Vector2Int(-1,1);
//         }

//         return Vector2Int.zero;
//     }

//     public List<Vector2Int> GetCellsForCar(
//     Vector2Int anchor,
//     Vector2Int facing,
//     CarSize size)
//     {
//         List<Vector2Int> cells=new List<Vector2Int>();

//         cells.Add(anchor);

//         Vector2Int behind=-facing;

//         for(int i=1;i<(int)size;i++)
//             cells.Add(anchor+behind*i);

//         return cells;
//     }

//     private CarMover GetPrefabForSize(CarSize size)
//     {
//         List<CarMover> pool;

//         switch(size)
//         {
//             case CarSize.Medium:
//                 pool=mediumCarPrefabs;
//                 break;

//             case CarSize.Bus:
//                 pool=busCarPrefabs;
//                 break;

//             default:
//                 pool=smallCarPrefabs;
//                 break;
//         }

//         if(pool==null || pool.Count==0)
//             pool=smallCarPrefabs;

//         if(pool==null || pool.Count==0)
//             return null;

//         return pool[Random.Range(0,pool.Count)];
//     }

//     public void FreeSlots(List<Vector2Int> cells)
//     {
//         if(cells==null)
//             return;

//         foreach(var cell in cells)
//             SetSlotOccupation(cell.x,cell.y,false);
//     }

//     //====================================================================

//     private void InitializeGridData()
//     {
//         gridSlots.Clear();
//         int centerCol = maxColumns / 2;
//         int centerRow = maxRows / 2;

//         for (int r = 0; r < maxRows; r++)
//         {
//             for (int c = 0; c < maxColumns; c++)
//             {
//                 float offsetX = (c - centerCol) * GridSlotSize;
//                 float offsetZ = (r - centerRow) * GridSlotSize;
//                 Vector3 spawnPosition = centerArea + new Vector3(offsetX, 0, offsetZ);
                
//                 gridSlots.Add(new GridSlot(new Vector2Int(c, r), spawnPosition));
//             }
//         }
//     }

//     private void ClearOldPositions()
//     {
//         foreach (GameObject pos in spawnedPositions)
//         {
//             if (pos != null) Destroy(pos);
//         }
//         spawnedPositions.Clear();
//         gridSlots.Clear(); 
//     }

//     void OnDrawGizmos()
//     {
//         // Gizmos.color = Color.green;
//         // Gizmos.DrawWireCube(centerArea, CarSpawnGrid);
//         int centerCol = maxColumns / 2;
//         int centerRow = maxRows / 2;
//         for (int r = 0; r < maxRows; r++)
//         {
//             for (int c = 0; c < maxColumns; c++)
//             {
//                 float offsetX = (c - centerCol) * GridSlotSize;
//                 float offsetZ = (r - centerRow) * GridSlotSize;
//                 if (c == centerCol && r == centerRow) Gizmos.color = Color.red;
//                 else Gizmos.color = Color.cyan;
//                 Gizmos.DrawWireCube(centerArea + new Vector3(offsetX, 0, offsetZ), new Vector3(GridSlotSize - 0.2f, 0.2f, GridSlotSize - 0.2f));
//             }
//         }
//     }

//     private void AnimateCarSpawn(CarMover car, int index)
//     {
//         if (car == null) return;
        
//         Transform t = car.transform;

//         // Vector3 finalScale = t.localScale;
//         // Vector3 finalScale = Vector3.one * 0.7f;
//         Vector3 finalScale = car.GetOriginalScale();

//         t.localScale = Vector3.zero;

//         Sequence seq = DOTween.Sequence();

//         seq.AppendInterval(index * 0.05f);

//         seq.Append(
//             t.DOScale(finalScale * 1.15f, 0.22f)
//             .SetEase(Ease.OutBack)
//         );

//         seq.Append(
//             t.DOScale(finalScale, 0.12f)
//             .SetEase(Ease.OutQuad)
//         );
//     }
// }
// #endregion

using System.Collections.Generic;
using UnityEngine;
using System.Linq;
using DG.Tweening;

public class SpawnCars : MonoBehaviour
{
    public static SpawnCars Instance { get; private set; }

    public class GridSlot
    {
        public Vector2Int GridIndex;
        public Vector3 WorldPosition;
        public bool IsOccupied;
        public bool IsExit;

        public GridSlot(Vector2Int index, Vector3 pos)
        {
            GridIndex = index;
            WorldPosition = pos;
            IsOccupied = false;
            IsExit = false;
        }
    }

    [Header("Prefabs & References")]
    [SerializeField] public GameObject carSpawnPrefab;

    [Header("Car Prefabs (Small Only)")]
    public List<CarMover> smallCarPrefabs;

    public SpawnPassengers spawnPassengers;
    public LevelData currentLevelData;

    [Header("Dot Grid Layout Settings")]
    public int maxColumns = 5;
    public int maxRows = 5;

    [Tooltip("Spacing between dot centers. Lower = tighter packing. ~2.0 for nearly touching cars.")]
    [SerializeField] private float DotSize = 2.0f;

    [SerializeField] private Vector3 centerArea = new Vector3(-2.4f, 0, -8.5f);

    [Header("Runtime Tracked Positions")]
    public List<GameObject> spawnedPositions = new List<GameObject>();
    
    private List<GridSlot> gridSlots = new List<GridSlot>();

    private void Awake()
    {
        if (Instance == null) Instance = this;
        else Destroy(gameObject);
    }

    public GridSlot GetSlotAt(int x, int y)
    {
        return gridSlots.Find(slot => slot.GridIndex.x == x && slot.GridIndex.y == y);
    }

    public bool IsSlotBlocked(int x, int y)
    {
        GridSlot slot = GetSlotAt(x, y);
        if (slot == null) return false;
        return slot.IsOccupied;
    }

    public void SetSlotOccupation(int x, int y, bool occupied)
    {
        GridSlot slot = GetSlotAt(x, y);
        if (slot != null)
        {
            slot.IsOccupied = occupied;
        }
    }

    public void FreeSlots(List<Vector2Int> cells)
    {
        if (cells == null) return;
        foreach (var cell in cells)
            SetSlotOccupation(cell.x, cell.y, false);
    }

    public void SpawnCarAsPerLevelNeed()
    {
        SpawnGridAndCars();
    }

    public void SpawnGridAndCars()
    {
        if (currentLevelData == null || carSpawnPrefab == null) return;
        
        ClearOldPositions();
        
        if (currentLevelData.carSpawns != null && currentLevelData.carSpawns.Count > 0)
        {
            maxColumns = currentLevelData.gridColumns;
            maxRows = currentLevelData.gridRows;
            InitializeGridData();

            foreach (var spawn in currentLevelData.carSpawns)
            {
                GridSlot slot = GetSlotAt(spawn.gridPosition.x, spawn.gridPosition.y);
                if (slot == null || slot.IsOccupied) continue;

                Quaternion carRotation = Quaternion.Euler(0, spawn.rotationY, 0);

                GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, carRotation);
                spawnedPositions.Add(slotIndicator);

                CarMover chosenPrefab = GetRandomSmallPrefab();
                if (chosenPrefab == null) continue;

                CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
                
                if (activeCar != null)
                {
                    activeCar.transform.position = slot.WorldPosition;
                    activeCar.transform.rotation = carRotation;
                    activeCar.ResetCapacity();
                    activeCar.ResetEnum();
                    activeCar.gameObject.SetActive(true);
                    activeCar.arrow.enabled = true;
                    activeCar.totalPassengerTxt.enabled = false;
                }
                else
                {
                    activeCar = Instantiate(chosenPrefab, slot.WorldPosition, carRotation);
                    activeCar.originalPrefab = chosenPrefab;
                }

                Carout caroutComp = activeCar.GetComponent<Carout>();
                if (caroutComp != null)
                {
                    caroutComp.currentGridIndex = slot.GridIndex;
                    caroutComp.SetOccupiedCells(new List<Vector2Int> { slot.GridIndex });
                }
                
                slot.IsOccupied = true;

                AnimateCarSpawn(activeCar, spawnPassengers.TotalCarsSpawn.Count);
                spawnPassengers.TotalCarsSpawn.Add(activeCar);
            }
        }
        else
        {
            int carsToSpawn = currentLevelData.carsToSpawn;
            if (carsToSpawn <= 0) return;

            int innerCols = Mathf.CeilToInt(Mathf.Sqrt(carsToSpawn));
            int innerRows = Mathf.CeilToInt((float)carsToSpawn / innerCols);
            maxColumns = innerCols + 2;
            maxRows = innerRows + 2;

            InitializeGridData();

            List<GridSlot> innerSlots = new List<GridSlot>();
            for (int r = 1; r < maxRows - 1; r++)
            {
                for (int c = 1; c < maxColumns - 1; c++)
                {
                    GridSlot s = GetSlotAt(c, r);
                    if (s != null) innerSlots.Add(s);
                }
            }

            for (int i = innerSlots.Count - 1; i > 0; i--)
            {
                int j = Random.Range(0, i + 1);
                var temp = innerSlots[i];
                innerSlots[i] = innerSlots[j];
                innerSlots[j] = temp;
            }

            float sumX = 0, sumY = 0;
            int usableCount = Mathf.Min(carsToSpawn, innerSlots.Count);
            for (int i = 0; i < usableCount; i++)
            {
                sumX += innerSlots[i].GridIndex.x;
                sumY += innerSlots[i].GridIndex.y;
            }
            float cx = sumX / usableCount;
            float cy = sumY / usableCount;

            int spawnedCount = 0;
            for (int i = 0; i < innerSlots.Count && spawnedCount < carsToSpawn; i++)
            {
                GridSlot slot = innerSlots[i];
                if (slot.IsOccupied) continue;

                float angle = CalculateOutwardAngle(
                    slot.GridIndex.x, slot.GridIndex.y, cx, cy);
                Quaternion rotation = Quaternion.Euler(0, angle, 0);

                GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, rotation);
                spawnedPositions.Add(slotIndicator);

                CarMover chosenPrefab = GetRandomSmallPrefab();
                if (chosenPrefab == null) return;
                
                CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);
                
                if (activeCar != null)
                {
                    activeCar.transform.position = slot.WorldPosition;
                    activeCar.transform.rotation = rotation;
                    activeCar.ResetCapacity();
                    activeCar.ResetEnum();
                    activeCar.gameObject.SetActive(true);
                    activeCar.arrow.enabled = true;
                    activeCar.totalPassengerTxt.enabled = false;
                }
                else
                {
                    activeCar = Instantiate(chosenPrefab, slot.WorldPosition, rotation);
                    activeCar.originalPrefab = chosenPrefab;
                }

                Carout caroutComp = activeCar.GetComponent<Carout>();
                if (caroutComp != null)
                {
                    caroutComp.currentGridIndex = slot.GridIndex;
                    caroutComp.SetOccupiedCells(new List<Vector2Int> { slot.GridIndex });
                }

                AnimateCarSpawn(activeCar, spawnedCount);
                spawnPassengers.TotalCarsSpawn.Add(activeCar);
                slot.IsOccupied = true;
                spawnedCount++;
            }
        }
    }

    public bool CheckBlockageForCar(Vector2Int currentGridIndex, Vector3 eulerAngles, out Vector2Int targetGridIndex)
    {
        float angle = Mathf.Repeat(Mathf.Round(eulerAngles.y / 45f) * 45f, 360f);
        int roundedAngle = Mathf.RoundToInt(angle);
        
        Vector2Int stepDir = Vector2Int.zero;
        switch (roundedAngle)
        {
            case 0:   stepDir = new Vector2Int(0, 1);   break;
            case 45:  stepDir = new Vector2Int(1, 1);   break;
            case 90:  stepDir = new Vector2Int(1, 0);   break;
            case 135: stepDir = new Vector2Int(1, -1);  break;
            case 180: stepDir = new Vector2Int(0, -1);  break;
            case 225: stepDir = new Vector2Int(-1, -1); break;
            case 270: stepDir = new Vector2Int(-1, 0);  break;
            case 315: stepDir = new Vector2Int(-1, 1);  break;
        }

        Vector2Int nextCheck = currentGridIndex + stepDir;
        targetGridIndex = currentGridIndex;

        while (true)
        {
            if (IsSlotBlocked(nextCheck.x, nextCheck.y))
            {
                targetGridIndex = nextCheck;
                return true;
            }

            if (GetSlotAt(nextCheck.x, nextCheck.y) == null)
            {
                targetGridIndex = nextCheck - stepDir;
                return false;
            }

            nextCheck += stepDir;
        }
    }

    public Vector2Int AngleToGridDir(float rotationY)
    {
        float angle = Mathf.Repeat(Mathf.Round(rotationY / 45f) * 45f, 360f);

        switch (Mathf.RoundToInt(angle))
        {
            case 0: return new Vector2Int(0, 1);
            case 45: return new Vector2Int(1, 1);
            case 90: return new Vector2Int(1, 0);
            case 135: return new Vector2Int(1, -1);
            case 180: return new Vector2Int(0, -1);
            case 225: return new Vector2Int(-1, -1);
            case 270: return new Vector2Int(-1, 0);
            case 315: return new Vector2Int(-1, 1);
        }

        return Vector2Int.zero;
    }

    private CarMover GetRandomSmallPrefab()
    {
        if (smallCarPrefabs == null || smallCarPrefabs.Count == 0)
            return null;
        return smallCarPrefabs[Random.Range(0, smallCarPrefabs.Count)];
    }

    private float CalculateOutwardAngle(int x, int y, float cx, float cy)
    {
        float dx = x - cx;
        float dy = y - cy;

        if (Mathf.Abs(dx) < 0.1f && Mathf.Abs(dy) < 0.1f)
        {
            float[] angles = { 0f, 45f, 90f, 135f, 180f, 225f, 270f, 315f };
            return angles[Random.Range(0, angles.Length)];
        }

        float angle = Mathf.Atan2(dx, dy) * Mathf.Rad2Deg;
        if (angle < 0) angle += 360f;
        return Mathf.Round(angle / 45f) * 45f;
    }

    private void InitializeGridData()
    {
        gridSlots.Clear();
        int centerCol = maxColumns / 2;
        int centerRow = maxRows / 2;

        for (int r = 0; r < maxRows; r++)
        {
            for (int c = 0; c < maxColumns; c++)
            {
                float offsetX = (c - centerCol) * DotSize;
                float offsetZ = (r - centerRow) * DotSize;
                Vector3 pos = centerArea + new Vector3(offsetX, 0, offsetZ);
                var slot = new GridSlot(new Vector2Int(c, r), pos);

                // Set default Exit cell at top-middle of the grid
                if (c == centerCol && r == maxRows - 1)
                {
                    slot.IsExit = true;
                }

                gridSlots.Add(slot);
            }
        }
    }

    private void ClearOldPositions()
    {
        foreach (GameObject pos in spawnedPositions)
        {
            if (pos != null) Destroy(pos);
        }
        spawnedPositions.Clear();
        gridSlots.Clear();
    }

    void OnDrawGizmos()
    {
        int centerCol = maxColumns / 2;
        int centerRow = maxRows / 2;
        for (int r = 0; r < maxRows; r++)
        {
            for (int c = 0; c < maxColumns; c++)
            {
                float offsetX = (c - centerCol) * DotSize;
                float offsetZ = (r - centerRow) * DotSize;
                
                if (c == centerCol && r == centerRow) 
                    Gizmos.color = Color.red;
                else 
                    Gizmos.color = Color.cyan;
                
                Gizmos.DrawWireSphere(
                    centerArea + new Vector3(offsetX, 0, offsetZ), 
                    DotSize * 0.15f);
            }
        }
    }

    private void AnimateCarSpawn(CarMover car, int index)
    {
        if (car == null) return;
        
        Transform t = car.transform;
        Vector3 finalScale = car.GetOriginalScale();

        t.localScale = Vector3.zero;

        Sequence seq = DOTween.Sequence();
        seq.AppendInterval(index * 0.05f);
        seq.Append(
            t.DOScale(finalScale * 1.15f, 0.22f)
            .SetEase(Ease.OutBack)
        );
        seq.Append(
            t.DOScale(finalScale, 0.12f)
            .SetEase(Ease.OutQuad)
        );
    }
}
```

---

## 5. Inspector Changes

To enable the new A* pathfinding system and verify size compatibility:
1. **`AStarPathfinder` Script**:
   - Must be attached to a new or existing GameObject in the scene (e.g., attach it directly to the same GameObject holding `SpawnCars` to keep components organized).
   - No serialized fields are required (uses `SpawnCars.Instance` dynamically).
2. **`CarMover` Script**:
   - The serialized field `carSize` (of enum type `CarSize`) is added.
   - Configure this field for each car prefab (e.g., `Small` for standard cars, `Medium` for sedans, `Large` for SUVs, `Bus` for long buses).

---

## 6. Scene / Prefab Changes

- Create a new empty GameObject or select the `SpawnCars` GameObject and attach the `AStarPathfinder` component.
- Ensure that the car prefabs in the project have their `carSize` field set correctly inside their Inspector window.
- Ensure that the grid layout size matches the boundaries configured in `SpawnCars` and `LevelData`.

---

## 7. Migration Steps

1. **Step 1: Create new scripts**: Create the files `CarSize.cs`, `PathSmoother.cs`, and `AStarPathfinder.cs` in the `Assets/Script` folder.
2. **Step 2: Update existing scripts**: Overwrite the code of `CarMover.cs`, `Carout.cs`, and `SpawnCars.cs` with the updated complete files.
3. **Step 3: Setup Scene Component**: Drag `AStarPathfinder` to the `SpawnCars` GameObject in your active game scene.
4. **Step 4: Prefab Configuration**: Set `carSize` on your car prefabs to matching sizes (Small, Medium, Large, Bus) in the Unity Inspector.
5. **Step 5: Run and verify**: Start the game and click a car. Verify that it follows a smooth curved spline towards the exit and parks correctly without lag.

---

## 8. Testing Checklist

Use this checklist during manual verification in the Unity Editor:
- [ ] **Straight movement**: A* plans straight routes with a cost of 10 per slot.
- [ ] **Diagonal movement**: A* plans diagonal routes with a cost of 14 per slot.
- [ ] **Corner cutting prevention**: Diagonal movement is blocked if the adjacent horizontal or vertical cell is blocked.
- [ ] **Shortest path selection**: Pathfinder chooses the optimal path to the exit.
- [ ] **Dynamic occupancy**: Leaving cars free up grid occupancy immediately for subsequent cars.
- [ ] **Curved path movement**: Catmull-Rom spline removes blocky turns.
- [ ] **Smooth rotation**: Slerp rotation matches path tangent continuously.
- [ ] **Size compatibility**: Ensure Small, Medium, Large, and Bus vehicles parse correctly.
- [ ] **Exit & Parking behavior**: Vehicles complete the path and park cleanly in matching slots.
- [ ] **Performance**: Allocation-free search avoids garbage collection frame drops.
