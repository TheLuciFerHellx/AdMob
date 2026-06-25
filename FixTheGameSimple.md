# 🎮 Car-OUT Puzzle Game — Complete Fix Guide

> **How to use this guide:** Every fix tells you **exactly which file to open**, **which lines to find**, and **what to replace/add/remove**. Follow each section in order.

---

## ✅ FIX 1 — Level Complete Popup / Reset / Car Doubling Bug

### 🔍 Root Cause Analysis

There are **three separate bugs** combined:

1. **Level Complete shows after reset/exit** → The coroutine `showLevelCompletePopup()` uses `WaitForSecondsRealtime(1f)` which ignores `Time.timeScale = 0` and keeps running even after a reset is triggered. DOTween `DelayedCall` and `DOScale` also keep running.
2. **Buttons (Adv/Next) don't appear** → `ShowButtonsSequentially()` is called from `AnimateStar(star1, ...)` — but after a mid-game reset the popup panel is re-enabled, stars re-animate, and the `DOVirtual.DelayedCall(2f, ...)` for NextButton was killed by DOTween's scene reset. So it fires but the 2-second delayed call orphans.
3. **Cars double on reset** → `ClearCurrentLevelData()` is called **twice**: once directly inside `RestartLevel()` → `ClearCurrentLevelData()` and then again inside `LevelLoadSequence()` which `LoadLevel()` calls → `ClearCurrentLevelData()` on line 521. This runs the clear loop twice, pooling the same objects a second time = duplicated cars.

---

### 🔧 Step-by-Step Fixes

---

#### STEP 1A — Fix `ParkingGameManager.cs`

**File:** `Assets/Script/ParkingGameManager.cs`

**Find this field block near the top (around line 20–35):**
```csharp
private bool gameOver = false;
public bool isLevelLoading = true;
```

**ADD one new private field directly below `gameOver`:**
```csharp
private bool gameOver = false;
private bool _levelCompleteTriggered = false;   // ← ADD THIS
public bool isLevelLoading = true;
```

---

**Find `CheckWinCondition()` method (line 132):**
```csharp
public void CheckWinCondition()
{
    if (gameOver || isLevelLoading)
        return;

    if (waitingPassengers.Count == 0)
    {
        gameOver = true;
        isLevelLoading = true;

        SoundManager.Instance.PlaySound(SoundManager.SoundName.LevelComplete);

        Debug.Log("Saved Currency: " + SaveData.Instance.CurrentSave.currency);

        StartCoroutine(showLevelCompletePopup());
    }
}
```

**REPLACE the entire method with this:**
```csharp
public void CheckWinCondition()
{
    if (gameOver || isLevelLoading || _levelCompleteTriggered)
        return;

    if (waitingPassengers.Count == 0)
    {
        gameOver = true;
        isLevelLoading = true;
        _levelCompleteTriggered = true;   // Guard: prevents double-fire

        SoundManager.Instance.PlaySound(SoundManager.SoundName.LevelComplete);

        Debug.Log("Saved Currency: " + SaveData.Instance.CurrentSave.currency);

        StartCoroutine(showLevelCompletePopup());
    }
}
```

---

**Find `ResetForNewLevel()` method (line 216):**
```csharp
public void ResetForNewLevel()
{
    keepStreakThisGameOver = false;

    gameOver = false;
    isLevelLoading = true; // Lock the check until spawning is done
    waitingPassengers.Clear(); // Clear any leftovers from the previous level

    // Added this line to reset combo when loading a new level/restarting
    FloatingTextManager.Instance?.ResetCombo(); 
    StartLevelTimer();
}
```

**REPLACE the entire method with this:**
```csharp
public void ResetForNewLevel()
{
    keepStreakThisGameOver = false;

    // ✅ Kill all running coroutines on this manager (stops showLevelCompletePopup mid-flight)
    StopAllCoroutines();

    // ✅ Kill all DOTween tweens owned by this GameObject (stops any DelayedCall, DOScale etc.)
    DG.Tweening.DOTween.Kill(this);

    // ✅ Reset all guard flags
    gameOver = false;
    _levelCompleteTriggered = false;
    isLevelLoading = true;

    waitingPassengers.Clear();
    isProcessingQueue = false;

    FloatingTextManager.Instance?.ResetCombo();
    StartLevelTimer();
}
```

---

#### STEP 1B — Fix `LevelManager.cs` — Remove Duplicate `ClearCurrentLevelData()` Call

**File:** `Assets/Script/Level/LevelManager.cs`

**Find `LevelLoadSequence()` coroutine (starting around line 518):**

```csharp
IEnumerator LevelLoadSequence()
{
    // 1. IMPORTANT: Reset the game state BEFORE starting spawning
    ClearCurrentLevelData();      // ← THIS IS THE DUPLICATE — REMOVE IT
     
    yield return new WaitForSeconds(0.5f);
    spawnPassengers.TotalCarsSpawn.Clear();
    ...
```

**REMOVE just the `ClearCurrentLevelData();` line and ALSO remove the VIP slot reset block that follows it inside `LevelLoadSequence` (because `ClearCurrentLevelData` already handles both).** The corrected method should look like this:

