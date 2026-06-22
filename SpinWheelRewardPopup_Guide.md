# 🎰 Spin Wheel Reward Popup — Complete Setup Guide

> When the spin wheel finishes, a popup appears **center screen** showing  
> what the player won — with a **rotating background**, **reward icon**, **big quantity number**, and **dismiss on click**.

---

## 🎯 What This Does

```
Wheel stops spinning
       ↓
Reward is given to SaveData (coins/powerup added)
       ↓
Popup appears at CENTER of screen
  [Rotating radiant BG image behind everything]
  [Reward icon in middle]
  [Big quantity text  e.g.  "+3"  or  "+40 🪙"]
  [Reward name label e.g. "Coins" / "Quick Fill"]
       ↓
Player taps/clicks anywhere
       ↓
Popup hides (scale-down animation)
       ↓
Spin button re-enables
```

---

## 📁 STEP 1 — Create the Script

Create a new C# script at:
```
Assets/Script/SpinWheel/SpinWheelRewardPopup.cs
```

Paste this **complete code**:

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

/// <summary>
/// Shows a center-screen popup after the spin wheel lands,
/// displaying what the player won. Click anywhere to dismiss.
/// </summary>
public class SpinWheelRewardPopup : MonoBehaviour
{
    public static SpinWheelRewardPopup Instance;

    // ─────────────────────────────────────────────────────────────
    // INSPECTOR FIELDS
    // ─────────────────────────────────────────────────────────────

    [Header("Panel")]
    [Tooltip("Root panel — full screen, blocks raycasts. Set INACTIVE by default.")]
    public GameObject popupPanel;

    [Tooltip("Full-screen dimmed overlay image (dark semi-transparent color)")]
    public Image overlayImage;

    [Header("Rotating Background")]
    [Tooltip("The radiant / starburst image that rotates behind the card")]
    public RectTransform rotatingBG;

    [Tooltip("Speed of the BG rotation in degrees per second")]
    public float rotateSpeed = 60f;

    [Header("Reward Card")]
    [Tooltip("The card GameObject that pops in with scale animation")]
    public RectTransform rewardCard;

    [Tooltip("Icon image — shows coin sprite or powerup icon")]
    public Image rewardIcon;

    [Tooltip("Big quantity label — e.g.  +40  or  +3")]
    public TextMeshProUGUI quantityText;

    [Tooltip("Reward name label — e.g.  Coins  /  Quick Fill  /  Helicopter")]
    public TextMeshProUGUI rewardNameText;

    [Header("Reward Icons — drag your sprites here")]
    [Tooltip("Coin icon sprite")]
    public Sprite coinSprite;

    [Tooltip("Quick Fill power-up icon sprite")]
    public Sprite fillSlotSprite;

    [Tooltip("Rearrange power-up icon sprite")]
    public Sprite rearrangeSprite;

    [Tooltip("Helicopter power-up icon sprite")]
    public Sprite helicopterSprite;

    [Header("Animation Settings")]
    [Tooltip("Card pop-in duration (seconds)")]
    public float popInDuration = 0.35f;

    [Tooltip("Card pop-in ease")]
    public Ease popInEase = Ease.OutBack;

    [Tooltip("Card pop-out duration on dismiss")]
    public float popOutDuration = 0.2f;

    // ─────────────────────────────────────────────────────────────
    // PRIVATE STATE
    // ─────────────────────────────────────────────────────────────

    private bool _isShowing = false;
    private bool _rotating  = false;

    // ─────────────────────────────────────────────────────────────

    void Awake()
    {
        if (Instance == null) Instance = this;
        else { Destroy(gameObject); return; }
    }

    void Start()
    {
        if (popupPanel != null) popupPanel.SetActive(false);
    }

