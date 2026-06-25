# 🚗 Final Car Movement Fix

You only need to replace **ONE** file. Everything else in your project (`SpawnCars`, `Carout`, `ParkingSlotManager`, passengers, pooling, etc.) is perfectly fine and untouched.

## 📦 Instructions
1. Open your Unity project.
2. Open `Assets/Script/CarMover.cs`.
3. Delete everything inside it.
4. Copy and paste the code below into it.
5. Save the file.

---

### [CarMover.cs](file:///c:/Users/ABHAYprajapati/Downloads/Carout/Car-OUT-jam-puzzle-Game-Grid/Car-OUT-jam-puzzle-Game-Grid/Assets/Script/CarMover.cs)

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

    [Header("Movement")]
    public float speed = 35f;
    [SerializeField] private float turnSpeed = 9f; // Steering smoothness

    private bool isMoving = false;

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

    // Helper class for BFS path construction
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

    void Awake()
    {
        carout = GetComponent<Carout>(); // Cache ONCE
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
            isMoving = false;
            yield break;
        }

        Vector2Int currentGrid = carout.currentGridIndex;

        // 1. Smoothly center onto starting grid slot before initiating path
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

        // 2. Detect starting direction
        Vector2Int dir = GetDirectionFromAngle(transform.eulerAngles.y);

        // 3. Build path using BFS smart search
        List<Vector3> waypoints = GeneratePath(currentGrid, dir);

        // Append final parking slot
        waypoints.Add(targetPosition);

        // 4. Follow waypoints continuously and smoothly
        int wpIndex = 0;

        while (wpIndex < waypoints.Count)
        {
            Vector3 wp = waypoints[wpIndex];
            Vector3 toTarget = wp - transform.position;
            toTarget.y = 0;

            float distance = toTarget.magnitude;
            float step = speed * Time.deltaTime;

            if (distance <= step || distance < 0.01f)
            {
                transform.position = wp;
                wpIndex++;
                continue;
            }

            Vector3 moveDir = toTarget.normalized;

            // Smooth rotation toward waypoint
            Quaternion targetRot = Quaternion.LookRotation(moveDir);
            transform.rotation = Quaternion.Slerp(
                transform.rotation,
                targetRot,
                turnSpeed * Time.deltaTime);

            // Scale movement based on alignment to prevent overshooting/circling
            float alignment = Vector3.Dot(transform.forward, moveDir);
            float speedFactor = Mathf.Max(0.15f, alignment); // slow down to turn if misaligned

            transform.position = Vector3.MoveTowards(
                transform.position,
                wp,
                speed * speedFactor * Time.deltaTime);

            yield return null;
        }

        // 5. Final parking slot snap
        transform.position = targetPosition;
        transform.rotation = Quaternion.Euler(0, 0, 0);

        isMoving = false;
        isParked = true;

        HapticManager.Instance?.CarFull();
    }

    private List<Vector3> GeneratePath(Vector2Int startGrid, Vector2Int initialFacingDir)
    {
        List<Vector3> worldPath = new List<Vector3>();

        int maxCols = SpawnCars.Instance.maxColumns;
        int maxRows = SpawnCars.Instance.maxRows;

        // BFS to find the shortest path to ANY border cell
        Queue<PathNode> openSet = new Queue<PathNode>();
        HashSet<Vector2Int> visited = new HashSet<Vector2Int>();

        PathNode startNode = new PathNode(startGrid, null);
        openSet.Enqueue(startNode);
        visited.Add(startGrid);

        Vector2Int[] cardinalDirs = new Vector2Int[]
        {
            new Vector2Int(0, 1),   // North
            new Vector2Int(0, -1),  // South
            new Vector2Int(1, 0),   // East
            new Vector2Int(-1, 0),  // West
        };

        PathNode borderNode = null;

        while (openSet.Count > 0)
        {
            PathNode current = openSet.Dequeue();
            Vector2Int pos = current.Pos;

            // Reached grid border?
            bool isBorder = pos.x == 0 || pos.x == maxCols - 1 || pos.y == 0 || pos.y == maxRows - 1;

            if (isBorder && pos != startGrid)
            {
                borderNode = current;
                break;
            }

            // Insert neighbors, prioritizing the preferred facing direction to minimize turns
            List<Vector2Int> neighbors = new List<Vector2Int>();
            Vector2Int preferred = pos + initialFacingDir;
            if (initialFacingDir != Vector2Int.zero)
            {
                neighbors.Add(preferred);
            }

            foreach (Vector2Int d in cardinalDirs)
            {
                Vector2Int n = pos + d;
                if (n != preferred)
                {
                    neighbors.Add(n);
                }
            }

            foreach (Vector2Int n in neighbors)
            {
                if (visited.Contains(n)) continue;

                // Check bounds
                if (n.x < 0 || n.x >= maxCols || n.y < 0 || n.y >= maxRows) continue;

                var slot = SpawnCars.Instance.GetSlotAt(n.x, n.y);
                if (slot == null) continue;

                // Skip occupied cells (unless it is startGrid itself)
                if (n != startGrid && SpawnCars.Instance.IsSlotBlocked(n.x, n.y)) continue;

                visited.Add(n);
                openSet.Enqueue(new PathNode(n, current));
            }
        }

        // Reconstruct path
        if (borderNode == null)
        {
            // Fallback: Drive straight in facing direction
            borderNode = FallbackStraightPath(startGrid, initialFacingDir, maxCols, maxRows);
        }

        List<Vector2Int> gridPath = new List<Vector2Int>();
        PathNode node = borderNode;
        while (node != null && node.Pos != startGrid)
        {
            gridPath.Add(node.Pos);
            node = node.Parent;
        }
        gridPath.Reverse();

        foreach (Vector2Int gp in gridPath)
        {
            var slot = SpawnCars.Instance.GetSlotAt(gp.x, gp.y);
            if (slot != null) worldPath.Add(slot.WorldPosition);
        }

        Vector2Int borderPos = (borderNode != null) ? borderNode.Pos : startGrid;
        AppendBorderRoad(worldPath, borderPos, maxCols, maxRows);

        return worldPath;
    }

    private PathNode FallbackStraightPath(Vector2Int start, Vector2Int dir, int maxCols, int maxRows)
    {
        PathNode prev = new PathNode(start, null);
        Vector2Int cur = start;

        for (int i = 0; i < Mathf.Max(maxCols, maxRows) + 1; i++)
        {
            Vector2Int next = cur + dir;
            bool outside = next.x < 0 || next.x >= maxCols || next.y < 0 || next.y >= maxRows;
            if (outside) break;

            cur = next;
            PathNode n = new PathNode(cur, prev);
            prev = n;
        }

        return prev;
    }

    private void AppendBorderRoad(List<Vector3> path, Vector2Int borderPos, int maxCols, int maxRows)
    {
        int topRow = maxRows - 1;
        int centerX = maxCols / 2;

        if (borderPos.x == 0) // LEFT
        {
            for (int y = borderPos.y + 1; y < maxRows; y++)
                AddSlot(path, 0, y);

            for (int x = 1; x <= centerX; x++)
                AddSlot(path, x, topRow);
        }
        else if (borderPos.x == maxCols - 1) // RIGHT
        {
            for (int y = borderPos.y + 1; y < maxRows; y++)
                AddSlot(path, maxCols - 1, y);

            for (int x = maxCols - 2; x >= centerX; x--)
                AddSlot(path, x, topRow);
        }
        else if (borderPos.y == 0) // BOTTOM
        {
            for (int x = borderPos.x + 1; x < maxCols; x++)
                AddSlot(path, x, 0);

            for (int y = 1; y < maxRows; y++)
                AddSlot(path, maxCols - 1, y);

            for (int x = maxCols - 2; x >= centerX; x--)
                AddSlot(path, x, topRow);
        }
        else if (borderPos.y == maxRows - 1) // TOP
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

    private void AddSlot(List<Vector3> path, int x, int y)
    {
        var slot = SpawnCars.Instance.GetSlotAt(x, y);
        if (slot == null) return;
        if (path.Count > 0 && Vector3.Distance(path[path.Count - 1], slot.WorldPosition) < 0.01f)
            return;

        path.Add(slot.WorldPosition);
    }

    private Vector2Int GetDirectionFromAngle(float angle)
    {
        angle = Mathf.Repeat(Mathf.Round(angle / 45f) * 45f, 360f);

        switch ((int)angle)
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