```csharp
IEnumerator LevelLoadSequence()
{
    // ClearCurrentLevelData is called by RestartLevel/LoadLevel BEFORE this coroutine.
    // Do NOT call it again here — that causes car doubling.

    yield return new WaitForSeconds(0.5f);
    spawnPassengers.TotalCarsSpawn.Clear();

    yield return new WaitForSeconds(0.5f);

    // Spawn Cars
    carSpawner.SpawnCarAsPerLevelNeed();
    
    yield return new WaitForSeconds(0.2f);

    // Spawn Passengers
    spawnPassengers.PassengerToSpawnBasedOnCarSpawn();

    yield return new WaitForEndOfFrame();

    // Now let the game check for the win condition
    ParkingGameManager.Instance.isLevelLoading = false;
}
```

> **⚠️ Important:** Also remove the second VIP slot block that was inside `LevelLoadSequence` (lines ~527–536 in the original). The block starting with:
> ```csharp
> // Reset VIP slot so it is deactivated on new levels
> if (PowerUps.Instance != null && PowerUps.Instance.vipSlot != null)
> ```
> **Delete that whole if-block from `LevelLoadSequence`** because `ClearCurrentLevelData()` already does it.

---

#### STEP 1C — Fix `LevelManager.cs` — `RestartLevel()` Must Call `ClearCurrentLevelData()` First

**Find `RestartLevel()` method (around line 369):**
```csharp
public void RestartLevel()
{
    ClearCurrentLevelData();
    // 4. Reload Level (Index remains the same)
    LoadLevel();
}
```

This is correct! BUT `LoadLevel()` internally calls `LevelLoadSequence()` which was calling `ClearCurrentLevelData()` again. After you fix Step 1B above, this will be fine.

Also **add `Time.timeScale = 1f;`** at the very top of `RestartLevel()` to unfreeze the game if it was paused during Game Over:

```csharp
public void RestartLevel()
{
    Time.timeScale = 1f;          // ← ADD THIS — unfreeze if coming from Game Over/Pause
    ClearCurrentLevelData();
    LoadLevel();
}
```

---

#### STEP 1D — Fix `LevelCompletePopup.cs` — Hide popup immediately on restart

**File:** `Assets/Script/UIManager/LevelCompletePopup.cs`

The `Awake()` already hides the buttons:
```csharp
void Awake()
{
    if(AdvButton != null && NextButton != null)
    {
        AdvButton.SetActive(false);
        NextButton.SetActive(false);
    }
}
```

**ADD an `OnDisable()` method** right after `Awake()` to kill all DOTween tweens on this popup when it's closed/disabled:

```csharp
void OnDisable()
{
    // Kill all DOTween tweens on stars and buttons when popup closes
    DG.Tweening.DOTween.Kill(transform);
    if (star1 != null) DG.Tweening.DOTween.Kill(star1.transform);
    if (star2 != null) DG.Tweening.DOTween.Kill(star2.transform);
    if (star3 != null) DG.Tweening.DOTween.Kill(star3.transform);
    if (AdvButton != null)
    {
        DG.Tweening.DOTween.Kill(AdvButton.transform);
        AdvButton.SetActive(false);
    }
    if (NextButton != null)
    {
        DG.Tweening.DOTween.Kill(NextButton.transform);
        NextButton.SetActive(false);
    }
}
```

---

#### STEP 1E — Fix `PausePopup.cs` — Reset button properly restarts the level

**File:** `Assets/Script/UIManager/PausePopup.cs`

Currently `PausePopup` has no Restart/Reset button. If you have a Reset button wired in your Pause panel in the scene, create a method in `PausePopup.cs` (or wherever the reset button calls). Add this method to `PausePopup.cs`:

```csharp
public void RestartFromPause()
{
    SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);

    // 1. Reset time FIRST (game is paused with timeScale=0)
    Time.timeScale = 1f;

    // 2. Close the pause popup
    UIPopupManager.Instance.ClosePopup();

    // 3. Restart the level (this will also call ParkingGameManager.ResetForNewLevel())
    LevelManager.Instance.RestartLevel();
}
```

> **In Unity Inspector:** Wire your Pause popup's "Restart" button `OnClick` event to this new `RestartFromPause()` method.

---

## ✅ FIX 2 — Shop: Prototype Free-Purchase Mode

### 🔍 Root Cause

`ShopManager.TryPurchase()` checks `playerCoins < cost` and blocks if not enough coins.  
`ShopItemCard.RefreshUI()` sets `buyButton.interactable = (playerCoins >= coinCost)` which greys out buttons.

---

### 🔧 Fix `ShopManager.cs`

**File:** `Assets/Script/Shop/ShopManager.cs`

**Find the `TryPurchase()` method (line 86). Find this coin-check block:**
```csharp
// ── Not enough coins → shake the card and bail ──────────────────
if (playerCoins < cost)
{
    Debug.LogWarning($"[ShopManager] Need {cost} coins but player only has {playerCoins}.");
    card.PlayInsufficientFeedback();
    return false;
}

// ── ATOMIC: modify CurrentSave directly, call Save() ONCE at end ─
// Step 1: deduct cost
SaveData.Instance.CurrentSave.currency -= cost;
```

