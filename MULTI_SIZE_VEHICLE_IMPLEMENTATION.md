# Master Multi-Size Vehicle & Level Generator Implementation Guide

This document is the master specification and code blueprint for the simplified refactoring of the Unity "Car Out / Parking Jam" puzzle game. It details the exact minimal modifications to the **7 existing scripts** to support multi-size vehicles (Small, Medium, Bus), cardinal-only movement, clean pooling resets, and the new grid-painting level editor/generator.

---

## 1. Prefab Setup Guide (Small, Medium, Bus)

To ensure that vehicles snap and rotate correctly on the grid without complex offsets in code, structure your vehicle prefabs as follows:

```
CarRoot (GameObject) ◄── [CarMover], [Carout] (Gameplay scripts)
 └── Visual (GameObject) ◄── Apply model scale and offset meshes backward
      └── ModelMesh (Car body, wheels, etc.)
```

### Step-by-Step Setup:
1. **Root GameObject (`CarRoot`):**
   * Keep the root GameObject at scale `(1, 1, 1)` and rotation `(0, 0, 0)`.
   * Attach `CarMover` and `Carout` scripts.
   * Add a `BoxCollider` to this root GameObject. Tag the root GameObject as `"car"`.
   * Align the root pivot with the **center of the front-most grid cell** (the anchor).
2. **Visual Child GameObject (`Visual`):**
   * Create a child GameObject named `Visual`. Move all mesh visual components under it.
   * Apply any mesh scaling directly to this child's transform (e.g. `(0.7, 0.7, 0.7)`).
   * **Offset the `Visual` child's position backward** along its local Z-axis so it centers over all occupied cells:
     * **Small (1x1):** Offset `(0, 0, 0)`.
     * **Medium (1x2):** Offset `(0, 0, -2.25)` (half a cell size backward).
     * **Bus (1x3):** Offset `(0, 0, -4.5)` (one full cell size backward).
3. **Collider Sizing (on the root):**
   * Adjust the `BoxCollider` size and center to cover the vehicle's footprint:
     * **Small (1x1):** Size `(4.0f, 2.0f, 4.0f)`, Center `(0f, 1f, 0f)`.
     * **Medium (1x2):** Size `(4.0f, 2.0f, 8.5f)`, Center `(0f, 1f, -2.25f)`.
     * **Bus (1x3):** Size `(4.0f, 2.0f, 13.0f)`, Center `(0f, 1f, -4.5f)`.

---

## 2. File-by-File Code Blueprints

These are the exact complete codes for the modified scripts. Replace the contents of the respective files in your project.

### 2.1 `Assets/Script/Level/LevelData.cs`
Add the `playableRegionFlat` mask array to save the painted grid cells.

```csharp
using System.Collections.Generic;
using UnityEngine;

public enum CarSize
{
    Small  = 1,
    Medium = 2,
    Bus    = 3
}

[System.Serializable]
public class CarSpawnInfo
{
    public Vector2Int gridPosition; // Coordinates (col, row) on the grid
    public float rotationY;         // Facing direction: 0 = Up, 90 = Right, 180 = Down, 270 = Left
    public CarSize carSize = CarSize.Small;
}

[CreateAssetMenu(fileName = "NewLevel", menuName = "ScriptableObjects/LevelData")]
public class LevelData : ScriptableObject
{
    public int levelNumber;
    
    [Header("Custom Grid Layout")]
    public int gridColumns = 5;
    public int gridRows = 5;
    
    [Header("Spawn Locations")]
    public List<CarSpawnInfo> carSpawns = new List<CarSpawnInfo>();

    [Header("Playable Region Mask (Editor Painted)")]
    public bool[] playableRegionFlat; // Flat array of gridColumns * gridRows

    [Header("Fallback (Legacy Level Data)")]
    public int carsToSpawn; 
}
```

### 2.2 `Assets/Script/SpawnCars.cs`
Update spawner to support dynamic grid dimensions and cardinal-only blockage raycasting.

