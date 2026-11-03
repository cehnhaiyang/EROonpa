---
name: ntronpa-static-translation-artifacts
description: EROonpa project only — where the 0.4 static Chinese translation actually hides (data/, data_orig/, plugins.js parameters) and the safe removal procedure; read before cleaning "leftover Chinese" or restoring English data.
whenToUse: The user reports Chinese still showing after cleanup, or asks to remove the static translation layer in NTRonpa 0.4.
---

# 0.4 static translation artifacts and removal

NTRonpa 0.4 shipped with a **static** Chinese translation: the English
originals were overwritten in place. The runtime plugin approach replaced it,
and removing the static layer required knowing exactly where it hides — the
first cleanup pass missed most of it.

## The three hiding places

1. **`data/*.json`** — dialogue/note text replaced in place.
   `data_orig/` holds the pristine English originals with identical structure
   (38,843 string values, 0 structural divergence). Restore by copying
   `data_orig/*.json` over `data/`. After a correct restore, exactly **13 CJK
   characters** remain in `data/` — they are BGM file names
   (`11_捜索02`, `03_離人症(Depersonalizatiion)`) in `code 241` parameters
   and are original content, NOT translation leftovers. That count is the
   pass/fail check for the restore.
2. **`data/System.json.bak2`** — a stale copy with fully translated `terms`
   and Chinese `gameTitle`. Delete it.
3. **`js/plugins.js` parameters** — the trap. Chinese hides inside VisuMZ
   plugin *parameters* (JSON-embedded strings), not in code:
   - `VisuMZ_1_OptionsCore`: `Categories:arraystruct` labels
     (`"Name:str":"常规"`, `切换分类`, `\C[16]探索`, `随机遇敌`, `未命名`,
     speed tiers `['无','一半','普通']`, `['关','部分','开']`,
     `['普通','快','更快','最快']`, `return '战斗动画显示'` inside TextJS)
   - `VisuMZ_1_SaveCore`: `"Name:str":"自动保存"`
   These show in the options screen. Fix by string replacement inside
   `plugins.js`, then re-parse the `$plugins` array to confirm JSON validity.
   Do NOT touch the ~11 CJK chars in `VisuMZ_1_MessageCore`'s
   `LanguageList` (`简体中文`, `繁體中文`) — those are VisuMZ's own language
   name list, shipped as-is.

Also expect Chinese-looking plugin **help comments** (`CGMZ_Crafting.js`
Chinese docs, `QuestSystem.js`/`PictureCallCommon.js`/`AltSaveScreen.js`
Japanese `@help`) — original author content, never displayed, never remove.

## Save files

`save/*.rmmzsave` are pako-deflate-compressed JSON that MZ double-encodes:
decompress requires `zlib.inflateSync(Buffer.from(raw.toString('utf8'),'latin1'))`
— read as UTF-8 then re-encode each codepoint as one byte. Saves store
in-flight event interpreter state including dialogue lines, so they can
contain English text that the runtime plugin re-translates on display; save
files are read-only analysis targets, never patch them for translation.

## Backup discipline

Before overwriting `data/`, copy it aside (e.g. `data_static_zh_backup/`).
Delete the backup only after verification. `save/file*.rmmzsave.bak` files
are engine autosave backups, unrelated to translation — leave them.
