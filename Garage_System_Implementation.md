# Garage System — Complete Implementation Guide

Read this fully before touching ANY script.
Every section is self-contained. All new code is production-ready.

---

## Table of Contents
1. Feature Overview
2. Architecture Diagram
3. Step 1 - Modify LevelData.cs
4. Step 2 - New Script: GarageController.cs
5. Step 3 - Modify SpawnCars.cs
6. Step 4 - Modify LevelManager.cs
7. Step 5 - Modify CarController.cs
8. Step 6 - Modify CarMover.cs
9. Step 7 - Modify ObjectPool.cs
10. Step 8 - Modify LevelEditorWindow.cs
11. Step 9 - Unity Setup (Prefab + Scene)
12. How It All Works Together
13. Quick Checklist

---

## 1. Feature Overview

| Property | Detail |
|---|---|
| Grid footprint | Exactly 2 cells — garage body (cell A) + open mouth (cell B in front) |
| Clickability | Garage itself is NOT clickable. Only the frontmost car is clickable |
| Car types | Small, Medium, Big — configurable per garage |
| Car count | 1 to N — set in Level Editor |
| Exit animation | Next car slides from behind via DOTween after previous car leaves |
| Empty state | Garage pools itself, frees both cells |
| Direction | All cars face same direction as the garage |

---

## 2. Architecture Diagram

```
LevelData.cs
  GarageSpawnInfo (new serializable class)
    gridPosition, rotationY, carSize, carCount

SpawnCars.cs
  SpawnGarages() -> places GarageController prefabs
    GarageController.cs (new script)
      Queue of CarMovers
      bodyCell, mouthCell
      Initialize(), ReleaseFrontCar(), OnCarDeparted(), EmptyGarage()

CarController.cs
  HandleCarSelection()
    if car.garageOwner != null -> garage.ReleaseFrontCar()
    else -> normal move logic

LevelEditorWindow.cs
  "Garages" mode toggle
  Click cell -> place garage (rotation, carSize, carCount)
  Garage cells colored orange on grid
```

---

## 3. Step 1 - Modify LevelData.cs

**File:** Assets/Script/Level/LevelData.cs

Replace the entire file with this:

```csharp
using System.Collections.Generic;
using UnityEngine;

public enum CarSize
{
    Small = 1,
    Medium = 2,
    Big = 3
}

public enum PlacementMode
{
    Auto,
    Manual
}

[System.Serializable]
public class CarSpawnInfo
{
    public Vector2Int gridPosition;
    public float rotationY;
    public CarSize carSize = CarSize.Small;
    public PlacementMode placementMode = PlacementMode.Auto;
}

// NEW CLASS -------------------------------------------------------
[System.Serializable]
public class GarageSpawnInfo
{
    /// Body cell of the garage prefab (NOT the mouth cell).
    public Vector2Int gridPosition;

    /// Y-axis rotation (0=Up, 90=Right, 180=Down, 270=Left).
    public float rotationY;

    /// Size of every car stored inside this garage.
    public CarSize carSize = CarSize.Small;

    /// How many cars are queued inside (1-N).
    public int carCount = 2;
}
// -----------------------------------------------------------------

[CreateAssetMenu(fileName = "NewLevel", menuName = "ScriptableObjects/LevelData")]
public class LevelData : ScriptableObject
{
    public int levelNumber;
    
    [Header("Custom Grid Layout")]
    public int gridColumns = 5;
    public int gridRows = 5;
    
    [Header("Spawn Locations")]
    public List<CarSpawnInfo> carSpawns = new List<CarSpawnInfo>();

    // NEW FIELD -------------------------------------------------------
    [Header("Garages")]
    public List<GarageSpawnInfo> garageSpawns = new List<GarageSpawnInfo>();
    // -----------------------------------------------------------------

    [Header("Fallback (Legacy Level Data)")]
    public int carsToSpawn; 
}
```

---

## 4. Step 2 - New Script: GarageController.cs

**Create NEW FILE:** Assets/Script/GarageController.cs