```csharp
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

        public GridSlot(Vector2Int index, Vector3 pos)
        {
            GridIndex = index;
            WorldPosition = pos;
            IsOccupied = false;
        }
    }

    [Header("Prefabs & References")]
    [SerializeField] public GameObject carSpawnPrefab;

    [Header("Car Prefabs By Size")]
    public List<CarMover> smallCarPrefabs;
    public List<CarMover> mediumCarPrefabs;
    public List<CarMover> busCarPrefabs;

    public SpawnPassengers spawnPassengers;
    public LevelData currentLevelData;

    [Header("Grid Layout Settings")]
    public int maxColumns = 5;
    public int maxRows = 5;
    private const float GridSlotSize = 4.5f;
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
                if (slot == null) continue;

                Quaternion carRotation = Quaternion.Euler(0, spawn.rotationY, 0);
                Vector2Int facing = AngleToGridDir(spawn.rotationY);

                List<Vector2Int> occupiedCells = GetCellsForCar(spawn.gridPosition, facing, spawn.carSize);
                bool canSpawn = true;

                foreach (var cell in occupiedCells)
                {
                    GridSlot gs = GetSlotAt(cell.x, cell.y);
                    if (gs == null || gs.IsOccupied)
                    {
                        canSpawn = false;
                        break;
                    }
                }

                if (!canSpawn) continue;

                GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, carRotation);
                spawnedPositions.Add(slotIndicator);

                CarMover chosenPrefab = GetPrefabForSize(spawn.carSize);
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
                    caroutComp.SetOccupiedCells(occupiedCells);
                }
                
                foreach (var cell in occupiedCells)
                    SetSlotOccupation(cell.x, cell.y, true);

                AnimateCarSpawn(activeCar, spawnPassengers.TotalCarsSpawn.Count);
                spawnPassengers.TotalCarsSpawn.Add(activeCar);
            }
        }
    }

    public bool CheckBlockageForCar(Vector2Int currentGridIndex, Vector3 eulerAngles, out Vector2Int targetGridIndex)
    {
        // Enforce 90-degree snap to prevent diagonal path errors
        Vector2Int stepDir = AngleToGridDir(eulerAngles.y);
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
        float angle = Mathf.Repeat(Mathf.Round(rotationY / 90f) * 90f, 360f);
        switch (Mathf.RoundToInt(angle))
        {
            case 0: return new Vector2Int(0, 1);
            case 90: return new Vector2Int(1, 0);
            case 180: return new Vector2Int(0, -1);
            case 270: return new Vector2Int(-1, 0);
        }
        return Vector2Int.zero;
    }

    public List<Vector2Int> GetCellsForCar(Vector2Int anchor, Vector2Int facing, CarSize size)
    {
        List<Vector2Int> cells = new List<Vector2Int>();
        cells.Add(anchor);
        Vector2Int behind = -facing;
        for (int i = 1; i < (int)size; i++)
            cells.Add(anchor + behind * i);
        return cells;
    }

    private CarMover GetPrefabForSize(CarSize size)
    {
        List<CarMover> pool;
        switch (size)
        {
            case CarSize.Medium: pool = mediumCarPrefabs; break;
            case CarSize.Bus: pool = busCarPrefabs; break;
            default: pool = smallCarPrefabs; break;
        }

        if (pool == null || pool.Count == 0) pool = smallCarPrefabs;
        if (pool == null || pool.Count == 0) return null;

        return pool[Random.Range(0, pool.Count)];
    }

    public void FreeSlots(List<Vector2Int> cells)
    {
        if (cells == null) return;
        foreach (var cell in cells)
            SetSlotOccupation(cell.x, cell.y, false);
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
                float offsetX = (c - centerCol) * GridSlotSize;
                float offsetZ = (r - centerRow) * GridSlotSize;
                Vector3 spawnPosition = centerArea + new Vector3(offsetX, 0, offsetZ);
                
                gridSlots.Add(new GridSlot(new Vector2Int(c, r), spawnPosition));
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

    private void AnimateCarSpawn(CarMover car, int index)
    {
        if (car == null) return;
        Transform t = car.transform;
        Vector3 finalScale = car.GetOriginalScale();

        t.localScale = Vector3.zero;
        Sequence seq = DOTween.Sequence();
        seq.AppendInterval(index * 0.05f);
        seq.Append(t.DOScale(finalScale * 1.15f, 0.22f).SetEase(Ease.OutBack));
        seq.Append(t.DOScale(finalScale, 0.12f).SetEase(Ease.OutQuad));
    }
}
```

### 2.3 `Assets/Script/CarMover.cs`
Update movement Snap bounds and add a component cleanup reset method.