**REPLACE those lines with this (skip deduction AND the coin check):**
```csharp
// ── PROTOTYPE MODE: All purchases are FREE — no coin check, no deduction ──
// Remove or comment the lines below to restore paid mode later:
// if (playerCoins < cost)
// {
//     card.PlayInsufficientFeedback();
//     return false;
// }
// SaveData.Instance.CurrentSave.currency -= cost;
```

So the full method now starts like:
```csharp
public bool TryPurchase(ShopItemCard card)
{
    if (SaveData.Instance == null)
    {
        Debug.LogError("[ShopManager] SaveData.Instance is null! Cannot purchase.");
        return false;
    }

    // PROTOTYPE: No cost check, no deduction. Just give rewards.

    // Step 1 (SKIPPED in prototype): deduct cost
    // SaveData.Instance.CurrentSave.currency -= card.coinCost;

    // Step 2: add coin reward (if any)
    if (card.coinsReward > 0)
        SaveData.Instance.CurrentSave.currency += card.coinsReward;

    // Step 3: add powerup rewards (if any)
    foreach (var reward in card.powerupRewards)
    {
        switch (reward.powerupType)
        {
            case ShopItemCard.PowerupType.FillSlot:
                SaveData.Instance.CurrentSave.fillSlotCount += reward.amount;
                break;
            case ShopItemCard.PowerupType.Rearrange:
                SaveData.Instance.CurrentSave.rearrangeCount += reward.amount;
                break;
            case ShopItemCard.PowerupType.Helicopter:
                SaveData.Instance.CurrentSave.helicopterCount += reward.amount;
                break;
        }
    }

    // Step 4: Save
    SaveData.Instance.Save();

    // Step 5: sync in-game HUD
    if (PowerUps.Instance != null)
        PowerUps.Instance.LoadFromSave();

    Debug.Log($"[ShopManager] PROTOTYPE Purchase OK: {card.itemName}");
    return true;
}
```

---

### 🔧 Fix `ShopItemCard.cs` — Always Keep Button Active

**File:** `Assets/Script/Shop/ShopItemCard.cs`

**Find `RefreshUI()` method (line 144). Find this block:**
```csharp
// Buy button: interactable only if player has enough coins
if (buyButton != null && SaveData.Instance != null)
{
    int playerCoins = SaveData.Instance.CurrentSave.currency;
    buyButton.interactable = (playerCoins >= coinCost);
}
```

**REPLACE with:**
```csharp
// PROTOTYPE MODE: Button is always interactable (no coin check needed)
if (buyButton != null)
    buyButton.interactable = true;
```

---

## ✅ FIX 3 — Daily Reward: No Miss-Reset, Watch Ad for Missed Days, UI Juice Animation

### 🔍 What changes

| Before | After |
|---|---|
| Missing a day resets streak to Day 1 | Missing days does NOT reset — they stay "missable" |
| No animation on current day card | Pulsing scale animation on today's card |
| Past cards look identical | Past unclaimed cards show a 🎬 "Watch Ad" indicator |
| Claimed cards show `claimedUI` | Same, but also colour-coded |

---

### 🔧 Step 3A — Fix `DailyRewardSystem.cs` — Remove Hard Reset on Miss

**File:** `Assets/Script/DailyReward/DailyRewardSystem.cs`

**Find the active `CheckDailyReward()` method (line 208). Find this entire block:**
```csharp
// Missed a day -> reset streak
if (daysPassed > 1)
{
    SaveData.Instance.CurrentSave.dailyRewardIndex = 0;

    for (int i = 0; i < 7; i++)
    {
        SaveData.Instance.CurrentSave.dailyRewardClaimed[i] = false;
    }

    Debug.Log("Streak Broken - Reset To Day 1");
}
// Exactly next day -> advance
else if (daysPassed == 1)
{
    int nextDay =
        (SaveData.Instance.GetDailyRewardIndex() + 1) % 7;

    SaveData.Instance.CurrentSave.dailyRewardIndex = nextDay;

    Debug.Log("Advanced To Day " + (nextDay + 1));
}
```

**REPLACE the entire if/else with this:**
```csharp
// Whether player missed 1 day or many days — we still advance the index.
// Missed rewards are NOT lost, they are shown as "Watch Ad to Claim".
if (daysPassed >= 1)
{
    // Advance by exactly 1 each day (cap at day 7, then wrap to 0)
    int nextDay = (SaveData.Instance.GetDailyRewardIndex() + 1) % 7;

    // If we hit day 7 (index 0 again), reset all claimed flags for a new cycle
    if (nextDay == 0)
    {
        for (int i = 0; i < 7; i++)
            SaveData.Instance.CurrentSave.dailyRewardClaimed[i] = false;

        Debug.Log("[DailyReward] Full 7-day cycle complete. Resetting.");
    }

    SaveData.Instance.CurrentSave.dailyRewardIndex = nextDay;
    Debug.Log("[DailyReward] Advanced To Day " + (nextDay + 1));
}
```

---

### 🔧 Step 3B — Add `missedRewardAdClaimed` to `SaveData.cs`

**File:** `Assets/Script/SaveData/SaveData.cs`

**Find `GameSaveData` class (line 7). Find:**
```csharp
public bool[] dailyRewardClaimed = new bool[7];//for daily Reward Check
```

