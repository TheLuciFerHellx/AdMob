# ⚡ Power-Up Tutorial Popups — Complete Setup Guide

> **Levels 4, 6, 7** — Full-screen popup → click → icon jumps from center → flies to button → button unlocks

---

## 🎯 What This Does

| Level | Power-Up | Name | Description |
|-------|----------|------|-------------|
| **4** | fillSlotButton | **Quick Fill** | Instantly fills the first car waiting in the parking slot! |
| **6** | rearrangeButton | **Rearrange** | Fills all cars in parking slots by rearranging all passengers in the queue! |
| **7** | helicopterButton | **Helicopter** | A helicopter picks up the required car and drops it on the VIP slot! |

### Animation Flow
```
Level 4/6/7 loads
      ↓
Full-screen popup appears  (BG image + icon at CENTER-RIGHT + title + desc)
      ↓
Player taps/clicks anywhere on popup
      ↓
Popup hides instantly
      ↓
Icon copy appears at CENTER of screen
      ↓
Icon JUMPS (arc curve) → flies to its power-up button position
      ↓
Icon disappears → Power-Up Button ENABLES (lock hides, button interactable)
```

> **"Seen" flag:** Uses `PlayerPrefs` key — popup only shows once per power-up, even after restarts.

---

## 📁 STEP 1 — Create the Script

Create a new C# script at:
```
Assets/Script/Tutorial/PowerUpTutorialController.cs
```