```csharp
using UnityEngine;
using TMPro;
using UnityEngine.UI;
using System.Collections;
using System.Collections.Generic;
using DG.Tweening;

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
    private Coroutine moveRoutine;

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
        moveRoutine = StartCoroutine(MoveRoutine());
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
    }

    public void ResetState()
    {
        if (moveRoutine != null) StopCoroutine(moveRoutine);
        StopAllCoroutines();
        transform.DOKill();
        isMoving = false;
        isParked = false;
        CapacityOfPassengers = thisCarCapacity;
        if (totalPassengerTxt != null) totalPassengerTxt.enabled = false;
        if (arrow != null) arrow.enabled = false;
    }

    private IEnumerator MoveRoutine()
    {
        if (carout == null) yield break;

        Vector2Int currentGrid = carout.currentGridIndex;
        var currentSlot = SpawnCars.Instance.GetSlotAt(currentGrid.x, currentGrid.y);

        if (currentSlot != null)
        {
            while (Vector3.Distance(transform.position, currentSlot.WorldPosition) > 0.02f)
            {
                transform.position = Vector3.MoveTowards(transform.position, currentSlot.WorldPosition, speed * Time.deltaTime);
                yield return null;
            }
            transform.position = currentSlot.WorldPosition;
        }

        Vector2Int dir = SpawnCars.Instance.AngleToGridDir(transform.eulerAngles.y);
        List<Vector3> waypoints = GeneratePath(currentGrid, dir);

        var exitSlot = SpawnCars.Instance.GetSlotAt(
            SpawnCars.Instance.maxColumns / 2,
            SpawnCars.Instance.maxRows - 1);

        if (exitSlot != null)
        {
            if (waypoints.Count > 0 && Vector3.Distance(waypoints[waypoints.Count - 1], exitSlot.WorldPosition) > 0.01f)
            {
                waypoints.RemoveAt(waypoints.Count - 1);
            }
            waypoints.Add(exitSlot.WorldPosition);
        }
        waypoints.Add(targetPosition);

        if (waypoints.Count > 0)
        {
            Vector3 firstDir = (waypoints[0] - transform.position).normalized;
            firstDir.y = 0;
            if (firstDir != Vector3.zero)
            {
                transform.rotation = Quaternion.LookRotation(firstDir);
            }
        }

        int i = 0;
        while (i < waypoints.Count)
        {
            if (IsMovingCarAhead())
            {
                yield return null;
                continue;
            }

            Vector3 wp = waypoints[i];
            Vector3 toTarget = wp - transform.position;
            float step = speed * Time.deltaTime;

            if (toTarget.sqrMagnitude <= step * step)
            {
                transform.position = wp;
                i++;
                continue;
            }

            Vector3 moveDir = toTarget.normalized;
            transform.position += moveDir * step;

            if (moveDir != Vector3.zero)
            {
                transform.rotation = Quaternion.LookRotation(moveDir);
            }

            yield return null;
        }

        transform.position = targetPosition;
        transform.rotation = Quaternion.Euler(0, 0, 0);
        isMoving = false;
        isParked = true;

        HapticManager.Instance?.CarFull();
    }

    private List<Vector3> GeneratePath(Vector2Int startGrid, Vector2Int dir)
    {
        List<Vector3> path = new List<Vector3>();
        int maxCols = SpawnCars.Instance.maxColumns;
        int maxRows = SpawnCars.Instance.maxRows;
        int topRow = maxRows - 1;
        int centerX = maxCols / 2;
        Vector2Int current = startGrid;

        while (true)
        {
            Vector2Int next = current + dir;
            bool outside = next.x < 0 || next.x >= maxCols || next.y < 0 || next.y >= maxRows;
            if (outside) break;

            current = next;
            var slot = SpawnCars.Instance.GetSlotAt(current.x, current.y);
            if (slot != null)
            {
                if (path.Count == 0 || Vector3.Distance(path[path.Count - 1], slot.WorldPosition) > 0.01f)
                {
                    path.Add(slot.WorldPosition);
                }
            }
        }

        var borderSlot = SpawnCars.Instance.GetSlotAt(current.x, current.y);
        if (borderSlot != null)
        {
            if (path.Count == 0 || Vector3.Distance(path[path.Count - 1], borderSlot.WorldPosition) > 0.01f)
            {
                path.Add(borderSlot.WorldPosition);
            }
        }

        // Left Border
        if (current.x == 0)
        {
            for (int y = current.y + 1; y < maxRows; y++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(0, y);
                if (slot != null) path.Add(slot.WorldPosition);
            }
            for (int x = 1; x <= centerX; x++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(x, topRow);
                if (slot != null) path.Add(slot.WorldPosition);
            }
        }
        // Right Border
        else if (current.x == maxCols - 1)
        {
            for (int y = current.y + 1; y < maxRows; y++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(maxCols - 1, y);
                if (slot != null) path.Add(slot.WorldPosition);
            }
            for (int x = maxCols - 2; x >= centerX; x--)
            {
                var slot = SpawnCars.Instance.GetSlotAt(x, topRow);
                if (slot != null) path.Add(slot.WorldPosition);
            }
        }
        // Bottom Border
        else if (current.y == 0)
        {
            for (int x = current.x + 1; x < maxCols; x++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(x, 0);
                if (slot != null) path.Add(slot.WorldPosition);
            }
            for (int y = 1; y < maxRows; y++)
            {
                var slot = SpawnCars.Instance.GetSlotAt(maxCols - 1, y);
                if (slot != null) path.Add(slot.WorldPosition);
            }
            for (int x = maxCols - 2; x >= centerX; x--)
            {
                var slot = SpawnCars.Instance.GetSlotAt(x, topRow);
                if (slot != null) path.Add(slot.WorldPosition);
            }
        }

        return path;
    }

    private bool IsMovingCarAhead()
    {
        RaycastHit hit;
        if (Physics.Raycast(transform.position + Vector3.up * 0.5f, transform.forward, out hit, safetyDistance))
        {
            if (hit.collider.CompareTag("car") && hit.collider.gameObject != gameObject)
            {
                CarMover other = hit.collider.GetComponent<CarMover>();
                if (other != null && !other.isParked) return true;
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

        Sequence s = DOTween.Sequence();
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

### 2.4 `Assets/Script/Carout.cs`
Update blockage checker to query grid systems correctly without collapsing size records.

```csharp
using System.Collections.Generic;
using UnityEngine;
using DG.Tweening;