```csharp
using System.Collections.Generic;
using UnityEngine;
using DG.Tweening;

/// Attached to the Garage prefab.
/// Controls the car queue, slide animation, and cleanup.
///
/// PREFAB SETUP:
///   GaragePrefab
///     - GarageController component
///     - BoxCollider (NOT trigger, tag = "garage" NOT "car")
///     - Child: SpawnPoint (Transform at mouth position, 1 unit forward)
public class GarageController : MonoBehaviour
{
    [Tooltip("World position where the frontmost car appears (mouth cell).")]
    public Transform spawnPoint;

    private Queue<CarMover> carQueue = new Queue<CarMover>();
    private CarMover frontCar;

    public Vector2Int bodyCell;
    public Vector2Int mouthCell;
    private Vector2Int facingDir;

    private bool isAnimating = false;

    private const float SlideInDuration  = 0.35f;
    private const float SlideInOffsetMul = 2.5f;

    // Called by SpawnCars after prefab is placed
    public void Initialize(
        Vector2Int body,
        Vector2Int mouth,
        Vector2Int facing,
        List<CarMover> cars)
    {
        bodyCell  = body;
        mouthCell = mouth;
        facingDir = facing;

        carQueue.Clear();
        foreach (var c in cars)
            carQueue.Enqueue(c);

        ShowNextCar(animate: false);
    }

    // Returns the car at the garage mouth (called by CarController to verify)
    public CarMover GetFrontCar() => frontCar;

    // Called by CarController when player clicks front car and path is clear
    public CarMover ReleaseFrontCar()
    {
        if (frontCar == null || isAnimating) return null;

        CarMover released = frontCar;
        frontCar = null;

        released.transform.SetParent(null);
        released.garageOwner = null;

        return released;
    }

    // Called by CarMover.DriveAway() OnComplete — slide next car in
    public void OnCarDeparted()
    {
        if (carQueue.Count > 0)
            ShowNextCar(animate: true);
        else
            EmptyGarage();
    }

    // Restores front car if no parking slot was available
    public void RestoreFrontCar(CarMover car)
    {
        frontCar = car;
        car.isParked = false;
        car.transform.position = spawnPoint != null
            ? spawnPoint.position
            : transform.position;
        car.transform.rotation = transform.rotation;

        if (SpawnCars.Instance != null)
            SpawnCars.Instance.SetSlotOccupation(mouthCell.x, mouthCell.y, true);
    }

    // Called by SpawnCars.ClearGarages() on level restart
    public void ForceEmpty()
    {
        if (frontCar != null)
        {
            frontCar.transform.SetParent(null);
            ObjectPool.Instance.AddToPool(frontCar.gameObject);
            frontCar = null;
        }

        while (carQueue.Count > 0)
        {
            CarMover car = carQueue.Dequeue();
            if (car != null)
            {
                car.transform.SetParent(null);
                if (car.gameObject.activeInHierarchy)
                    ObjectPool.Instance.AddToPool(car.gameObject);
            }
        }

        if (SpawnCars.Instance != null)
        {
            SpawnCars.Instance.SetSlotOccupation(bodyCell.x,  bodyCell.y,  false);
            SpawnCars.Instance.SetSlotOccupation(mouthCell.x, mouthCell.y, false);
            SpawnCars.Instance.UnregisterGarage(this);
        }

        ObjectPool.Instance.AddToPool(gameObject);
    }

    // INTERNAL
    private void ShowNextCar(bool animate)
    {
        if (carQueue.Count == 0)
        {
            EmptyGarage();
            return;
        }

        CarMover next = carQueue.Dequeue();
        next.transform.SetParent(transform);
        next.garageOwner = this;

        Vector3 mouthWorld = spawnPoint != null
            ? spawnPoint.position
            : transform.position + new Vector3(facingDir.x, 0, facingDir.y);

        Vector3 behindMouth = mouthWorld
            - new Vector3(facingDir.x, 0, facingDir.y) * SlideInOffsetMul;

        next.transform.rotation = transform.rotation;

        if (animate)
        {
            isAnimating = true;
            next.gameObject.SetActive(true);
            next.transform.position = behindMouth;

            next.transform.DOMove(mouthWorld, SlideInDuration)
                .SetEase(Ease.OutCubic)
                .OnComplete(() =>
                {
                    isAnimating = false;
                    frontCar = next;
                    if (SpawnCars.Instance != null)
                        SpawnCars.Instance.SetSlotOccupation(mouthCell.x, mouthCell.y, true);
                });
        }
        else
        {
            next.gameObject.SetActive(true);
            next.transform.position = mouthWorld;
            frontCar = next;
            if (SpawnCars.Instance != null)
                SpawnCars.Instance.SetSlotOccupation(mouthCell.x, mouthCell.y, true);
        }
    }

    private void EmptyGarage()
    {
        if (SpawnCars.Instance != null)
        {
            SpawnCars.Instance.SetSlotOccupation(bodyCell.x,  bodyCell.y,  false);
            SpawnCars.Instance.SetSlotOccupation(mouthCell.x, mouthCell.y, false);
            SpawnCars.Instance.UnregisterGarage(this);
        }

        ObjectPool.Instance.AddToPool(gameObject);
    }
}
```