Paste this **complete code**:

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class PowerUpTutorialController : MonoBehaviour
{
    public static PowerUpTutorialController Instance;

    // ─────────────────────────────────────────────
    // SHARED UI (one panel for all 3 tutorials)
    // ─────────────────────────────────────────────
    [Header("Shared Panel UI")]
    [Tooltip("Root panel GameObject — full screen, set inactive by default")]
    public GameObject popupPanel;

    [Tooltip("The full-screen background Image inside the panel")]
    public Image bgImage;

    [Tooltip("The power-up icon Image shown in the panel (anchored center-right)")]
    public Image iconImage;

    [Tooltip("Title label — e.g. 'Quick Fill'")]
    public TextMeshProUGUI titleText;

    [Tooltip("Description label — short sentence about the power-up")]
    public TextMeshProUGUI descText;

    [Tooltip("Separate Image used ONLY for the fly animation (set inactive by default)")]
    public Image flyingIconImage;

    // ─────────────────────────────────────────────
    // TUTORIAL 1 — LEVEL 4 : Quick Fill
    // ─────────────────────────────────────────────
    [Header("Tutorial 1 — Level 4: Quick Fill")]
    [Tooltip("Full-screen background sprite for this tutorial")]
    public Sprite t1_bgSprite;

    [Tooltip("Icon sprite for Quick Fill power-up")]
    public Sprite t1_iconSprite;

    [Tooltip("Drag the fillSlotButton's RectTransform here")]
    public RectTransform t1_targetButton;

    // ─────────────────────────────────────────────
    // TUTORIAL 2 — LEVEL 6 : Rearrange
    // ─────────────────────────────────────────────
    [Header("Tutorial 2 — Level 6: Rearrange")]
    [Tooltip("Full-screen background sprite for this tutorial")]
    public Sprite t2_bgSprite;

    [Tooltip("Icon sprite for Rearrange power-up")]
    public Sprite t2_iconSprite;

    [Tooltip("Drag the rearrangeButton's RectTransform here")]
    public RectTransform t2_targetButton;

    // ─────────────────────────────────────────────
    // TUTORIAL 3 — LEVEL 7 : Helicopter
    // ─────────────────────────────────────────────
    [Header("Tutorial 3 — Level 7: Helicopter")]
    [Tooltip("Full-screen background sprite for this tutorial")]
    public Sprite t3_bgSprite;

    [Tooltip("Icon sprite for Helicopter power-up")]
    public Sprite t3_iconSprite;

    [Tooltip("Drag the helicopterButton's RectTransform here")]
    public RectTransform t3_targetButton;

    // ─────────────────────────────────────────────
    // FLY ANIMATION SETTINGS
    // ─────────────────────────────────────────────
    [Header("Fly Animation Settings")]
    [Tooltip("How long the icon takes to fly to the button (seconds)")]
    public float flyDuration = 0.55f;

    [Tooltip("Ease type for the fly animation")]
    public Ease flyEase = Ease.InBack;

    [Tooltip("How high the arc goes during the jump (pixels)")]
    public float arcHeight = 200f;

    // ─────────────────────────────────────────────
    // PRIVATE
    // ─────────────────────────────────────────────
    private const string KEY_P1 = "pup_tut_4_seen";
    private const string KEY_P2 = "pup_tut_6_seen";
    private const string KEY_P3 = "pup_tut_7_seen";

    private int   _currentLevel;
    private RectTransform _flyingRect;

    // ─────────────────────────────────────────────

    void Awake()
    {
        if (Instance == null) Instance = this;
        else { Destroy(gameObject); return; }
    }

    void Start()
    {
        // Make sure popup is hidden at start
        if (popupPanel  != null) popupPanel.SetActive(false);
        if (flyingIconImage != null) flyingIconImage.gameObject.SetActive(false);

        _flyingRect = flyingIconImage != null ? flyingIconImage.GetComponent<RectTransform>() : null;
    }

    // ─────────────────────────────────────────────
    // CALLED BY LevelManager after level load
    // ─────────────────────────────────────────────
    public void TryShowTutorial(int level)
    {
        _currentLevel = level;

        if (level == 4 && PlayerPrefs.GetInt(KEY_P1, 0) == 0)
        {
            ShowPopup(t1_bgSprite, t1_iconSprite,
                      "Quick Fill",
                      "Instantly fills the first car\nwaiting in the parking slot!",
                      t1_targetButton);
        }
        else if (level == 6 && PlayerPrefs.GetInt(KEY_P2, 0) == 0)
        {
            ShowPopup(t2_bgSprite, t2_iconSprite,
                      "Rearrange",
                      "Fills all cars in parking slots by\nrearranging all passengers in the queue!",
                      t2_targetButton);
        }
        else if (level == 7 && PlayerPrefs.GetInt(KEY_P3, 0) == 0)
        {
            ShowPopup(t3_bgSprite, t3_iconSprite,
                      "Helicopter",
                      "A helicopter picks up the required car\nand drops it on the VIP slot!",
                      t3_targetButton);
        }
    }

    // ─────────────────────────────────────────────
    // SHOW THE POPUP
    // ─────────────────────────────────────────────
    private RectTransform _pendingTarget;
    private Sprite        _pendingIcon;

    private void ShowPopup(Sprite bg, Sprite icon, string title, string desc, RectTransform target)
    {
        // Store for use when player clicks
        _pendingTarget = target;
        _pendingIcon   = icon;

        // Set content
        if (bgImage   != null) bgImage.sprite   = bg;
        if (iconImage != null) iconImage.sprite  = icon;
        if (titleText != null) titleText.text    = title;
        if (descText  != null) descText.text     = desc;

        // Position icon at center-right of the panel
        RectTransform iconRect = iconImage.GetComponent<RectTransform>();
        iconRect.anchoredPosition = new Vector2(300f, 0f); // center-right

        // Show panel
        if (popupPanel != null) popupPanel.SetActive(true);
    }

    // ─────────────────────────────────────────────
    // CALLED BY BUTTON / onClick ON THE PANEL
    // ─────────────────────────────────────────────
    public void OnPopupClicked()
    {
        if (popupPanel == null || !popupPanel.activeSelf) return;

        // Hide the popup immediately
        popupPanel.SetActive(false);

        // Mark as seen
        if      (_currentLevel == 4) PlayerPrefs.SetInt(KEY_P1, 1);
        else if (_currentLevel == 6) PlayerPrefs.SetInt(KEY_P2, 1);
        else if (_currentLevel == 7) PlayerPrefs.SetInt(KEY_P3, 1);
        PlayerPrefs.Save();

        // Start the fly animation
        StartCoroutine(FlyIconToButton());
    }

    // ─────────────────────────────────────────────
    // FLY ANIMATION
    // ─────────────────────────────────────────────
    private IEnumerator FlyIconToButton()
    {
        if (_flyingRect == null || _pendingTarget == null)
        {
            // No fly object assigned — just unlock immediately
            UnlockCurrentPowerUp();
            yield break;
        }

        // Set flying icon sprite
        flyingIconImage.sprite = _pendingIcon;

        // Get canvas for coordinate conversion
        Canvas canvas = flyingIconImage.GetComponentInParent<Canvas>();
        if (canvas == null) { UnlockCurrentPowerUp(); yield break; }

        RectTransform canvasRect = canvas.GetComponent<RectTransform>();

        // ── START POSITION: Center of screen (0, 0 in canvas space)
        _flyingRect.anchoredPosition = Vector2.zero;
        flyingIconImage.gameObject.SetActive(true);

        // ── END POSITION: The button's position in canvas space
        Vector2 targetPos = GetCanvasPosition(_pendingTarget, canvasRect);

        // ── ARC JUMP using DOTween sequence
        Sequence seq = DOTween.Sequence();

        // X moves straight to target
        seq.Join(_flyingRect.DOAnchorPosX(targetPos.x, flyDuration).SetEase(Ease.InOutSine));

        // Y goes UP first (arc), then DOWN to target
        seq.Join(_flyingRect.DOAnchorPosY(arcHeight, flyDuration * 0.45f)
            .SetEase(Ease.OutQuad)
            .OnComplete(() =>
            {
                _flyingRect.DOAnchorPosY(targetPos.y, flyDuration * 0.55f)
                    .SetEase(flyEase);
            }));

        // Scale punch on land
        seq.OnComplete(() =>
        {
            flyingIconImage.transform
                .DOPunchScale(Vector3.one * 0.4f, 0.3f, 6, 0.5f)
                .OnComplete(() =>
                {
                    flyingIconImage.gameObject.SetActive(false);
                    UnlockCurrentPowerUp();
                });
        });

        yield return seq.WaitForCompletion();
    }

    // ─────────────────────────────────────────────
    // UNLOCK THE POWER-UP BUTTON AFTER ANIMATION
    // ─────────────────────────────────────────────
    private void UnlockCurrentPowerUp()
    {
        if (LevelManager.Instance != null)
        {
            LevelManager.Instance.UpdatePowerUpUnlocks();
        }
    }

    // ─────────────────────────────────────────────
    // HELPER — Convert a RectTransform world pos → canvas anchored pos
    // ─────────────────────────────────────────────
    private Vector2 GetCanvasPosition(RectTransform target, RectTransform canvasRect)
    {
        Vector3[] corners = new Vector3[4];
        target.GetWorldCorners(corners);
        Vector3 center = (corners[0] + corners[2]) / 2f;

        Canvas canvas = canvasRect.GetComponent<Canvas>();
        if (canvas.renderMode == RenderMode.ScreenSpaceOverlay)
        {
            // Direct screen pos
            RectTransformUtility.ScreenPointToLocalPointInRectangle(
                canvasRect, center, null, out Vector2 localPoint);
            return localPoint;
        }
        else
        {
            RectTransformUtility.ScreenPointToLocalPointInRectangle(
                canvasRect, center, canvas.worldCamera, out Vector2 localPoint);
            return localPoint;
        }
    }

    // ─────────────────────────────────────────────
    // UTILITY — Reset seen flags (for testing in Editor)
    // ─────────────────────────────────────────────
    [ContextMenu("DEV: Reset All Tutorial Seen Flags")]
    public void DEV_ResetSeenFlags()
    {
        PlayerPrefs.DeleteKey(KEY_P1);
        PlayerPrefs.DeleteKey(KEY_P2);
        PlayerPrefs.DeleteKey(KEY_P3);
        PlayerPrefs.Save();
        Debug.Log("[PowerUpTutorial] All seen flags reset. Tutorials will show again.");
    }
}
```

---

## 📝 STEP 2 — Modify LevelManager.cs

### 2a — Make `UpdatePowerUpUnlocks()` public

Find this line in [LevelManager.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-Grid/Car-OUT-jam-puzzle-Game-Grid/Assets/Script/Level/LevelManager.cs#L301):

```csharp
// BEFORE (line ~301):
private void UpdatePowerUpUnlocks()