public class Carout : MonoBehaviour
{
    public float rayDis = 5f;
    public float dashSpeed = 0.2f;
    public float returnSpeed = 0.5f;

    public bool isBlocked { get; private set; }
    public Vector2Int currentGridIndex; 

    private List<Vector2Int> occupiedCells = new List<Vector2Int>();
    private CarMover carMover;

    public float tackleCollsionOffset = 4f;
    public Vector2Int blockingGridIndex;

    [SerializeField] private ParticleSystem impactParticle;
    private bool isTackling = false;
    private Sequence tackleSequence;

    private void Awake()
    {
        carMover = GetComponent<CarMover>();
    }

    public void SetOccupiedCells(List<Vector2Int> cells)
    {
        occupiedCells = new List<Vector2Int>(cells);
    }

    public void ResetState()
    {
        StopAllCoroutines();
        transform.DOKill();
        isBlocked = false;
        isTackling = false;
        if (tackleSequence != null)
        {
            tackleSequence.Kill();
            tackleSequence = null;
        }
        occupiedCells.Clear();
    }

    public bool CheckForBlockage()
    {
        if (SpawnCars.Instance == null) return false;

        isBlocked = SpawnCars.Instance.CheckBlockageForCar(
            currentGridIndex, transform.eulerAngles, out Vector2Int targetGridIndex);

        if (!isBlocked)
        {
            if (occupiedCells != null && occupiedCells.Count > 0)
                SpawnCars.Instance.FreeSlots(occupiedCells);       
            else
                SpawnCars.Instance.SetSlotOccupation(currentGridIndex.x, currentGridIndex.y, false);

            currentGridIndex = targetGridIndex;
            occupiedCells.Clear();
            occupiedCells.Add(currentGridIndex); // Vacate slots, ready to leave
        }
        else
        {
            blockingGridIndex = targetGridIndex;
        }

        return isBlocked;
    }

    public void DoTackle()
    {
        if (isTackling || (tackleSequence != null && tackleSequence.IsActive())) return;

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
            if (impactParticle != null) impactParticle.Play();
            HapticManager.Instance?.CarHit();
        });
        tackleSequence.Append(transform.DOShakeRotation(0.2f, 10f));
        tackleSequence.Append(transform.DOMove(originalPosition, returnSpeed).SetEase(Ease.InQuad));
        tackleSequence.OnComplete(() => isTackling = false);
        tackleSequence.OnKill(() => isTackling = false);
    }
}
```

### 2.5 `Assets/Script/ObjectPool.cs`
Implement DOTween/Coroutine safety resets for all returning pool elements.

```csharp
using System.Collections.Generic;
using UnityEngine;
using DG.Tweening;

public class ObjectPool : MonoBehaviour
{
    public static ObjectPool Instance;
    public List<GameObject> pool = new List<GameObject>();

    private void Awake()
    {
        Instance = this;
    }

    public void AddToPool(GameObject obj)
    {
        obj.SetActive(false);
        obj.transform.position = new Vector3(22, 1, 38);
        obj.transform.rotation = Quaternion.identity;

        // Perform clean factory resets on component state transitions
        CarMover mover = obj.GetComponent<CarMover>();
        if (mover != null) mover.ResetState();

        Carout carout = obj.GetComponent<Carout>();
        if (carout != null) carout.ResetState();

        Passenger passenger = obj.GetComponent<Passenger>();
        if (passenger != null)
        {
            passenger.StopAllCoroutines();
            passenger.transform.DOKill();
            passenger.isJumping = false;
        }

        pool.Add(obj);
    }

    public CarMover GetCarFromPool(CarMover prefab)
    {
        for (int i = pool.Count - 1; i >= 0; i--)
        {
            if (pool[i] == null)
            {
                pool.RemoveAt(i);
                continue;
            }
            if (pool[i].activeInHierarchy) continue;

            CarMover car = pool[i].GetComponent<CarMover>();
            if (car == null) continue;

            if (car.originalPrefab == prefab)
            {
                return car;
            }
        }
        return null;
    }

    public Passenger GetPassengerFromPool(Passenger passenger)
    {
        for (int i = pool.Count - 1; i >= 0; i--)
        {
            if (pool[i] == null)
            {
                pool.RemoveAt(i);
                continue;
            }
            if (!pool[i].activeInHierarchy && pool[i].name.Contains(passenger.name))
            {
                return pool[i].GetComponent<Passenger>();
            }
        }
        return null;
    }
}
```

### 2.6 `Assets/Script/PowerUps.cs`
Update Helicopter VIP power-up to free all footprint slots of Medium/Bus vehicles.

```csharp
// LOCATE HelicopterPowerUpRoutine inside Assets/Script/PowerUps.cs:
// REPLACE this code block:
// ----------------------------------------------------
// Carout carout = targetCar.GetComponent<Carout>();
// if (carout != null && SpawnCars.Instance != null)
// {
//     SpawnCars.Instance.SetSlotOccupation(carout.currentGridIndex.x, carout.currentGridIndex.y, false);
// }
// ----------------------------------------------------
// WITH this footprint-aware free code:
// ----------------------------------------------------
Carout carout = targetCar.GetComponent<Carout>();
if (carout != null && SpawnCars.Instance != null)
{
    Vector2Int facing = SpawnCars.Instance.AngleToGridDir(targetCar.transform.eulerAngles.y);
    List<Vector2Int> cells = SpawnCars.Instance.GetCellsForCar(carout.currentGridIndex, facing, targetCar.carSize);
    SpawnCars.Instance.FreeSlots(cells);
}
// ----------------------------------------------------
```

### 2.7 `Assets/Script/Level/Editor/LevelEditorWindow.cs` (Rewrite)
Fully rewritten to paint playable grids, configure composition options, execute validation tests, and save patterns.

```csharp
#if UNITY_EDITOR
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