**ADD one new field directly below it:**
```csharp
public bool[] dailyRewardClaimed = new bool[7]; // for daily Reward Check
public bool[] dailyRewardAdClaimed = new bool[7]; // missed rewards claimed via Ad
```

---

### 🔧 Step 3C — Rewrite `DailyRewardCard.cs`

**File:** `Assets/Script/DailyReward/DailyRewardCard.cs`

> This is the biggest change. **Replace the entire file content** with the following:

```csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class DailyRewardCard : MonoBehaviour
{
    public int cardIndex; // 0–6
    public int coinReward;

    [Header("UI References")]
    public Button button;
    public GameObject lockedUI;
    public GameObject claimedUI;
    public TextMeshProUGUI rewardText;

    [Header("Watch Ad Button (for missed days)")]
    public Button watchAdButton;       // A separate button on the card — assign in Inspector
    public TextMeshProUGUI watchAdBtnText; // Text on that button

    [Header("Juice Animation Settings")]
    public float pulseDuration = 0.6f;
    public float pulseScale = 1.12f;

    private Tween _pulseTween;

    void OnEnable()
    {
        UpdateUI();
    }

    void OnDisable()
    {
        // Stop animation when card goes off-screen
        _pulseTween?.Kill();
        transform.localScale = Vector3.one;
    }

    // ────────────────────────────────────────────────
    //  CLAIM TODAY'S REWARD (normal click)
    // ────────────────────────────────────────────────
    public void OnClickCard()
    {
        if (SaveData.Instance == null) return;

        int currentDay = SaveData.Instance.GetDailyRewardIndex();

        if (cardIndex != currentDay) return;
        if (SaveData.Instance.HasClaimedToday()) return;

        GiveReward();

        SaveData.Instance.CurrentSave.dailyRewardClaimed[cardIndex] = true;
        SaveData.Instance.SetLastRewardDate(System.DateTime.UtcNow.ToString("yyyy-MM-dd"));
        SaveData.Instance.Save();

        UpdateAllCards();
        Debug.Log("[DailyReward] Claimed day " + (cardIndex + 1));
    }

    // ────────────────────────────────────────────────
    //  CLAIM MISSED REWARD VIA AD
    // ────────────────────────────────────────────────
    public void OnClickWatchAd()
    {
        if (SaveData.Instance == null) return;

        int currentDay = SaveData.Instance.GetDailyRewardIndex();

        // Only allow missed cards (past days, not today, not already claimed)
        bool isMissed = IsMissedCard(currentDay);
        if (!isMissed) return;

        // Already claimed this missed day via ad
        bool[] adClaimed = SaveData.Instance.CurrentSave.dailyRewardAdClaimed;
        if (adClaimed != null && cardIndex < adClaimed.Length && adClaimed[cardIndex]) return;

        // Show rewarded ad
        if (AdMobManager.Instance == null)
        {
            Debug.LogWarning("[DailyReward] AdMobManager not found!");
            return;
        }

        AdMobManager.Instance.ShowRewardedAd(() =>
        {
            // Ad completed → give reward
            GiveReward();

            SaveData.Instance.CurrentSave.dailyRewardAdClaimed[cardIndex] = true;
            SaveData.Instance.Save();

            UpdateAllCards();
            Debug.Log("[DailyReward] Missed day " + (cardIndex + 1) + " claimed via Ad.");
        });
    }

    // ────────────────────────────────────────────────
    //  GIVE COIN REWARD
    // ────────────────────────────────────────────────
    private void GiveReward()
    {
        SaveData.Instance.CurrentSave.currency += coinReward;
    }

    // ────────────────────────────────────────────────
    //  IS THIS CARD A MISSED (PAST, UNCLAIMED) CARD?
    // ────────────────────────────────────────────────
    private bool IsMissedCard(int currentDay)
    {
        // "Missed" = before today's index AND not already claimed normally
        if (cardIndex >= currentDay) return false; // Future/today cards are NOT missed

        bool[] claimed = SaveData.Instance.CurrentSave.dailyRewardClaimed;
        bool alreadyClaimed =
            claimed != null && cardIndex < claimed.Length && claimed[cardIndex];

        bool[] adClaimed = SaveData.Instance.CurrentSave.dailyRewardAdClaimed;
        bool alreadyAdClaimed =
            adClaimed != null && cardIndex < adClaimed.Length && adClaimed[cardIndex];

        return !alreadyClaimed && !alreadyAdClaimed;
    }

    // ────────────────────────────────────────────────
    //  UPDATE THIS CARD'S VISUAL STATE
    // ────────────────────────────────────────────────
    public void UpdateUI()
    {
        if (SaveData.Instance == null) return;

        int currentDay = SaveData.Instance.GetDailyRewardIndex();
        bool isToday = (cardIndex == currentDay);
        bool isFuture = (cardIndex > currentDay);

        bool[] claimed = SaveData.Instance.CurrentSave.dailyRewardClaimed;
        bool isClaimed = claimed != null && cardIndex < claimed.Length && claimed[cardIndex];

        bool[] adClaimed = SaveData.Instance.CurrentSave.dailyRewardAdClaimed;
        bool isAdClaimed = adClaimed != null && cardIndex < adClaimed.Length && adClaimed[cardIndex];

        bool claimedToday = SaveData.Instance.HasClaimedToday();
        bool isMissed = IsMissedCard(currentDay);

        // ── Reward text ──
        if (rewardText != null)
            rewardText.text = coinReward.ToString();

        // ── Locked overlay (future days) ──
        if (lockedUI != null)
            lockedUI.SetActive(isFuture);

        // ── Claimed overlay ──
        if (claimedUI != null)
            claimedUI.SetActive(isClaimed || isAdClaimed);

        // ── Main claim button ──
        if (button != null)
            button.interactable = isToday && !claimedToday;

        // ── Watch Ad button (missed days only) ──
        if (watchAdButton != null)
        {
            bool showAdBtn = isMissed; // only show for unclaimed missed days
            watchAdButton.gameObject.SetActive(showAdBtn);

            if (watchAdBtnText != null)
                watchAdBtnText.text = "📺 Watch Ad";
        }

        // ── JUICE ANIMATION on today's unclaimed card ──
        _pulseTween?.Kill();
        transform.localScale = Vector3.one;

        if (isToday && !claimedToday)
        {
            // Looping pulse to draw attention
            _pulseTween = transform
                .DOScale(pulseScale, pulseDuration)
                .SetLoops(-1, LoopType.Yoyo)
                .SetEase(Ease.InOutSine)
                .SetUpdate(true); // works even with timeScale=0
        }

        // ── Colour tint: dim past/future cards, highlight today ──
        var cardImage = GetComponent<Image>();
        if (cardImage != null)
        {
            if (isToday && !claimedToday)
                cardImage.color = new Color(1f, 1f, 1f, 1f);      // Full bright
            else if (isClaimed || isAdClaimed)
                cardImage.color = new Color(0.6f, 1f, 0.6f, 1f);  // Soft green = claimed
            else if (isMissed)
                cardImage.color = new Color(1f, 0.85f, 0.4f, 1f); // Yellow = missed (watchable)
            else
                cardImage.color = new Color(0.6f, 0.6f, 0.6f, 1f); // Grey = locked future
        }
    }

    void UpdateAllCards()
    {
        DailyRewardCard[] cards = FindObjectsOfType<DailyRewardCard>();
        foreach (var c in cards)
            c.UpdateUI();
    }
}
```