---

## 5. Step 3 - Modify SpawnCars.cs

**File:** Assets/Script/SpawnCars.cs

### CHANGE A — Add these fields after the existing `[Header("Prefabs & References")]` block:

```csharp
[Header("Garage")]
public GameObject garagePrefab;

[HideInInspector]
public List<GarageController> activeGarages = new List<GarageController>();
```

### CHANGE B — Add SpawnGarages() and helpers at end of class (before last closing brace):

```csharp
// =====================================================
// GARAGE SPAWNING
// =====================================================

public void SpawnGarages()
{
    if (currentLevelData == null) return;
    if (currentLevelData.garageSpawns == null || currentLevelData.garageSpawns.Count == 0) return;
    if (garagePrefab == null)
    {
        Debug.LogWarning("SpawnCars: garagePrefab is not assigned!");
        return;
    }

    foreach (var info in currentLevelData.garageSpawns)
    {
        if (info.carCount <= 0) continue;

        Vector2Int facing = AngleToGridDir(info.rotationY);
        if (facing == Vector2Int.zero) facing = new Vector2Int(0, 1);

        Vector2Int bodyCell  = info.gridPosition;
        Vector2Int mouthCell = info.gridPosition + facing;

        GridSlot bodySlot  = GetSlotAt(bodyCell.x,  bodyCell.y);
        GridSlot mouthSlot = GetSlotAt(mouthCell.x, mouthCell.y);

        if (bodySlot == null || mouthSlot == null)
        {
            Debug.LogWarning("GarageSpawn: body or mouth cell out of grid at " + bodyCell);
            continue;
        }
        if (bodySlot.IsOccupied || mouthSlot.IsOccupied)
        {
            Debug.LogWarning("GarageSpawn: cells already occupied at " + bodyCell);
            continue;
        }

        SetSlotOccupation(bodyCell.x,  bodyCell.y,  true);
        SetSlotOccupation(mouthCell.x, mouthCell.y, true);

        Quaternion garageRot = Quaternion.Euler(0, info.rotationY, 0);
        GameObject garageGO  = ObjectPool.Instance.GetGarageFromPool(garagePrefab);
        if (garageGO == null)
        {
            garageGO = Instantiate(garagePrefab, bodySlot.WorldPosition, garageRot);
        }
        else
        {
            garageGO.transform.position = bodySlot.WorldPosition;
            garageGO.transform.rotation = garageRot;
            garageGO.SetActive(true);
        }

        GarageController gc = garageGO.GetComponent<GarageController>();
        if (gc == null)
        {
            Debug.LogError("GaragePrefab is missing GarageController component!");
            continue;
        }

        List<CarMover> queuedCars = new List<CarMover>();
        for (int i = 0; i < info.carCount; i++)
        {
            CarMover prefab = GetPrefabForSize(info.carSize);
            if (prefab == null) continue;

            CarMover car = ObjectPool.Instance.GetCarFromPool(prefab);
            if (car == null)
            {
                car = Instantiate(prefab);
                car.originalPrefab = prefab;
            }

            car.carSize = info.carSize;
            car.ResetCapacity();
            car.ResetEnum();
            car.gameObject.SetActive(false);
            car.arrow.enabled = true;
            car.totalPassengerTxt.enabled = false;

            spawnPassengers.TotalCarsSpawn.Add(car);
            queuedCars.Add(car);
        }

        gc.Initialize(bodyCell, mouthCell, facing, queuedCars);
        activeGarages.Add(gc);
    }
}

public void UnregisterGarage(GarageController gc)
{
    activeGarages.Remove(gc);
}

public void ClearGarages()
{
    for (int i = activeGarages.Count - 1; i >= 0; i--)
    {
        if (activeGarages[i] != null)
            activeGarages[i].ForceEmpty();
    }
    activeGarages.Clear();
}
```

### CHANGE C — Add SpawnGarages() call at end of SpawnGridAndCars():

Find the CLOSING BRACE of SpawnGridAndCars() — it is after the entire else block.
Add one line JUST BEFORE that closing brace:

