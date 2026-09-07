# Pirates Outlaws (航海奇闻) — Android build with official PC mod support

> This repository provides a **modified game APK** that lets the Android version load
> **official PC mods from the Steam Workshop**.
>
> **Download**: from the [Releases](https://github.com/caffetsong/andriodPiratesOutlawsModed/releases) page
>
> **Checksum** (SHA-256):
>
> ```
> 9c7514b2f9b8c484a00a77dba06f6e5d26bfc1c9b23e585583b3e87c7af5ad30
> ```

- Package: `com.piratesoutlaws.roguelikecard.fabled.cn` (build 1.97)
- Requirements: **no root**; because the APK is signed with a different key, the official version
  **must be uninstalled first**

---

## 1. What's new: official PC mods

The stock Android build has **no mod entry point at all** — yet the entire official mod codebase
is still present in the package: `ModManager`, `ReadTable`, `FindMapping`, `ExtraAssetsPivot`, and
the `-ModGroup` string constant are all there.

There is exactly one reason it never works: **nothing ever passes `-ModGroup` to the game**, so
`UtilsConfig.UseMod` stays `false` and the whole mod pipeline sleeps.

This build feeds that argument in and fixes the loader's Android-specific defects. Everything else
**runs through the official code paths** — not a single line of game script is changed.

| | Stock Android | This build |
|---|---|---|
| Loading Workshop mods | ✗ impossible | ✓ supported; all three directory layouts (`Table/` + `Image/` + `Localization/`) are auto-detected, **no `mods.txt` needed** |
| ID collisions | — | ✓ the official auto-remapping is reused (cards 9001+, characters 30001+, relics 3001+ …), so mod authors don't need to care |
| Several mods side by side | — | ✓ same-name sprite collisions are **isolated automatically**: renamed in place, with that mod's own table references rewritten |
| Card names / ability text | — | ✓ mod localization is merged into the language table |
| Portraits / card art / ability icons | — | ✓ render correctly |
| Saves | shared with the base game | ✓ **separate mod save** `save1_mod_<group>.sav`; the original save is untouched |

---

## 2. Four defects in the **game itself**, fixed along the way

None of these come from modding. They are all code that is **fine on PC and fatal on Android** —
and they are the real reason official mods never worked on mobile.

| # | Where | Symptom | What this build does |
|---|---|---|---|
| 1 | `System.String.Replace` | `ReadTable.Read` computes its lookup key with `Name.Replace(Extension, "")`, but all 52 call sites pass names **without** an extension → `Replace("")` throws `ArgumentException`: **the game crashes at the splash screen as soon as any mod is installed** | An empty `oldValue` becomes a no-op ("nothing to replace") |
| 2 | `ModManager.GetModPaths` | It enumerates `…/workshop/content/1046300/` (the PC Workshop folder). On Android that path does not exist, and enumerating it **does not throw — it segfaults**. The crash happens *after* local mods have been queued, which is deeply misleading: the log says "1 mod loaded" while not a single table was read | Takes over this pure path-discovery function and builds a usable mod directory itself |
| 3 | `ModManager.LoadAllImages` | Same-name sprites are first-come-first-served (`if (!modSpriteDic.ContainsKey(name))`), so two mods shipping the same status icon overwrite each other | Colliding images are **renamed in place** and that mod's own references are rewritten, so each mod keeps its own art |
| 4 | `ModeView.Open` / `AdventureControl.EndGame` | Giving up inside the tutorial goes through `EndGame`, which **never writes** `Tutorial_AllFinish`; the main menu's `ModeView.Open` sets it to 1 but **never persists it** → the next `LoadGame` resets it to 0 → **re-entering Adventure starts the tutorial again, with no way out** | Calls the game's own `QuickSave` after the original returns, so the flag is actually written to disk |

> Defect 3 exists on PC as well (it is the game's own design); this build fixes it too.

---

## 3. Optional features

| Feature | Description | Toggle (default) |
|---|---|---|
| **Skip shuffle animation** | The shuffle animation and its 2-second wait are removed when the draw pile runs out, **while the card-moving logic is left completely untouched** (the original method still runs; only the presentation layer is dropped). Sound is kept | `skip_shuffle=1` in `<mod>/pomod.cfg` or `<files>/pomod.cfg` (default **off**) |
| **Battle log** | `<files>/battle.log`: battle header plus card add/remove, relics, card plays, fight-data reads/writes and condition attachments, each line stamped with **seconds since the battle started** and flushed immediately | disabled by `<files>/no_battle_log` (default **on**) |
| **Cross-mod mechanic isolation** | Fixes "another mod's ability executing on your hero": measured with a non-Mage hero, the Mage's element-settlement chain ran and inserted its own card into your hand | disabled by `<files>/no_mod_filter` (**default on**) |
| **No gallery pollution** | Mod images live in a media-scanned directory yet never show up in the gallery | automatic |

---

## 4. Installation

### 1. Uninstall the official version

The signatures differ, so it **cannot be installed over the top**. Uninstalling wipes local
progress; back up
`Android/data/com.piratesoutlaws.roguelikecard.fabled.cn/files/save1china.sav` first if that matters.

### 2. Install this build

Install the APK downloaded from Releases.

### 3. Add mods

Copy a Workshop mod folder (containing `Table/` + `Image/` + `Localization/`) to **any** of the
locations below; the loader discovers them automatically:

```
/sdcard/Android/media/com.piratesoutlaws.roguelikecard.fabled.cn/<mod name>/
```

---

## 5. What this build does **not** include

- **No mods.** This APK only turns the game into one that *can* load mods; mod files are supplied separately.
- **No changes to game values, rules or balance.** Every change is limited to "letting mods load" plus the four defect fixes above.
- No changes related to ads or in-app purchases.

---

## 6. Known limitations

1. **Self-signed**: cannot be installed over the official version, and official-account features are unavailable.
2. **The mobile build ships fewer assets than PC**: `event_images_6`, some `heroposNN` files and `StreamingAssets/ExtraAssets` are missing. Mods referencing them will show missing art.
3. **Mod authors may hard-code runtime IDs**: one measured mod's start event contains `AddHideRelics("9112")` — a **runtime number** (the correct form is `ModRelicID(9001)`). Under a different mod set such hard-coding can point at completely unrelated content; this is a mod-side issue and worth reporting to the author.
4. **Cross-mod synergies are isolated**: the default isolation ("a mod's ability only executes on that mod's own character") also blocks deliberately designed cross-mod interactions (none have been observed so far).

---