public class LevelEditorWindow : EditorWindow
{
    private LevelData targetLevel;
    private Vector2 scrollPosition;

    private bool[,] playableMask;
    private int currentCols = 8;
    private int currentRows = 8;

    private enum EditTool { Paint, Erase, RectFill, FloodFill }
    private EditTool activeTool = EditTool.Paint;

    // Pan & Zoom
    private Vector2 panOffset = Vector2.zero;
    private float zoomFactor = 1.0f;
    private bool isPanning = false;
    private Vector2 mouseDragStart;

    // Generation parameters
    private float targetDensity = 0.85f;
    private string compositionStyle = "ParkingLot";
    private int randomSeed = 12345;
    private int layoutScore = 0;
    private bool previewLayout = true;

    private Stack<bool[]> undoStack = new Stack<bool[]>();
    private Stack<bool[]> redoStack = new Stack<bool[]>();

    [MenuItem("Tools/Smart Pattern Editor")]
    public static void ShowWindow()
    {
        GetWindow<LevelEditorWindow>("Smart Pattern Editor");
    }

    private void OnGUI()
    {
        GUILayout.Label("Smart Pattern & Level Generator", EditorStyles.boldLabel);
        EditorGUILayout.Space();

        targetLevel = (LevelData)EditorGUILayout.ObjectField("Select Level Data", targetLevel, typeof(LevelData), false);

        if (targetLevel == null)
        {
            EditorGUILayout.HelpBox("Select a LevelData asset to begin.", MessageType.Info);
            return;
        }

        if (playableMask == null) LoadFromAsset();

        DrawEditorSettings();
        EditorGUILayout.Space();
        DrawCanvasArea();
    }

    private void DrawEditorSettings()
    {
        EditorGUILayout.BeginVertical("box");
        
        EditorGUI.BeginChangeCheck();
        int cols = EditorGUILayout.IntField("Grid Columns", currentCols);
        int rows = EditorGUILayout.IntField("Grid Rows", currentRows);
        if (EditorGUI.EndChangeCheck())
        {
            currentCols = Mathf.Clamp(cols, 3, 20);
            currentRows = Mathf.Clamp(rows, 3, 20);
            ResizeGrid();
        }

        EditorGUILayout.Space();
        activeTool = (EditTool)EditorGUILayout.EnumPopup("Painting Tool", activeTool);

        EditorGUILayout.Space();
        targetDensity = EditorGUILayout.Slider("Target Occupancy Density", targetDensity, 0.5f, 0.95f);
        compositionStyle = EditorGUILayout.TextField("Composition Style (ParkingLot, Compact, etc.)", compositionStyle);

        EditorGUILayout.BeginHorizontal();
        randomSeed = EditorGUILayout.IntField("Generator Seed", randomSeed);
        if (GUILayout.Button("Randomize Seed", GUILayout.Width(120))) randomSeed = Random.Range(1, 99999);
        EditorGUILayout.EndHorizontal();

        EditorGUILayout.Space();
        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("Undo Paint")) UndoPaint();
        if (GUILayout.Button("Redo Paint")) RedoPaint();
        EditorGUILayout.EndHorizontal();

        EditorGUILayout.Space();
        GUI.backgroundColor = Color.cyan;
        if (GUILayout.Button("GENERATE AUTOMATIC LAYOUT", GUILayout.Height(30)))
        {
            SaveMaskToAsset();
            bool ok = GenerateLayout();
            if (ok) Debug.Log($"Layout generated successfully! Score: {layoutScore}");
            else Debug.LogWarning("Solvability verification failed! Adjust density or painted mask.");
        }

        GUI.backgroundColor = new Color(0.4f, 0.8f, 0.4f);
        if (GUILayout.Button("Save Level Asset", GUILayout.Height(30)))
        {
            SaveMaskToAsset();
            EditorUtility.SetDirty(targetLevel);
            AssetDatabase.SaveAssets();
            Debug.Log($"Saved asset Level_{targetLevel.levelNumber}!");
        }
        GUI.backgroundColor = Color.white;