```csharp
        // ... (end of else block for infinite/random levels)
    }

    SpawnGarages(); // ADD THIS LINE
}  // <-- this is the closing brace of SpawnGridAndCars()
```

---

## 6. Step 4 - Modify LevelManager.cs

**File:** Assets/Script/Level/LevelManager.cs

Find `ClearCurrentLevelData()` and add ONE LINE at the very start of the method body:

```csharp
public void ClearCurrentLevelData()
{
    SoundManager.Instance.StopHelicopterSound();
    ResetPowerUpFullState();

    // NEW LINE BELOW ----------------------------------------
    if (carSpawner != null) carSpawner.ClearGarages();
    // -------------------------------------------------------

    // 1. Clear Cars (existing code unchanged)
    if (spawnPassengers.TotalCarsSpawn != null)
    {
        // ... existing code ...
    }
    // ... rest of method unchanged ...
}
```

---

## 7. Step 5 - Modify CarController.cs

**File:** Assets/Script/CarController.cs

Replace the existing HandleCarSelection() method with this:

```csharp
void HandleCarSelection()
{
    Ray ray = _mainCamera.ScreenPointToRay(Input.mousePosition);
    RaycastHit hit;

    if (Physics.Raycast(ray, out hit))
    {
        if (hit.collider.CompareTag("car"))
        {
            // TUTORIAL CHECK
            if (TutorialManager.Instance != null && TutorialManager.Instance.IsTutorialActive())
            {
                GameObject targetTutorialCar = TutorialManager.Instance.GetTargetCarObject();
                if (hit.collider.gameObject != targetTutorialCar)
                    return;
                TutorialManager.Instance.OnCarClicked(hit.collider.gameObject);
            }

            CarMover clickedMover = hit.collider.GetComponent<CarMover>();
            if (clickedMover == null) return;

            // GARAGE LOGIC -----------------------------------
            if (clickedMover.garageOwner != null)
            {
                GarageController garage = clickedMover.garageOwner;

                if (garage.GetFrontCar() != clickedMover) return;

                Carout obstacleDetector = clickedMover.GetComponent<Carout>();
                bool blocked = obstacleDetector != null && obstacleDetector.CheckForBlockage();

                if (blocked)
                {
                    Debug.Log("Cannot move garage car: blocked!");
                    obstacleDetector.DoTackle();
                    SoundManager.Instance.PlaySound(SoundManager.SoundName.CarDash);
                    StartCoroutine(CrashJiggle());
                    return;
                }

                CarMover released = garage.ReleaseFrontCar();
                if (released == null) return;

                ParkingSlotManger emptySlot = parkingGameManager.FindEmptyParkingSlot();
                if (emptySlot != null)
                {
                    emptySlot.isReserved  = true;
                    emptySlot.incomingCar = released;
                    released.isParked     = true;
                    released.SetDestination(emptySlot.transform.position);
                }
                else
                {
                    Debug.Log("No empty slots! Garage car re-queued.");
                    released.transform.SetParent(garage.transform);
                    released.garageOwner = garage;
                    garage.RestoreFrontCar(released);
                }
                return;
            }
            // END GARAGE LOGIC --------------------------------

            // NORMAL CAR LOGIC
            if (clickedMover.isParked) return;

            Carout caroutDetector = hit.collider.GetComponent<Carout>();
            bool isBlocked = caroutDetector != null && caroutDetector.CheckForBlockage();

            if (isBlocked)
            {
                Debug.Log("Cannot move: Car in front!");
                caroutDetector.DoTackle();
                SoundManager.Instance.PlaySound(SoundManager.SoundName.CarDash);
                StartCoroutine(CrashJiggle());
                return;
            }

            ParkingSlotManger slot = parkingGameManager.FindEmptyParkingSlot();
            if (slot != null)
            {
                slot.isReserved  = true;
                slot.incomingCar = clickedMover;
                clickedMover.isParked = true;
                clickedMover.SetDestination(slot.transform.position);
            }
            else
            {
                Debug.Log("No empty slots available!");
            }
        }
    }
}
```

---

## 8. Step 6 - Modify CarMover.cs

**File:** Assets/Script/CarMover.cs

### CHANGE A — Add garageOwner field (add after the existing originalPrefab field):

```csharp
[HideInInspector]
public CarMover originalPrefab;

// NEW FIELD
[HideInInspector]
public GarageController garageOwner; // null if NOT inside a garage
```