---

### 🔧 Step 3D — Unity Inspector: Add the Watch Ad button

For each `DailyRewardCard` GameObject in your scene:

1. **Add a new Button** as a child of the card (name it `WatchAdButton`).
2. Set its text to `📺 Watch Ad`.
3. In the `DailyRewardCard` component in the Inspector, drag this new button into the **`Watch Ad Button`** slot and drag the button's text into **`Watch Ad Btn Text`**.
4. Wire the button's `OnClick()` → the card's `OnClickWatchAd()` method.
5. The script will auto-show/hide this button based on state.

---

## ✅ FIX 4 — Spin Wheel: First Spin Free → Watch Ad → Spin Flow

### 🔍 Desired Behaviour

| State | Button Text | What Happens on Click |
|---|---|---|
| First time today | **FREE** | Spin for free, then → state 2 |
| After free spin | **WATCH AD TO SPIN** | Shows rewarded ad |
| After ad watched OR cancelled | **SPIN** | One more spin, then → state 2 again |
| After that spin | **WATCH ADS** | Loops: show ad → spin |

The "free spin" resets **every day** (one free spin per day).

---

### 🔧 Step 4A — Add `freeSpinUsedDate` to `SaveData.cs`

**File:** `Assets/Script/SaveData/SaveData.cs`

**Find `GameSaveData` class. Find:**
```csharp
public bool[] dailyRewardAdClaimed = new bool[7]; // missed rewards claimed via Ad
```

**ADD directly below:**
```csharp
public string freeSpinUsedDate = ""; // Tracks when the free daily spin was used
```

---

### 🔧 Step 4B — Rewrite `SpinWheelController.cs`

**File:** `Assets/Script/SpinWheel/SpinWheelController.cs`