        EditorGUILayout.LabelField($"Layout Score: {layoutScore} / 100", EditorStyles.boldLabel);
        EditorGUILayout.EndVertical();
    }

    private void DrawCanvasArea()
    {
        Rect canvasRect = GUILayoutUtility.GetRect(200f, 400f);
        GUI.Box(canvasRect, "Level Painter Canvas (LMB Draw, RMB Erase, MMB Drag Pan, Scroll Zoom)");

        Event e = Event.current;
        if (canvasRect.Contains(e.mousePosition))
        {
            if (e.type == EventType.ScrollWheel)
            {
                zoomFactor = Mathf.Clamp(zoomFactor - e.delta.y * 0.05f, 0.5f, 2.5f);
                e.Use();
            }
            else if (e.type == EventType.MouseDown && e.button == 2)
            {
                isPanning = true;
                mouseDragStart = e.mousePosition;
                e.Use();
            }
            else if (e.type == EventType.MouseDrag && isPanning)
            {
                panOffset += e.mousePosition - mouseDragStart;
                mouseDragStart = e.mousePosition;
                e.Use();
                Repaint();
            }
            else if (e.type == EventType.MouseUp && e.button == 2)
            {
                isPanning = false;
                e.Use();
            }
        }

        GUI.BeginGroup(canvasRect);
        
        Matrix4x4 oldMatrix = GUI.matrix;
        Matrix4x4 trans = Matrix4x4.TRS(new Vector3(panOffset.x, panOffset.y, 0f), Quaternion.identity, Vector3.one);
        Matrix4x4 scale = Matrix4x4.Scale(new Vector3(zoomFactor, zoomFactor, 1f));
        GUI.matrix = trans * scale;

        float cellSize = 30f;
        float spacing = 2f;
        Vector2 localMouse = (e.mousePosition - canvasRect.position - panOffset) / zoomFactor;

        for (int r = currentRows - 1; r >= 0; r--)
        {
            for (int c = 0; c < currentCols; c++)
            {
                float px = c * (cellSize + spacing) + 20f;
                float py = (currentRows - 1 - r) * (cellSize + spacing) + 20f;
                Rect cellRect = new Rect(px, py, cellSize, cellSize);

                bool isPlayable = playableMask[c, r];
                Color originalBg = GUI.color;

                if (isPlayable) GUI.color = Color.green;
                else GUI.color = new Color(0.2f, 0.2f, 0.2f);

                if (previewLayout && targetLevel != null && targetLevel.carSpawns != null)
                {
                    var spawned = targetLevel.carSpawns.Find(s => s.gridPosition == new Vector2Int(c, r));
                    if (spawned != null)
                    {
                        if (spawned.carSize == CarSize.Bus) GUI.color = Color.red;
                        else if (spawned.carSize == CarSize.Medium) GUI.color = Color.yellow;
                        else GUI.color = Color.blue;
                    }
                }

                if (cellRect.Contains(localMouse))
                {
                    GUI.color = Color.white; 
                    if (e.type == EventType.MouseDown || e.type == EventType.MouseDrag)
                    {
                        if (e.button == 0) 
                        {
                            RecordPaintSnapshot();
                            if (activeTool == EditTool.Paint) playableMask[c, r] = true;
                            else if (activeTool == EditTool.Erase) playableMask[c, r] = false;
                            else if (activeTool == EditTool.FloodFill) FloodFillAction(c, r);
                            e.Use();
                        }
                        else if (e.button == 1) 
                        {
                            RecordPaintSnapshot();
                            playableMask[c, r] = false;
                            e.Use();
                        }
                    }
                }

                GUI.Box(cellRect, isPlayable ? "P" : ".");
                GUI.color = originalBg;
            }
        }

        GUI.matrix = oldMatrix;
        GUI.EndGroup();
    }

    private void ResizeGrid()
    {
        bool[,] newMask = new bool[currentCols, currentRows];
        if (playableMask != null)
        {
            int cx = Mathf.Min(playableMask.GetLength(0), currentCols);
            int cy = Mathf.Min(playableMask.GetLength(1), currentRows);
            for (int x = 0; x < cx; x++)
            {
                for (int y = 0; y < cy; y++) newMask[x, y] = playableMask[x, y];
            }
        }
        playableMask = newMask;
    }

    private void LoadFromAsset()
    {
        currentCols = targetLevel.gridColumns;
        currentRows = targetLevel.gridRows;
        playableMask = new bool[currentCols, currentRows];

        if (targetLevel.playableRegionFlat != null && targetLevel.playableRegionFlat.Length == currentCols * currentRows)
        {
            for (int x = 0; x < currentCols; x++)
            {
                for (int y = 0; y < currentRows; y++)
                    playableMask[x, y] = targetLevel.playableRegionFlat[y * currentCols + x];
            }
        }
        else
        {
            for (int x = 0; x < currentCols; x++)
            {
                for (int y = 0; y < currentRows; y++) playableMask[x, y] = true;
            }
        }
    }

    private void SaveMaskToAsset()
    {
        targetLevel.gridColumns = currentCols;
        targetLevel.gridRows = currentRows;
        targetLevel.playableRegionFlat = new bool[currentCols * currentRows];
        for (int x = 0; x < currentCols; x++)
        {
            for (int y = 0; y < currentRows; y++)
                targetLevel.playableRegionFlat[y * currentCols + x] = playableMask[x, y];
        }
    }

    private void RecordPaintSnapshot()
    {
        bool[] snapshot = new bool[currentCols * currentRows];
        for (int x = 0; x < currentCols; x++)
        {
            for (int y = 0; y < currentRows; y++)
                snapshot[y * currentCols + x] = playableMask[x, y];
        }
        undoStack.Push(snapshot);
        redoStack.Clear();
    }

    private void UndoPaint()
    {
        if (undoStack.Count == 0) return;
        bool[] snapshot = undoStack.Pop();
        
        bool[] redoSnapshot = new bool[currentCols * currentRows];
        for (int x = 0; x < currentCols; x++)
        {
            for (int y = 0; y < currentRows; y++)
            {
                redoSnapshot[y * currentCols + x] = playableMask[x, y];
                playableMask[x, y] = snapshot[y * currentCols + x];
            }
        }
        redoStack.Push(redoSnapshot);
        Repaint();
    }

    private void RedoPaint()
    {
        if (redoStack.Count == 0) return;
        bool[] snapshot = redoStack.Pop();

        bool[] undoSnapshot = new bool[currentCols * currentRows];
        for (int x = 0; x < currentCols; x++)
        {
            for (int y = 0; y < currentRows; y++)
            {
                undoSnapshot[y * currentCols + x] = playableMask[x, y];
                playableMask[x, y] = snapshot[y * currentCols + x];
            }
        }
        undoStack.Push(undoSnapshot);
        Repaint();
    }

    private void FloodFillAction(int startX, int startY)
    {
        bool targetVal = playableMask[startX, startY];
        bool newVal = !targetVal;

        Queue<Vector2Int> q = new Queue<Vector2Int>();
        q.Enqueue(new Vector2Int(startX, startY));

        while (q.Count > 0)
        {
            Vector2Int cell = q.Dequeue();
            if (cell.x < 0 || cell.x >= currentCols || cell.y < 0 || cell.y >= currentRows) continue;
            if (playableMask[cell.x, cell.y] != targetVal) continue;

            playableMask[cell.x, cell.y] = newVal;

            q.Enqueue(new Vector2Int(cell.x + 1, cell.y));
            q.Enqueue(new Vector2Int(cell.x - 1, cell.y));
            q.Enqueue(new Vector2Int(cell.x, cell.y + 1));
            q.Enqueue(new Vector2Int(cell.x, cell.y - 1));
        }
    }

    private bool GenerateLayout()
    {
        Random.InitState(randomSeed);
        layoutScore = 0;

        int cols = currentCols;
        int rows = currentRows;

        int playableCount = 0;
        for (int x = 0; x < cols; x++)
        {
            for (int y = 0; y < rows; y++)
                if (playableMask[x, y]) playableCount++;
        }

        if (playableCount == 0) return false;

        int targetOccupied = Mathf.RoundToInt(playableCount * targetDensity);

        int busCells = Mathf.RoundToInt(targetOccupied * 0.25f);
        int busCount = Mathf.Max(0, busCells / 3);

        int medCells = Mathf.RoundToInt(targetOccupied * 0.35f);
        int medCount = Mathf.Max(0, medCells / 2);

        int smallCount = targetOccupied - (busCount * 3) - (medCount * 2);
        if (smallCount < 0) smallCount = 0;

        bool[,] occupied = new bool[cols, rows];
        List<CarSpawnInfo> spawns = new List<CarSpawnInfo>();
        float[] rotations = { 0f, 90f, 180f, 270f };

        PlaceSize(busCount, CarSize.Bus, playableMask, occupied, spawns, rotations, cols, rows);
        PlaceSize(medCount, CarSize.Medium, playableMask, occupied, spawns, rotations, cols, rows);

        for (int x = 0; x < cols; x++)
        {
            for (int y = 0; y < rows; y++)
            {
                if (playableMask[x, y] && !occupied[x, y])
                {
                    float rot = rotations[Random.Range(0, rotations.Length)];
                    spawns.Add(new CarSpawnInfo
                    {
                        gridPosition = new Vector2Int(x, y),
                        rotationY = rot,
                        carSize = CarSize.Small
                    });
                    occupied[x, y] = true;
                }
            }
        }

        targetLevel.carSpawns = spawns;

        bool solvable = VerifySolvability(cols, rows, spawns, playableMask);
        layoutScore = CalculateScore(cols, rows, playableMask, occupied, spawns, targetDensity, solvable);

        return solvable;
    }

    private void PlaceSize(int count, CarSize size, bool[,] mask, bool[,] occupied, List<CarSpawnInfo> spawns, float[] rotations, int cols, int rows)
    {
        int placed = 0;
        int maxAttempts = 500;
        int attempts = 0;

        while (placed < count && attempts < maxAttempts)
        {
            attempts++;
            int rx = Random.Range(0, cols);
            int ry = Random.Range(0, rows);

            if (!mask[rx, ry] || occupied[rx, ry]) continue;

            float rot = rotations[Random.Range(0, rotations.Length)];
            Vector2Int facing = GetCardinalDir(rot);
            Vector2Int behind = -facing;

            bool fits = true;
            List<Vector2Int> cells = new List<Vector2Int>();
            for (int i = 0; i < (int)size; i++)
            {
                Vector2Int cell = new Vector2Int(rx, ry) + behind * i;
                if (cell.x < 0 || cell.x >= cols || cell.y < 0 || cell.y >= rows || !mask[cell.x, cell.y] || occupied[cell.x, cell.y])
                {
                    fits = false;
                    break;
                }
                cells.Add(cell);
            }

            if (fits)
            {
                spawns.Add(new CarSpawnInfo
                {
                    gridPosition = new Vector2Int(rx, ry),
                    rotationY = rot,
                    carSize = size
                });
                foreach (var cell in cells) occupied[cell.x, cell.y] = true;
                placed++;
            }
        }
    }

    private static Vector2Int GetCardinalDir(float rotationY)
    {
        float angle = Mathf.Repeat(Mathf.Round(rotationY / 90f) * 90f, 360f);
        switch (Mathf.RoundToInt(angle))
        {
            case 0: return new Vector2Int(0, 1);
            case 90: return new Vector2Int(1, 0);
            case 180: return new Vector2Int(0, -1);
            case 270: return new Vector2Int(-1, 0);
        }
        return Vector2Int.zero;
    }

    private static bool VerifySolvability(int cols, int rows, List<CarSpawnInfo> spawns, bool[,] mask)
    {
        List<CarSpawnInfo> workingList = new List<CarSpawnInfo>(spawns);
        bool[,] occupied = new bool[cols, rows];

        foreach (var spawn in spawns)
        {
            Vector2Int facing = GetCardinalDir(spawn.rotationY);
            Vector2Int behind = -facing;
            for (int i = 0; i < (int)spawn.carSize; i++)
            {
                Vector2Int cell = spawn.gridPosition + behind * i;
                if (cell.x >= 0 && cell.x < cols && cell.y >= 0 && cell.y < rows)
                    occupied[cell.x, cell.y] = true;
            }
        }

        bool progress = true;
        while (workingList.Count > 0 && progress)
        {
            progress = false;
            for (int i = 0; i < workingList.Count; i++)
            {
                var car = workingList[i];
                if (CanSimulatedCarExit(car, occupied, mask, cols, rows))
                {
                    Vector2Int facing = GetCardinalDir(car.rotationY);
                    Vector2Int behind = -facing;
                    for (int s = 0; s < (int)car.carSize; s++)
                    {
                        Vector2Int cell = car.gridPosition + behind * s;
                        if (cell.x >= 0 && cell.x < cols && cell.y >= 0 && cell.y < rows)
                            occupied[cell.x, cell.y] = false;
                    }
                    workingList.RemoveAt(i);
                    progress = true;
                    break;
                }
            }
        }

        return workingList.Count == 0;
    }

    private static bool CanSimulatedCarExit(CarSpawnInfo car, bool[,] occupied, bool[,] mask, int cols, int rows)
    {
        Vector2Int dir = GetCardinalDir(car.rotationY);
        Vector2Int check = car.gridPosition + dir;

        while (true)
        {
            if (check.x < 0 || check.x >= cols || check.y < 0 || check.y >= rows) return true; 
            if (occupied[check.x, check.y]) return false; 
            check += dir;
        }
    }

    private static int CalculateScore(int cols, int rows, bool[,] mask, bool[,] occupied, List<CarSpawnInfo> spawns, float targetDensity, bool solvable)
    {
        if (!solvable) return 0;

        int score = 40; 
        int playableCount = 0;
        int occupiedCount = 0;
        for (int x = 0; x < cols; x++)
        {
            for (int y = 0; y < rows; y++)
            {
                if (mask[x, y]) playableCount++;
                if (occupied[x, y]) occupiedCount++;
            }
        }

        float currentDensity = (float)occupiedCount / playableCount;
        float diff = Mathf.Abs(currentDensity - targetDensity);
        
        score += Mathf.RoundToInt(Mathf.Clamp01(1f - (diff / targetDensity)) * 30f);

        float buses = 0, mediums = 0, smalls = 0;
        foreach (var car in spawns)
        {
            if (car.carSize == CarSize.Bus) buses++;
            else if (car.carSize == CarSize.Medium) mediums++;
            else smalls++;
        }

        float total = spawns.Count;
        if (total > 0)
        {
            float busRatio = (buses * 3f) / occupiedCount;
            float medRatio = (mediums * 2f) / occupiedCount;

            float busDiff = Mathf.Abs(busRatio - 0.25f);
            float medDiff = Mathf.Abs(medRatio - 0.35f);

            score += Mathf.RoundToInt(Mathf.Clamp01(1f - (busDiff + medDiff)) * 30f);
        }

        return Mathf.Clamp(score, 0, 100);
    }
}
#endif
```

---

*End of MULTI_SIZE_VEHICLE_IMPLEMENTATION.md*