    void Update()
    {
        // Continuously rotate the background starburst
        if (_rotating && rotatingBG != null)
        {
            rotatingBG.Rotate(0f, 0f, rotateSpeed * Time.deltaTime);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // PUBLIC — Call this from SpinWheelController after GiveReward
    // ─────────────────────────────────────────────────────────────

    /// <summary>
    /// Call this right after giving the reward.
    /// rewardType: 0=Coins, 1=FillSlot, 2=Rearrange, 3=Helicopter
    /// amount: the number won (e.g. 40, 1, 2)
    /// rewardName: display label (e.g. "Coins", "Quick Fill")
    /// </summary>
    public void ShowReward(SpinWheelController.RewardType rewardType, int amount, string rewardName)
    {
        if (_isShowing) return;
        _isShowing = true;

        // ── Set icon ──────────────────────────────────────────
        if (rewardIcon != null)
        {
            rewardIcon.sprite = GetSprite(rewardType);
        }

        // ── Set quantity text ─────────────────────────────────
        if (quantityText != null)
        {
            quantityText.text = "+" + amount;
        }

        // ── Set name label ────────────────────────────────────
        if (rewardNameText != null)
        {
            rewardNameText.text = rewardName;
        }

        // ── Show panel ────────────────────────────────────────
        if (popupPanel != null) popupPanel.SetActive(true);

        // ── Start BG rotation ─────────────────────────────────
        _rotating = true;

        // ── Animate card pop in ───────────────────────────────
        if (rewardCard != null)
        {
            rewardCard.localScale = Vector3.zero;
            rewardCard
                .DOScale(Vector3.one, popInDuration)
                .SetEase(popInEase);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // CALLED BY BUTTON onClick on the panel (any click dismisses)
    // ─────────────────────────────────────────────────────────────

    public void OnPopupClicked()
    {
        if (!_isShowing) return;

        _rotating  = false;

        // Pop-out scale animation then hide
        if (rewardCard != null)
        {
            rewardCard
                .DOScale(Vector3.zero, popOutDuration)
                .SetEase(Ease.InBack)
                .OnComplete(() =>
                {
                    _isShowing = false;
                    if (popupPanel != null) popupPanel.SetActive(false);
                });
        }
        else
        {
            _isShowing = false;
            if (popupPanel != null) popupPanel.SetActive(false);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // HELPER — Pick the right sprite per reward type
    // ─────────────────────────────────────────────────────────────

    private Sprite GetSprite(SpinWheelController.RewardType type)
    {
        switch (type)
        {
            case SpinWheelController.RewardType.Coins:              return coinSprite;
            case SpinWheelController.RewardType.FillSlotPowerup:    return fillSlotSprite;
            case SpinWheelController.RewardType.RearrangePowerup:   return rearrangeSprite;
            case SpinWheelController.RewardType.HelicopterPowerup:  return helicopterSprite;
            default:                                                 return coinSprite;
        }
    }
}
```

---

## 📝 STEP 2 — Edit SpinWheelController.cs (2 small changes)

### Change 1 — Add a field for the popup reference

Inside the `[Header("UI")]` block, add one line:

```csharp
[Header("UI")]
[SerializeField] private Transform wheelTransform;
[SerializeField] private Transform pointer;
[SerializeField] public Button spinButton;
[SerializeField] private GameObject spinPanel;
[SerializeField] private TextMeshProUGUI spinButtonText;

// ── ADD THIS LINE ──
[SerializeField] private SpinWheelRewardPopup rewardPopup; // ← drag SpinWheelRewardPopup here
```

### Change 2 — Call ShowReward at the end of `GiveReward()`

Find the existing `GiveReward` method (around line 413) and add **2 lines** at the very end:

```csharp
private void GiveReward(WheelReward reward)
{
    switch (reward.rewardType)
    {
        case RewardType.Coins:
            SaveData.Instance.AddCurrency(reward.amount);
            break;
        case RewardType.FillSlotPowerup:
            SaveData.Instance.SetFillSlotCount(
                SaveData.Instance.CurrentSave.fillSlotCount + reward.amount);
            break;
        case RewardType.RearrangePowerup:
            SaveData.Instance.SetRearrangeCount(
                SaveData.Instance.CurrentSave.rearrangeCount + reward.amount);
            break;
        case RewardType.HelicopterPowerup:
            SaveData.Instance.SetHelicopterCount(
                SaveData.Instance.CurrentSave.helicopterCount + reward.amount);
            break;
    }
    SaveData.Instance.Save();
    PowerUps.Instance.LoadFromSave();

    // ── ADD THESE 2 LINES ────────────────────────────────────────
    if (rewardPopup != null)
        rewardPopup.ShowReward(reward.rewardType, reward.amount, reward.rewardName);
    // ────────────────────────────────────────────────────────────
}
```

> That's it for code changes — 1 new field, 2 lines. Everything else stays exactly the same.

---

## 🎨 STEP 3 — Build the UI in Unity Editor

### Hierarchy (build inside your Spin Wheel Canvas)

```
YourSpinCanvas  (or main Game Canvas)
 └── SpinRewardPopupPanel               ← new empty GO, add Image + Button
      │   [Full screen stretch, Raycast Target ON, set INACTIVE by default]
      │   [Button component → OnClick() → SpinWheelRewardPopup.OnPopupClicked()]
      │
      ├── OverlayImage                  ← Image, full screen, dark color (black α 150)
      │
      ├── RotatingBG                    ← Image, your starburst/radiant sprite
      │   [Anchor: center/center, ~600×600, Raycast OFF]
      │
      └── RewardCard                    ← Image, centered card ~450×550
           │   [Anchor: center/center, nice card background]
           │
           ├── RewardIcon_Image         ← Image, ~160×160, centered upper area
           │
           ├── QuantityText             ← TextMeshProUGUI, BIG bold e.g. "+40"
           │   [Center below icon, font size ~80–100]
           │
           └── RewardNameText           ← TextMeshProUGUI, smaller label
               [e.g. "Coins" or "Quick Fill", font size ~36]
```

### Rect Settings

| Object | Anchor | Size | Notes |
|--------|--------|------|-------|
| SpinRewardPopupPanel | Stretch all | fills screen | Raycast Target ON |
| OverlayImage | Stretch all | fills screen | Color: black, Alpha ~150 |
| RotatingBG | Center/Center | 600×600 | Raycast Target OFF |
| RewardCard | Center/Center | 450×550 | Nice card bg sprite |
| RewardIcon_Image | Center/Center | 160×160 | Upper part of card |
| QuantityText | Center/Center | auto | Font ~80, bold |
| RewardNameText | Center/Center | auto | Font ~36 |

---

## 🔌 STEP 4 — Wire Up Inspector Fields

### On `SpinWheelRewardPopup` component

| Inspector Field | What to drag here |
|-----------------|-------------------|
| **Popup Panel** | `SpinRewardPopupPanel` GameObject |
| **Overlay Image** | `OverlayImage` Image component |
| **Rotating BG** | `RotatingBG` RectTransform |
| **Rotate Speed** | `60` (degrees/sec, adjust to taste) |
| **Reward Card** | `RewardCard` RectTransform |
| **Reward Icon** | `RewardIcon_Image` Image component |
| **Quantity Text** | `QuantityText` TextMeshProUGUI |
| **Reward Name Text** | `RewardNameText` TextMeshProUGUI |
| **Coin Sprite** | Your coin icon sprite |
| **Fill Slot Sprite** | Your Quick Fill icon sprite |
| **Rearrange Sprite** | Your Rearrange icon sprite |
| **Helicopter Sprite** | Your Helicopter icon sprite |
| **Pop In Duration** | `0.35` |
| **Pop In Ease** | `OutBack` |
| **Pop Out Duration** | `0.2` |

### On `SpinWheelController` component

| Inspector Field | What to drag here |
|-----------------|-------------------|
| **Reward Popup** | `SpinWheelRewardPopup` component (from any GO in scene) |

### Button on the Panel

- Add a **Button** component on `SpinRewardPopupPanel`
- Set **Transition** → `None`
- **OnClick()** → drag `SpinWheelRewardPopup` GO → select `OnPopupClicked()`

---

## 🧪 STEP 5 — Test Checklist

- [ ] Spin the wheel → wheel stops → popup appears center screen
- [ ] Rotating starburst BG animates continuously behind the card
- [ ] Correct icon shows (coin for coins, correct powerup icon)
- [ ] Quantity shows correct number e.g. `+20` or `+2`
- [ ] Name shows correctly e.g. `Coins`, `Quick Fill`, `Rearrange +2`
- [ ] Tap anywhere → card scales down → popup hides
- [ ] Spin button re-enables after popup (it already re-enables in DoSpin's OnComplete)
- [ ] Spin again → same flow works again

---

## ⚠️ Notes

> **Spin button re-enable timing**: Currently `SpinWheelController` re-enables the spin button inside `DoSpin()`'s `OnComplete` callback (before `GiveReward` is called). The popup appears on top but the button is already re-enabled underneath. This is fine — the popup blocks taps until dismissed. No extra code needed.

> **DOTween**: Already used in your project — no extra import needed.

> **`rewardName` field**: Your `WheelReward` already has a `rewardName` string (e.g. `"10 Coins"`, `"Fill +2"`, `"Helicopter +1"`). The popup uses this directly — so whatever name you type in the Inspector list is what players see.

> **If you want `rewardName` to show differently**: For example showing `"Quick Fill"` instead of `"Fill +1"`, you can change the `rewardName` values in your rewards list in `SpinWheelController`'s Inspector.
