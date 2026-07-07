# Fix Double Level Loading Issue

This plan describes how to resolve the bug where the level is loaded twice if a user triggers a restart or level completion while a level is already loading.

## Proposed Solution

We will implement two layers of defense to completely resolve this issue:
1. **UI Layer (Visual & Interaction Block)**: We will move the pause button off-screen (or hide/disable it) when a level starts loading, and return it to its original position once the level has finished loading. This matches your requested approach.
2. **Logic Layer (Guard Checks)**: We will add checks to `LevelManager.cs` and `PausePopup.cs` to ignore restart, next level, or pause requests if the game state indicates the level is already loading (`isLevelLoading == true`).

---

## Proposed Changes

### [UIManager Component]

#### [MODIFY] [PausePopup.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/UIManager/PausePopup.cs)

Add a guard check at the beginning of `ShowPausePopup()` to prevent the pause screen from opening if the level is loading.

```diff
     public void ShowPausePopup()
     {
-        if (ParkingGameManager.Instance.IsTransitioning)
+        if (ParkingGameManager.Instance.IsTransitioning || ParkingGameManager.Instance.isLevelLoading)
             return;
 
         SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);
```

---

### [Level & Game Loop Component]

#### [MODIFY] [LevelManager.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Script/Level/LevelManager.cs)

Modify `LevelManager.cs` to add a reference to the Pause Button, save its original position, move it off-screen on level load, bring it back when load finishes, and add safety checks in `RestartLevel` and `CompleteLevel`.

**Note:** We offer two options for moving the pause button—**Instant Shift** and **Smooth DOTween Transition** (recommended for premium visual polish).

##### 1. Variable Declarations
Add fields to store the Pause Button reference and its original position:

```csharp
    [Header("UI Controls")]
    public RectTransform pauseButton;
    private Vector2 originalPauseButtonPos;
    private bool isPauseButtonPosSaved = false;
```

##### 2. Prevent Double Actions in `RestartLevel` and `CompleteLevel`
Add guards to return early if `isLevelLoading` is `true`:

```diff
     public void CompleteLevel()
     {
+        if (ParkingGameManager.Instance != null && ParkingGameManager.Instance.isLevelLoading)
+        {
+            Debug.LogWarning("Level is already loading. Ignoring CompleteLevel request.");
+            return;
+        }
+
         currentLevelIndex++;
 
         SaveData.Instance.SetLevel(currentLevelIndex + 1);
```

```diff
     #region ResetLevel
     public void RestartLevel()
     {
+        if (ParkingGameManager.Instance != null && ParkingGameManager.Instance.isLevelLoading)
+        {
+            Debug.LogWarning("Level is already loading. Ignoring RestartLevel request.");
+            return;
+        }
+
         ClearCurrentLevelData();
 
         if (TutorialManager.Instance != null)
```

##### 3. Move Pause Button Off-screen when `LoadLevel` starts
Inside `LoadLevel()`:

* **Option A: Instant Shift (Simple)**
  ```csharp
  public void LoadLevel()
  {
      rewardButton.ResetForNewLevel();
      
      // Move Pause Button off-screen
      if (pauseButton != null)
      {
          if (!isPauseButtonPosSaved)
          {
              originalPauseButtonPos = pauseButton.anchoredPosition;
              isPauseButtonPosSaved = true;
          }
          // Move the button 1000 units up to hide it off-screen
          pauseButton.anchoredPosition = new Vector2(originalPauseButtonPos.x, originalPauseButtonPos.y + 1000f);
      }
      ...
  ```

* **Option B: Smooth Slide Transition using DOTween (Recommended for Premium Design)**
  Since the project already uses `DG.Tweening` (DOTween), we can smoothly animate the pause button sliding off-screen and back:
  ```csharp
  public void LoadLevel()
  {
      rewardButton.ResetForNewLevel();
      
      // Smoothly animate Pause Button off-screen
      if (pauseButton != null)
      {
          if (!isPauseButtonPosSaved)
          {
              originalPauseButtonPos = pauseButton.anchoredPosition;
              isPauseButtonPosSaved = true;
          }
          // Slide up out of view
          pauseButton.DOKill();
          pauseButton.DOAnchorPos(new Vector2(originalPauseButtonPos.x, originalPauseButtonPos.y + 1000f), 0.3f)
              .SetUpdate(true); // Works even if timeScale is 0
      }
      ...
  ```

##### 4. Bring Pause Button back in `LevelLoadSequence` when loading finishes
At the end of the `LevelLoadSequence()` coroutine:

* **Option A: Instant Restore (Simple)**
  ```csharp
          // 3. Now let the game check for the win condition
          ParkingGameManager.Instance.isLevelLoading = false;

          // Put the pause button back to its original position
          if (pauseButton != null && isPauseButtonPosSaved)
          {
              pauseButton.anchoredPosition = originalPauseButtonPos;
          }

          if (TutorialManager.Instance != null)
          ...
  ```

* **Option B: Smooth Slide In (Recommended for Premium Design)**
  ```csharp
          // 3. Now let the game check for the win condition
          ParkingGameManager.Instance.isLevelLoading = false;

          // Slide the pause button back into view
          if (pauseButton != null && isPauseButtonPosSaved)
          {
              pauseButton.DOKill();
              pauseButton.DOAnchorPos(originalPauseButtonPos, 0.3f)
                  .SetUpdate(true);
          }

          if (TutorialManager.Instance != null)
          ...
  ```

---

## Unity Editor Setup Instructions

After compiling the code changes, perform the following steps in the Unity Editor:

1. Open the main scene: [SampleScene.unity](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Car-OUT-jam-puzzle-Game-CustomeLevelEditor/Assets/Scenes/SampleScene.unity).
2. Locate the game object containing the `LevelManager` script (typically a manager object in the hierarchy).
3. Find the new **Pause Button** property field in the inspector under the `LevelManager` component.
4. Drag and drop the **Pause Button** RectTransform from your UI hierarchy into this field.

---

## Verification Plan

### Manual Verification
1. **Spam Restart Button**: Play the game, open the Pause menu, and click "Restart" rapidly several times. Confirm that the level only loads once and there are no duplicate spawned cars/passengers.
2. **Win and Restart**: Complete a level, and as the new level begins its loading sequence, verify that:
   - The Pause button is moved off-screen (or animated off-screen).
   - Once loading completes, the Pause button transitions smoothly back to its original position.
3. **State Guard Verification**: Monitor the Unity Console during loading. Verify that any accidental clicks are logged and safely ignored without starting concurrent load sequences.