> **Replace the entire file** with the code below:

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class SpinWheelController : MonoBehaviour
{
    public enum RewardType
    {
        Coins,
        FillSlotPowerup,
        RearrangePowerup,
        HelicopterPowerup
    }

    [System.Serializable]
    public class WheelReward
    {
        public string rewardName;
        public RewardType rewardType;
        public int amount;
        public float weight;
    }

    // ── Spin state machine ──────────────────────────────────────────
    private enum SpinState
    {
        FreeAvailable,   // First spin of the day → button shows "FREE"
        WatchAdToSpin,   // Free spin used → button shows "WATCH AD TO SPIN"
        ReadyToSpin      // Ad watched/cancelled → button shows "SPIN"
    }

    [Header("UI")]
    [SerializeField] private Transform wheelTransform;
    [SerializeField] private Transform pointer;
    [SerializeField] public Button spinButton;
    [SerializeField] private GameObject spinPanel;
    [SerializeField] private TextMeshProUGUI spinButtonText; // ← ASSIGN the TMP text on your spin button

    [Header("Spin Settings")]
    [SerializeField] private float spinDuration = 4.5f;
    [SerializeField] private int spinCost = 10; // kept for reference, not used as cost anymore

    [Header("Rewards (10 Sectors)")]
    public List<WheelReward> rewards = new List<WheelReward>()
    {
        new WheelReward { rewardName="10 Coins",      rewardType=RewardType.Coins,             amount=10,  weight=25f },
        new WheelReward { rewardName="20 Coins",      rewardType=RewardType.Coins,             amount=20,  weight=20f },
        new WheelReward { rewardName="40 Coins",      rewardType=RewardType.Coins,             amount=40,  weight=15f },
        new WheelReward { rewardName="60 Coins",      rewardType=RewardType.Coins,             amount=60,  weight=5f  },
        new WheelReward { rewardName="Fill +1",       rewardType=RewardType.FillSlotPowerup,   amount=1,   weight=12f },
        new WheelReward { rewardName="Fill +2",       rewardType=RewardType.FillSlotPowerup,   amount=2,   weight=6f  },
        new WheelReward { rewardName="Rearrange +1",  rewardType=RewardType.RearrangePowerup,  amount=1,   weight=10f },
        new WheelReward { rewardName="Rearrange +2",  rewardType=RewardType.RearrangePowerup,  amount=2,   weight=4f  },
        new WheelReward { rewardName="Helicopter +1", rewardType=RewardType.HelicopterPowerup, amount=1,   weight=2f  },
        new WheelReward { rewardName="Helicopter +2", rewardType=RewardType.HelicopterPowerup, amount=2,   weight=1f  }
    };

    private bool isSpinning;
    private SpinState _state;

    // ──────────────────────────────────────────────────────────────
    private void Start()
    {
        spinButton.onClick.AddListener(OnSpinButtonClicked);
        RefreshButtonState();
    }

    private void OnEnable()
    {
        // Refresh every time the panel opens
        RefreshButtonState();
    }

    // ── Open / Close ───────────────────────────────────────────────
    public void OpenWheel()
    {
        spinPanel.SetActive(true);
        RefreshButtonState();
    }

    public void CloseWheel()
    {
        if (isSpinning) return;
        spinPanel.SetActive(false);
    }

    // ── Button label & state ───────────────────────────────────────
    private void RefreshButtonState()
    {
        string today = System.DateTime.UtcNow.ToString("yyyy-MM-dd");
        bool freeSpinUsedToday =
            SaveData.Instance != null &&
            SaveData.Instance.CurrentSave.freeSpinUsedDate == today;

        if (!freeSpinUsedToday)
        {
            _state = SpinState.FreeAvailable;
        }
        else if (_state != SpinState.ReadyToSpin)
        {
            // Default to WatchAd after free spin has been used
            _state = SpinState.WatchAdToSpin;
        }
        // If state is already ReadyToSpin (ad was just watched), keep it

        UpdateButtonLabel();
    }

    private void UpdateButtonLabel()
    {
        if (spinButtonText == null) return;

        switch (_state)
        {
            case SpinState.FreeAvailable:
                spinButtonText.text = "FREE";
                break;
            case SpinState.WatchAdToSpin:
                spinButtonText.text = "WATCH AD TO SPIN";
                break;
            case SpinState.ReadyToSpin:
                spinButtonText.text = "SPIN";
                break;
        }
    }

    // ── Main button handler ────────────────────────────────────────
    private void OnSpinButtonClicked()
    {
        if (isSpinning) return;

        switch (_state)
        {
            case SpinState.FreeAvailable:
                // Mark free spin as used for today
                string today = System.DateTime.UtcNow.ToString("yyyy-MM-dd");
                SaveData.Instance.CurrentSave.freeSpinUsedDate = today;
                SaveData.Instance.Save();
                DoSpin();
                break;

            case SpinState.WatchAdToSpin:
                // Show rewarded ad. On complete OR on cancel → set state to ReadyToSpin
                if (AdMobManager.Instance == null)
                {
                    Debug.LogWarning("[SpinWheel] AdMobManager not found.");
                    return;
                }
                spinButton.interactable = false;
                spinButtonText.text = "Loading Ad...";

                AdMobManager.Instance.ShowRewardedAd(() =>
                {
                    // Ad completed successfully
                    _state = SpinState.ReadyToSpin;
                    UpdateButtonLabel();
                    spinButton.interactable = true;
                });

                // Also handle ad cancelled/closed without reward:
                // AdMob doesn't have a direct cancel callback in all SDK versions,
                // so we use a fallback: the button reverts to SPIN after a short time
                // even if they didn't finish the ad (simulated here with a delayed call).
                // ↓ Remove this DOVirtual block if your AdMobManager has a cancel callback.
                DG.Tweening.DOVirtual.DelayedCall(1.5f, () =>
                {
                    if (!isSpinning && _state == SpinState.WatchAdToSpin)
                    {
                        // Ad was dismissed without completing — still allow spin
                        _state = SpinState.ReadyToSpin;
                        UpdateButtonLabel();
                        spinButton.interactable = true;
                    }
                });
                break;

            case SpinState.ReadyToSpin:
                DoSpin();
                break;
        }
    }

    // ── Core spin logic ────────────────────────────────────────────
    private void DoSpin()
    {
        isSpinning = true;
        spinButton.interactable = false;

        SoundManager.Instance.PlaySound(SoundManager.SoundName.SpinSlotMachineReel);

        int index = GetWeightedIndex();
        WheelReward reward = rewards[index];

        float anglePerSlice = 360f / rewards.Count;
        float targetAngle = index * anglePerSlice;
        float fullRotations = 360f * 6f;
        float finalAngle = fullRotations + (360f - targetAngle);

        wheelTransform.localEulerAngles = Vector3.zero;

        wheelTransform
            .DORotate(new Vector3(0, 0, finalAngle), spinDuration, RotateMode.FastBeyond360)
            .SetEase(Ease.OutCubic)
            .SetUpdate(true)
            .OnComplete(() =>
            {
                SoundManager.Instance.PlaySound(SoundManager.SoundName.SpinWin);
                GiveReward(reward);
                isSpinning = false;

                // After spin, always go back to "WATCH AD TO SPIN"
                _state = SpinState.WatchAdToSpin;
                UpdateButtonLabel();
                spinButton.interactable = true;

                Debug.Log("[SpinWheel] Won: " + reward.rewardName);
            });
    }

    // ── Weighted random ───────────────────────────────────────────
    private int GetWeightedIndex()
    {
        float total = 0f;
        foreach (var r in rewards) total += r.weight;
        float rand = Random.Range(0, total);
        float current = 0f;
        for (int i = 0; i < rewards.Count; i++)
        {
            current += rewards[i].weight;
            if (rand <= current) return i;
        }
        return 0;
    }

    // ── Give reward ───────────────────────────────────────────────
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
    }
}
```

---

### 🔧 Step 4C — Unity Inspector: Assign `spinButtonText`

1. Open the **SpinWheel panel** in your scene hierarchy.
2. Select the `SpinWheelController` component.
3. In the Inspector, find the **`Spin Button Text`** field.
4. Drag the **TMP Text component** that is inside your Spin Button into that slot.

---

## 📋 Summary Checklist

| Fix | File(s) Changed | Done? |
|---|---|---|
| 1A. Add `_levelCompleteTriggered` guard | `ParkingGameManager.cs` | ☐ |
| 1A. Update `CheckWinCondition()` | `ParkingGameManager.cs` | ☐ |
| 1A. Update `ResetForNewLevel()` | `ParkingGameManager.cs` | ☐ |
| 1B. Remove duplicate `ClearCurrentLevelData()` from `LevelLoadSequence` | `LevelManager.cs` | ☐ |
| 1C. Add `Time.timeScale = 1f` to `RestartLevel()` | `LevelManager.cs` | ☐ |
| 1D. Add `OnDisable()` DOTween kill to popup | `LevelCompletePopup.cs` | ☐ |
| 1E. Add `RestartFromPause()` method | `PausePopup.cs` | ☐ |
| 2A. Remove coin check in `TryPurchase()` | `ShopManager.cs` | ☐ |
| 2B. Always enable buy button in `RefreshUI()` | `ShopItemCard.cs` | ☐ |
| 3A. Remove hard-reset on missed day | `DailyRewardSystem.cs` | ☐ |
| 3B. Add `dailyRewardAdClaimed` array | `SaveData.cs` | ☐ |
| 3C. Rewrite `DailyRewardCard.cs` | `DailyRewardCard.cs` | ☐ |
| 3D. Add Watch Ad button in Unity Inspector | Unity Scene | ☐ |
| 4A. Add `freeSpinUsedDate` to save data | `SaveData.cs` | ☐ |
| 4B. Rewrite `SpinWheelController.cs` | `SpinWheelController.cs` | ☐ |
| 4C. Assign `spinButtonText` in Inspector | Unity Scene | ☐ |

---

## ⚠️ Notes & Tips

> [!IMPORTANT]
> After making these code changes, **delete your save file** (`gamesave.json` in `Application.persistentDataPath`) once to clear old data so the new fields (`dailyRewardAdClaimed`, `freeSpinUsedDate`) are initialized correctly.

> [!TIP]
> To find `persistentDataPath` on Android: `/data/data/<package>/files/`. On Windows Editor: `%APPDATA%\..\LocalLow\<Company>\<Product>\gamesave.json`. Just click "Delete Save" in your settings if you have that button.

> [!NOTE]
> The `DOVirtual.DelayedCall(1.5f, ...)` in Spin Wheel Fix 4B is a **fallback** for when the user cancels the ad. If your `AdMobManager.ShowRewardedAd()` already has a `onAdClosed`/`onAdCancelled` callback, wire `_state = SpinState.ReadyToSpin` there directly and remove the DelayedCall.

> [!WARNING]
> If you use **DOTween Safe Mode**, make sure `DOTween.Kill(this)` in `ResetForNewLevel()` won't throw errors. Safe mode handles this gracefully by default.


+++++++++++++++++++++++++++++++++++++++++++CustomeLevelEditor+++++++++++++++++++++++++++++++++++++++++++++++++++++++

Here is a **clean, structured, “copy-paste ready” prompt for Antigravity / AI refactor tool** based on your full system.

It includes:

* NO breaking existing gameplay
* NO changes unless required
* Image → grid generation system
* Dynamic vehicle sizes (car/bus)
* Safe refactor rules
* Requirement for MD documentation
* Strong safety around runtime flow

---

## ✅ ANTI-GRAVITY PROMPT (COPY THIS)

```text
You are working inside an existing Unity project that already has a fully functional car puzzle / grid-based gameplay system.

