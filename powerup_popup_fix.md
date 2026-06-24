# 🔔 PowerUp Popup Bug — Fix Guide

## 🐛 The Bug in Simple Words

```
Level 4 starts → Popup appears → Player CLOSES GAME before clicking popup
       ↓
Player opens game again → Level 4 loads again
       ↓
Popup appears on TOP of all other UI
```

**Why?** Because the popup only marks itself as "seen" (`PlayerPrefs.SetInt = 1`)
**WHEN THE PLAYER CLICKS IT.**
If the player closes the game WITHOUT clicking → `PlayerPrefs` is never saved → next time the game thinks popup was never shown → shows it again.

---

## 🔍 Exactly Where the Bug Is

**File:** [PowerUpTutorialController.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-Grid/Car-OUT-jam-puzzle-Game-Grid/Assets/Script/Tutorial/PowerUpTutorialController.cs)

**Line 115–139 — `TryShowTutorial()`**

```csharp
public void TryShowTutorial(int level)
{
    _currentLevel = level;

    if (level == 4 && PlayerPrefs.GetInt(KEY_P1, 0) == 0)   // ← checks if seen
    {
        ShowPopup(...);   // ← shows popup BUT doesn't save "seen" yet
    }
    ...
}
```

**Line 171–186 — `OnPopupClicked()`**

```csharp
public void OnPopupClicked()
{
    popupPanel.SetActive(false);

    // ← "seen" is ONLY saved here — when player CLICKS
    if (_currentLevel == 4) PlayerPrefs.SetInt(KEY_P1, 1);
    PlayerPrefs.Save();
}
```

**The gap:** Between `ShowPopup()` and `OnPopupClicked()` → if game closes → flag never saved.

---

## ✅ The Fix — Save "seen" the MOMENT popup shows

### Where to change: `TryShowTutorial()` in [PowerUpTutorialController.cs](file:///c:/Users/ABHAYprajapati/Downloads/Car-OUT-jam-puzzle-Game-Grid/Car-OUT-jam-puzzle-Game-Grid/Assets/Script/Tutorial/PowerUpTutorialController.cs)

**BEFORE (broken):**
```csharp
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
```

**AFTER (fixed) — mark as seen RIGHT WHEN popup shows:**
```csharp
public void TryShowTutorial(int level)
{
    _currentLevel = level;

    if (level == 4 && PlayerPrefs.GetInt(KEY_P1, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P1, 1);   // ← SAVE IMMEDIATELY
        PlayerPrefs.Save();              // ← WRITE TO DISK NOW
        ShowPopup(t1_bgSprite, t1_iconSprite,
                  "Quick Fill",
                  "Instantly fills the first car\nwaiting in the parking slot!",
                  t1_targetButton);
    }
    else if (level == 6 && PlayerPrefs.GetInt(KEY_P2, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P2, 1);   // ← SAVE IMMEDIATELY
        PlayerPrefs.Save();              // ← WRITE TO DISK NOW
        ShowPopup(t2_bgSprite, t2_iconSprite,
                  "Rearrange",
                  "Fills all cars in parking slots by\nrearranging all passengers in the queue!",
                  t2_targetButton);
    }
    else if (level == 7 && PlayerPrefs.GetInt(KEY_P3, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P3, 1);   // ← SAVE IMMEDIATELY
        PlayerPrefs.Save();              // ← WRITE TO DISK NOW
        ShowPopup(t3_bgSprite, t3_iconSprite,
                  "Helicopter",
                  "A helicopter picks up the required car\nand drops it on the VIP slot!",
                  t3_targetButton);
    }
}
```

> [!IMPORTANT]
> The `OnPopupClicked()` function already has `PlayerPrefs.SetInt` + `PlayerPrefs.Save()` too — **leave that as-is**. It's harmless to save twice. No need to remove it.

---

## 🗺️ What the Fix Does — Timeline

```
BEFORE FIX:
Level 4 loads → popup shows → [game closes] → flag NOT saved → popup shows again ❌

AFTER FIX:
Level 4 loads → flag saved to disk IMMEDIATELY → popup shows → [game closes]
       ↓
Game opens again → flag is already 1 → popup does NOT show ✅
```

---

## 📋 Full Fixed `TryShowTutorial()` — Copy This Exactly

Replace lines **115 to 140** in `PowerUpTutorialController.cs` with:

```csharp
public void TryShowTutorial(int level)
{
    _currentLevel = level;

    if (level == 4 && PlayerPrefs.GetInt(KEY_P1, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P1, 1);
        PlayerPrefs.Save();
        ShowPopup(t1_bgSprite, t1_iconSprite,
                  "Quick Fill",
                  "Instantly fills the first car\nwaiting in the parking slot!",
                  t1_targetButton);
    }
    else if (level == 6 && PlayerPrefs.GetInt(KEY_P2, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P2, 1);
        PlayerPrefs.Save();
        ShowPopup(t2_bgSprite, t2_iconSprite,
                  "Rearrange",
                  "Fills all cars in parking slots by\nrearranging all passengers in the queue!",
                  t2_targetButton);
    }
    else if (level == 7 && PlayerPrefs.GetInt(KEY_P3, 0) == 0)
    {
        PlayerPrefs.SetInt(KEY_P3, 1);
        PlayerPrefs.Save();
        ShowPopup(t3_bgSprite, t3_iconSprite,
                  "Helicopter",
                  "A helicopter picks up the required car\nand drops it on the VIP slot!",
                  t3_targetButton);
    }
}
```

---

## 🧪 How to Test

1. Make the fix
2. In Unity, **right-click** `PowerUpTutorialController` component → click **"DEV: Reset All Tutorial Seen Flags"** (it's already in your code at line 286)
3. Hit Play → go to Level 4 → popup appears
4. **DO NOT click it** — stop Play mode (simulates closing game)
5. Hit Play again → go to Level 4 → **popup should NOT appear** ✅

---

> [!NOTE]
> Only **3 lines added** per level (`SetInt` + `Save()`). Zero other changes needed anywhere.