// CHANGE TO:
public void UpdatePowerUpUnlocks()
```

### 2b — Call the tutorial at end of `LevelLoadSequence()`

Find the bottom of `LevelLoadSequence()` (around line 594–598):

```csharp
// EXISTING CODE (leave as is):
ParkingGameManager.Instance.isLevelLoading = false;

if (TutorialManager.Instance != null)
{
    TutorialManager.Instance.CheckStartTutorial();
}
```

**Add these lines RIGHT AFTER** the `TutorialManager` block:

```csharp
// ── NEW: Power-Up Tutorial ──
if (PowerUpTutorialController.Instance != null)
{
    int lvl = SaveData.Instance.CurrentSave.currentLevel;
    PowerUpTutorialController.Instance.TryShowTutorial(lvl);
}
```

---

## 🎨 STEP 3 — Build the UI in Unity Editor

### Panel Hierarchy

Build this under your **Game Canvas**:

```
Canvas (your existing game canvas)
 └── PowerUpTutorialPanel          ← new empty GameObject, add Image component
      │    [Full screen stretch, blocks raycasts, set INACTIVE by default]
      │    [Add a Button component here — OnClick calls OnPopupClicked()]
      │
      ├── BG_Image                  ← Image component, full screen stretch
      │    [This will show the background sprite]
      │
      ├── PopupCard                 ← empty GameObject with Image (card bg, optional)
      │    ├── Icon_Image           ← Image component ~180×180
      │    │    [anchored center-right: X=300, Y=0]
      │    ├── Title_Text           ← TextMeshProUGUI, big bold font ~48pt
      │    └── Desc_Text            ← TextMeshProUGUI, smaller font ~28pt
      │
      └── FlyingIcon_Image          ← Image component, ~120×120
           [Set INACTIVE by default — only used during fly animation]
           [Anchor: Center/Center, Pivot: 0.5/0.5]