### CHANGE B — Notify garage in DriveAway() OnComplete:

Find DriveAway(). Find the s.OnComplete lambda. Add 3 lines BEFORE the pool call:

```csharp
s.OnComplete(() =>
{
    // NEW: Notify garage so next car can slide out --------
    GarageController owner = garageOwner;
    garageOwner = null;
    if (owner != null) owner.OnCarDeparted();
    // -----------------------------------------------------

    ObjectPool.Instance.AddToPool(gameObject);
    isParked = false;
});
```

Note: We capture `garageOwner` into `owner` BEFORE nulling it. The lambda closes over `owner` not `garageOwner`, so it works correctly.

---

## 9. Step 7 - Modify ObjectPool.cs

**File:** Assets/Script/ObjectPool.cs

Add this method at the end of the class (before closing brace):

```csharp
// NEW: Garage pooling
public GameObject GetGarageFromPool(GameObject prefab)
{
    for (int i = pool.Count - 1; i >= 0; i--)
    {
        if (pool[i] == null) { pool.RemoveAt(i); continue; }
        if (!pool[i].activeInHierarchy && pool[i].name.Contains(prefab.name))
        {
            GameObject go = pool[i];
            pool.RemoveAt(i);
            return go;
        }
    }
    return null;
}
```

---

## 10. Step 8 - Modify LevelEditorWindow.cs

**File:** Assets/Script/Level/Editor/LevelEditorWindow.cs

### CHANGE A — Add new variables inside the class (near the top with other private fields):

```csharp
// NEW: Garage editor state
private bool editingGarageMode = false;
private Vector2Int selectedGarageCell = new Vector2Int(-1, -1);
private float garageRotation = 0f;
private CarSize garageCarSize = CarSize.Small;
private int garageCarCount = 2;
```

### CHANGE B — Add mode toolbar UI (add right after the Placement Mode block and before the "Draw Pattern Grid" label):

```csharp
EditorGUILayout.Space();
EditorGUILayout.BeginVertical("box");
GUILayout.Label("Editing Mode", EditorStyles.boldLabel);
EditorGUILayout.BeginHorizontal();
if (GUILayout.Toggle(!editingGarageMode, "Cars", "Button", GUILayout.Height(28)))
    editingGarageMode = false;
if (GUILayout.Toggle(editingGarageMode, "Garages", "Button", GUILayout.Height(28)))
    editingGarageMode = true;
EditorGUILayout.EndHorizontal();

if (editingGarageMode)
{
    EditorGUILayout.Space();
    GUILayout.Label("Garage Settings");
    int dirIndex = System.Array.IndexOf(directions, garageRotation);
    if (dirIndex < 0) dirIndex = 0;
    dirIndex = GUILayout.SelectionGrid(dirIndex, directionNames, 4);
    garageRotation  = directions[dirIndex];
    garageCarSize   = (CarSize)EditorGUILayout.EnumPopup("Car Size Inside", garageCarSize);
    garageCarCount  = EditorGUILayout.IntSlider("Car Count", garageCarCount, 1, 10);
    EditorGUILayout.HelpBox(
        "Click a grid cell to place/remove a garage.\n" +
        "Body = clicked cell. Mouth = 1 step in facing direction.\n" +
        "Orange = body cell. Bright orange = mouth cell.",
        MessageType.Info);
}
EditorGUILayout.EndVertical();
EditorGUILayout.Space();
```

### CHANGE C — In the grid drawing loop, add garage cell coloring.

Find this block inside the for-loops (after `Color originalBg = GUI.backgroundColor;`):

```csharp
// ADD BEFORE the existing "if (occupiedPreview[c, r])" block:
bool isGarageBody  = false;
bool isGarageMouth = false;
if (targetLevel.garageSpawns != null)
{
    foreach (var gs in targetLevel.garageSpawns)
    {
        if (gs.gridPosition == new Vector2Int(c, r))
            isGarageBody = true;

        Vector2Int gFacing = AngleToGridDirEditor(gs.rotationY);
        if (gs.gridPosition + gFacing == new Vector2Int(c, r))
            isGarageMouth = true;
    }
}

if (isGarageBody)
    GUI.backgroundColor = new Color(0.7f, 0.4f, 0.1f);
else if (isGarageMouth)
    GUI.backgroundColor = new Color(1.0f, 0.7f, 0.2f);
else if (occupiedPreview[c, r])  // <-- this "else if" was originally "if"
{
    // ... existing car coloring code ...
}
```

