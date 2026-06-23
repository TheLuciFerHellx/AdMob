# 🎰 Spin Wheel Freeze Fix Guide

When the Spin Wheel panel is open, the game sets `Time.timeScale = 0` (pausing the game).
Because the popup card's DOTween scaling and the starburst rotation were running on standard time scale, they froze.

Here is the exact script and the changes to resolve the freeze.

---

## 📁 The Script: `SpinWheelRewardPopup.cs`

Create this script in your project. It uses `.SetUpdate(true)` on DOTween animations to run on unscaled time, and `Time.unscaledDeltaTime` for rotation:

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class SpinWheelRewardPopup : MonoBehaviour
{
    public static SpinWheelRewardPopup Instance;

    [Header("Panel")]
    public GameObject popupPanel;
    public Image overlayImage;

    [Header("Rotating Background")]
    public RectTransform rotatingBG;
    public float rotateSpeed = 60f;

    [Header("Reward Card")]
    public RectTransform rewardCard;
    public Image rewardIcon;
    public TextMeshProUGUI quantityText;
    public TextMeshProUGUI rewardNameText;

    [Header("Reward Icons")]
    public Sprite coinSprite;
    public Sprite fillSlotSprite;
    public Sprite rearrangeSprite;
    public Sprite helicopterSprite;

    [Header("Animation Settings")]
    public float popInDuration = 0.35f;
    public Ease popInEase = Ease.OutBack;
    public float popOutDuration = 0.2f;

    private bool _isShowing = false;
    private bool _rotating  = false;

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
        // ── Fix: Use unscaledDeltaTime so it rotates when paused ──
        if (_rotating && rotatingBG != null)
        {
            rotatingBG.Rotate(0f, 0f, rotateSpeed * Time.unscaledDeltaTime);
        }
    }

    public void ShowReward(SpinWheelController.RewardType rewardType, int amount, string rewardName)
    {
        if (_isShowing) return;
        _isShowing = true;

        if (rewardIcon != null) rewardIcon.sprite = GetSprite(rewardType);
        if (quantityText != null) quantityText.text = "+" + amount;
        if (rewardNameText != null) rewardNameText.text = rewardName;

        if (popupPanel != null) popupPanel.SetActive(true);
        _rotating = true;

        // ── Fix: Added SetUpdate(true) so it scales open when timeScale = 0 ──
        if (rewardCard != null)
        {
            rewardCard.localScale = Vector3.zero;
            rewardCard
                .DOScale(Vector3.one, popInDuration)
                .SetEase(popInEase)
                .SetUpdate(true);
        }
    }

    public void OnPopupClicked()
    {
        if (!_isShowing) return;
        _rotating = false;

        // ── Fix: Added SetUpdate(true) so it scales closed when timeScale = 0 ──
        if (rewardCard != null)
        {
            rewardCard
                .DOScale(Vector3.zero, popOutDuration)
                .SetEase(Ease.InBack)
                .SetUpdate(true)
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

## 📝 Integration into `SpinWheelController.cs`

Add these two changes to your existing `SpinWheelController.cs` to trigger the popup:

### 1. Add the field at the top:
```csharp
[SerializeField] private SpinWheelRewardPopup rewardPopup;
```

### 2. Update `GiveReward` at the bottom:
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

    // ── Call the reward popup here ──
    if (rewardPopup != null)
    {
        rewardPopup.ShowReward(reward.rewardType, reward.amount, reward.rewardName);
    }
}
```

---

## 📝 Modify `UIPopupAnimator.cs`

In your `UIPopupAnimator.cs` open method, make sure the background fade also runs on unscaled time:

```csharp
// Find this line in Open():
if (background != null)
    background.DOFade(0.6f, duration).SetEase(Ease.Linear);

// Change it to:
if (background != null)
    background.DOFade(0.6f, duration).SetEase(Ease.Linear).SetUpdate(true);
```