```

### Rect Settings

| Object | Anchor | Pivot | Size |
|--------|--------|-------|------|
| PowerUpTutorialPanel | Stretch/Stretch | 0.5/0.5 | fills screen |
| BG_Image | Stretch/Stretch | 0.5/0.5 | fills screen |
| Icon_Image | Center/Center | 0.5/0.5 | 180×180 |
| Title_Text | Center/Center (above icon) | 0.5/0.5 | auto |
| Desc_Text | Center/Center (below icon) | 0.5/0.5 | auto |
| FlyingIcon_Image | Center/Center | 0.5/0.5 | 120×120 |

### Button on the Panel

- Add a **Button** component directly on `PowerUpTutorialPanel`
- Set its **Transition** to `None` (no visual change on click)
- Under **OnClick()** → drag the `PowerUpTutorialController` GameObject → call `OnPopupClicked()`

---

## 🔌 STEP 4 — Wire Up the Script in Inspector

### Add the script to a GameObject

1. Create an empty GameObject in your scene → name it **`PowerUpTutorialController`**
2. Add the `PowerUpTutorialController` script component to it

### Assign fields in Inspector

| Inspector Field | What to drag here |
|-----------------|-------------------|
| **Popup Panel** | The `PowerUpTutorialPanel` GameObject |
| **Bg Image** | The `BG_Image` Image component |
| **Icon Image** | The `Icon_Image` Image component inside PopupCard |
| **Title Text** | The `Title_Text` TextMeshProUGUI |
| **Desc Text** | The `Desc_Text` TextMeshProUGUI |
| **Flying Icon Image** | The `FlyingIcon_Image` Image component |
| **T1 Bg Sprite** | Your background sprite for Level 4 tutorial |
| **T1 Icon Sprite** | Your Quick Fill power-up icon sprite |
| **T1 Target Button** | Drag `fillSlotButton`'s **RectTransform** |
| **T2 Bg Sprite** | Your background sprite for Level 6 tutorial |
| **T2 Icon Sprite** | Your Rearrange power-up icon sprite |
| **T2 Target Button** | Drag `rearrangeButton`'s **RectTransform** |
| **T3 Bg Sprite** | Your background sprite for Level 7 tutorial |
| **T3 Icon Sprite** | Your Helicopter power-up icon sprite |
| **T3 Target Button** | Drag `helicopterButton`'s **RectTransform** |
| **Fly Duration** | `0.55` (default, adjust to taste) |
| **Fly Ease** | `InBack` (default — gives a nice snap) |
| **Arc Height** | `200` (how high the icon jumps) |

---

## 🧪 STEP 5 — Test It

### Quick Test Without Playing Through Levels

In the Unity **Inspector** while the game is NOT running:
1. Select your `PowerUpTutorialController` GameObject
2. Right-click the script component → **"DEV: Reset All Tutorial Seen Flags"**
3. This clears PlayerPrefs so tutorials show again

Then in your **SaveData** JSON file, temporarily set `"currentLevel": 4` to test Level 4 popup.

### Test Checklist

- [ ] Level 4 loads → popup appears with BG + icon at center-right + "Quick Fill" title + desc
- [ ] Click anywhere → popup hides, icon appears at screen center, jumps in arc → lands on fillSlotButton
- [ ] fillSlotButton becomes interactable (lock hides)
- [ ] Replay Level 4 → popup does NOT appear again
- [ ] Level 6 loads → same flow with Rearrange
- [ ] Level 7 loads → same flow with Helicopter
- [ ] Levels 1–3, 5 → no popup, normal lock/unlock behavior

---

## ⚠️ Notes

> **Canvas Render Mode**: The `GetCanvasPosition` helper handles both `ScreenSpaceOverlay` and `ScreenSpaceCamera`. Make sure your Canvas is one of these (not `WorldSpace`).

> **DOTween**: Your project already uses `DG.Tweening` (confirmed in existing scripts), so no extra import needed.

> **PowerUpTutorialPanel Sort Order**: Make sure this panel's Canvas Group or sorting layer is on TOP of all other UI so it blocks input correctly.

> **If no sprites yet**: Leave the `Sprite` fields empty in Inspector — the popup will still show (just with no image), so you can test the flow immediately.