### CHANGE D — In the grid button click handler, add garage placement at the very top of the button block:

Find `if (GUI.Button(cellRect, label))` and add this at the very beginning of the block:

```csharp
if (GUI.Button(cellRect, label))
{
    // NEW: Garage mode click
    if (editingGarageMode)
    {
        Vector2Int clicked = new Vector2Int(c, r);
        GarageSpawnInfo existing = targetLevel.garageSpawns != null
            ? targetLevel.garageSpawns.Find(g => g.gridPosition == clicked)
            : null;

        if (existing != null)
        {
            targetLevel.garageSpawns.Remove(existing);
        }
        else
        {
            if (targetLevel.garageSpawns == null)
                targetLevel.garageSpawns = new System.Collections.Generic.List<GarageSpawnInfo>();
            targetLevel.garageSpawns.Add(new GarageSpawnInfo
            {
                gridPosition = clicked,
                rotationY    = garageRotation,
                carSize      = garageCarSize,
                carCount     = garageCarCount
            });
            selectedGarageCell = clicked;
        }
        EditorUtility.SetDirty(targetLevel);
        Repaint();
        // Note: do NOT put "return" here because this is inside a loop.
        // Use a goto or break approach — set a flag and skip normal processing:
    }
    else
    {
        // ... MOVE ALL EXISTING BUTTON CLICK CODE INSIDE THIS ELSE BLOCK ...
        label = "";
        if (occupiedPreview[c,r] && !drawnPattern[c,r]) { ... }
        if (removeMode) { ... }
        else if (!isFilled) { ... }
        else { ... }
        Repaint();
    }
}
```

IMPORTANT NOTE: Wrap the existing button-click code inside an `else` block tied to `editingGarageMode`.

### CHANGE E — Add garage list panel below the grid (before Save button):

```csharp
// ADD BEFORE the Save button section:
if (targetLevel.garageSpawns != null && targetLevel.garageSpawns.Count > 0)
{
    EditorGUILayout.Space();
    EditorGUILayout.BeginVertical("box");
    GUILayout.Label("Placed Garages", EditorStyles.boldLabel);
    for (int gi = 0; gi < targetLevel.garageSpawns.Count; gi++)
    {
        GarageSpawnInfo gs = targetLevel.garageSpawns[gi];
        EditorGUILayout.BeginHorizontal();
        GUILayout.Label(
            "[" + gi + "] Cell(" + gs.gridPosition.x + "," + gs.gridPosition.y + ")" +
            " Rot:" + gs.rotationY + "deg " + gs.carSize + " x" + gs.carCount);

        if (GUILayout.Button("Edit", GUILayout.Width(40)))
        {
            selectedGarageCell = gs.gridPosition;
            garageRotation  = gs.rotationY;
            garageCarSize   = gs.carSize;
            garageCarCount  = gs.carCount;
            editingGarageMode = true;
        }

        GUI.backgroundColor = Color.red;
        if (GUILayout.Button("X", GUILayout.Width(25)))
        {
            targetLevel.garageSpawns.RemoveAt(gi);
            EditorUtility.SetDirty(targetLevel);
            Repaint();
            break;
        }
        GUI.backgroundColor = Color.white;
        EditorGUILayout.EndHorizontal();
    }
    EditorGUILayout.EndVertical();
}
```

### CHANGE F — Reset garage state in LoadPatternFromLevel():

Add these 2 lines at the END of LoadPatternFromLevel():

```csharp
// ADD AT END of LoadPatternFromLevel():
selectedGarageCell = new Vector2Int(-1, -1);
editingGarageMode  = false;
```

---

## 11. Step 9 - Unity Setup (Prefab + Scene)

### Create Garage Prefab:
1. Right-click Project window -> Create Empty -> name it "GaragePrefab"
2. Add GarageController component
3. Add BoxCollider (NOT trigger) - set tag to "garage" NOT "car"
4. Create child GameObject "SpawnPoint" - position it 1 unit forward (0, 0, DotSize) in local space
5. Drag SpawnPoint into GarageController.SpawnPoint field
6. Add your garage visual mesh/model as additional children

### Assign in Scene:
1. Select SpawnCars GameObject in Hierarchy
2. Find new "Garage" header in Inspector
3. Drag GaragePrefab into "Garage Prefab" field