CRITICAL RULE:
Do NOT break existing gameplay flow under any condition. The current game must continue to run exactly as it is. If any change risks breaking gameplay, you must either:
1. Preserve old logic, OR
2. Add new logic as optional fallback only.

------------------------------------------------------------
🎯 MAIN GOAL
------------------------------------------------------------

Implement a NEW “Image-to-Grid Level Generator” system inside the existing LevelEditorWindow.

This system must:
- Convert imported images into grid-based puzzle layouts
- Support scaling any image (small or large) into a fixed or dynamic grid
- Work with current LevelData system
- NOT replace existing manual grid editor (it should still work)

------------------------------------------------------------
🧠 IMAGE TO GRID REQUIREMENTS
------------------------------------------------------------

1. User imports a Texture2D image in editor.
2. Image is NOT pixel-to-cell mapping.
3. Instead:
   - Image is analyzed in regions
   - Each grid cell samples a region of the image
   - Result is normalized into current grid size (example: 7x7, 9x9, 11x11)

4. Image must be automatically:
   - Cropped (remove empty/white space)
   - Centered
   - Scaled to fit grid shape

5. Output:
   bool[,] drawnPattern

6. Dark areas in image = active car spawn cells
7. Light areas = empty cells

------------------------------------------------------------
🚗 VEHICLE SYSTEM (IMPORTANT)
------------------------------------------------------------