### Test:
1. Open Tools -> Smart Pattern Editor
2. Select a LevelData asset
3. Click "Garages" toggle
4. Set Direction=Right, CarSize=Small, Count=2
5. Click a grid cell (not on border row)
6. See dark orange = body, bright orange = mouth
7. Click GENERATE LEVEL LAYOUT -> Save Level Asset
8. Hit Play and test

---

## 12. How It All Works Together

```
LEVEL LOAD
SpawnCars.SpawnGridAndCars()
  -> spawns normal cars
  -> SpawnGarages()
       -> reads LevelData.garageSpawns
       -> for each garage: marks bodyCell + mouthCell as occupied
       -> instantiates GaragePrefab at bodyCell world position
       -> creates N CarMovers, SetActive(false), adds to TotalCarsSpawn
       -> GarageController.Initialize()
            -> enqueues all cars
            -> ShowNextCar(animate:false)
                 -> dequeues first car, SetActive(true), positions at SpawnPoint
                 -> frontCar = car[0]

PLAYER CLICKS FRONT CAR
CarController.HandleCarSelection()
  -> hit car has garageOwner != null
  -> garage.GetFrontCar() == clickedMover (confirmed front car)
  -> Carout.CheckForBlockage() -> not blocked
  -> garage.ReleaseFrontCar()
       -> frontCar = null, parent = null, garageOwner = null
  -> FindEmptyParkingSlot() -> found slot
  -> released.SetDestination(slot.position)

CAR DRIVES AND PARKS
CarMover.MoveRoutine() -> reaches slot -> isParked = true
CarMover.DriveAway() -> car drives off screen
  -> OnComplete lambda:
       -> owner (captured before null) = garage reference
       -> owner.OnCarDeparted()
            -> carQueue.Count > 0 -> ShowNextCar(animate:true)
                 -> DOMove next car from behind to SpawnPoint
                 -> frontCar = car[1]
  -> ObjectPool.Instance.AddToPool(this car)

LAST CAR DEPARTS
CarMover.DriveAway().OnComplete()
  -> owner.OnCarDeparted()
       -> carQueue.Count == 0 -> EmptyGarage()
            -> Free bodyCell + mouthCell
            -> UnregisterGarage()
            -> AddToPool(garageGO)
  -> Cells (bodyCell, mouthCell) are now FREE for other cars to path through
```

---

## 13. Quick Checklist Before Testing

- [ ] LevelData.cs - Added GarageSpawnInfo class + garageSpawns list
- [ ] GarageController.cs - Created as new script
- [ ] SpawnCars.cs - Added garagePrefab field, activeGarages list, SpawnGarages(), ClearGarages(), UnregisterGarage(), added SpawnGarages() call at end of SpawnGridAndCars()
- [ ] LevelManager.cs - Added carSpawner.ClearGarages() call in ClearCurrentLevelData()
- [ ] CarMover.cs - Added garageOwner field + OnCarDeparted notification in DriveAway()
- [ ] CarController.cs - Replaced HandleCarSelection() with garage-aware version
- [ ] ObjectPool.cs - Added GetGarageFromPool() method
- [ ] LevelEditorWindow.cs - Added garage mode toggle, grid coloring, click handler, list panel
- [ ] Prefab - Created GaragePrefab with GarageController + SpawnPoint child
- [ ] Inspector - Assigned GaragePrefab in SpawnCars component
- [ ] Test - Placed garage in Level Editor, generated layout, saved, tested in Play mode

---

## 14. Key Design Decisions

INACTIVE QUEUE CARS
Cars inside the garage (not the front car) are SetActive(false).
Their colliders are disabled automatically.
Raycasts CANNOT hit them. Only frontCar (active) can be clicked.
This requires ZERO extra code to enforce.

WIN CONDITION
Every garage car is added to spawnPassengers.TotalCarsSpawn at spawn time.
When each car parks -> DriveAway() -> pools it.
SpawnPassengers removes it from TotalCarsSpawn.
When count reaches 0 -> ParkingGameManager.CheckWinCondition() fires.
NO changes needed in ParkingGameManager or SpawnPassengers.

CELL BLOCKING
mouthCell stays IsOccupied=true while a car is there.
This means other cars cannot path through the mouth cell.
When the garage empties, both cells are freed -> cars can path through.

DO NOT place garage mouth cell on the grid border row.
The border row is the exit path used by CarMover.GeneratePath().
Keep garages at least 2 rows inside the grid boundary.