The game supports multiple vehicle types:

- Small Car (1x1 or 1x2)
- Medium Car (1x2 or 1x3)
- Bus (larger footprint, configurable size)

You MUST implement or extend system so that:

✔ Vehicles are NOT fixed-size assumptions
✔ Each vehicle has a footprint size (width, height)
✔ Grid placement must respect footprint collision
✔ Vehicles must never overlap
✔ Grid must validate placement before spawning

Add a clean abstraction like:

VehicleSize / Footprint system

Example:
- SmallCar → 1x1
- Sedan → 1x2
- Bus → 1x3 or larger

------------------------------------------------------------
🚗 SPAWN SYSTEM RULES
------------------------------------------------------------

- Must integrate with existing SpawnCars system
- Must NOT break current LevelData loading
- Must preserve current car spawn logic

If image-based system exists:
- It should override ONLY pattern generation
- NOT override runtime spawn logic

------------------------------------------------------------
🔁 COMPATIBILITY RULE
------------------------------------------------------------

Existing systems MUST continue working:
- Manual grid editor
- Legacy LevelData (carsToSpawn fallback)
- Current spawn logic in SpawnCars
- Object pooling system
- Car movement logic

If needed:
- Use feature flags or optional toggles

------------------------------------------------------------
📦 LEVELDATA RULE
------------------------------------------------------------

Extend LevelData ONLY if necessary.

Must still support:
- gridColumns
- gridRows
- carSpawns (existing system)
- carsToSpawn fallback

Do NOT break serialization.

------------------------------------------------------------
🚗 CAR ROTATION SYSTEM
------------------------------------------------------------

Existing system uses:
- 0, 45, 90, 135, 180, 225, 270, 315 degrees

Ensure:
- Image-generated layouts still support outward-facing rotation logic
- Do NOT break CalculateOutwardDirection logic
- Maintain compatibility with SpawnCars rotation system

------------------------------------------------------------
⚡ PERFORMANCE RULE
------------------------------------------------------------

- Use cached pixel reads if possible
- Avoid per-frame computation
- Editor-only heavy processing is allowed
- Runtime must remain fast

------------------------------------------------------------
📘 DOCUMENTATION REQUIREMENT (MANDATORY)
------------------------------------------------------------

Before finishing:
Generate a detailed MARKDOWN FILE explaining:

1. Architecture overview
2. Image-to-grid algorithm
3. Vehicle footprint system
4. Integration with SpawnCars
5. LevelData structure
6. How to extend system
7. Performance considerations
8. Known limitations

File name:
IMAGE_LEVEL_SYSTEM.md

------------------------------------------------------------
🚨 SAFETY RULE
------------------------------------------------------------

If any change can break gameplay:
- DO NOT apply it directly
- Provide backward-compatible alternative
- Or wrap in optional toggle

------------------------------------------------------------
✅ FINAL GOAL
------------------------------------------------------------

After implementation:
- Game must run exactly like before
- New system must be optional enhancement
- Designers should be able to generate levels from images
- Vehicles must support multiple sizes correctly
- No spawning bugs or overlap issues
```

---

## If you want next upgrade (I can help)

I can also help you design:

* 🔥 Smart “AI level generator from doodles”
* 🔥 Road/path detection instead of just blocks
* 🔥 Automatic difficulty balancing
* 🔥 Traffic-style simulation mode
* 🔥 True puzzle solver validation (no unsolvable levels)

Just tell me 👍
