---
name: ntronpa-translation-plugin
description: EROonpa project only — how the runtime Chinese translation layer (js/plugins/NTRonpa_Chinese.js + NTRonpa.json) is hooked, registered, and verified in the NTRonpa RPG Maker MZ games; read before touching the plugin, its dictionary, or plugins.js.
whenToUse: Editing, re-registering, or debugging the runtime translation plugin in D:\EROonpa\NTRonpa 0.3 or 0.4.
---

# NTRonpa runtime translation plugin

The game is RPG Maker MZ on NW.js. Chinese display is produced entirely by the
plugin `js/plugins/NTRonpa_Chinese.js`, which reads the dictionary
`NTRonpa.json` from the game root. Without this plugin the game renders its
original English (the MTool injector that `与工具一同启动.bat` used is long
gone from the machine; do not try to restore MTool).

## Hook points — changing these silently breaks translation

- `Window_Base.convertEscapeCharacters`: translate the **argument**, not the
  return value. The engine rewrites every `\` into `\x1b` inside this
  function, so the return value no longer contains `\N[4]` / `\LastGainObj`
  and the token scanner would eat control codes as ordinary text.
- `Bitmap.prototype.drawText` AND `Bitmap.prototype.measureTextWidth`: hook
  both. If measurement does not see the same translated string as drawing,
  centring and word-wrap drift.
- `Game_System.prototype.mainFontFace`: prepend a CJK font stack
  (`"Microsoft YaHei", "SimHei", "Noto Sans CJK SC", ...`) before the generic
  `sans-serif`, otherwise Chinese renders as tofu blocks.

## Registration in js/plugins.js

`js/plugins.js` is `var $plugins = [ ... ];`. Append the plugin entry before
the closing `]` and re-validate by parsing with python:
`json.loads(s[s.index('['):s.rindex(']')+1])`. The plugin must load AFTER
`VisuMZ_1_MessageCore` — that plugin is obfuscated and overrides
`convertEscapeCharacters` itself; loading before it means its override wins.

## Parameters

- `PluginManager.parameters()` lowercases all keys (`setParameters` does
  `name.toLowerCase()`), so parameter lookups must be case-insensitive.
- `Minimum Key Length` must be **3**, not 4. A value of 4 silently drops
  3-char menu words like `Buy`.

## Dictionary file resolution

Priority order: `nw.App.startPath` → `dirname(process.execPath)` → page
location → `process.cwd()` LAST. An earlier version tried `process.cwd()`
first and, when cwd pointed at the other game version, silently loaded the
wrong dictionary (0.3's dict while running 0.4). Never let cwd win.

## Text codes that must survive translation untouched

- Bracketed: `\N[x]`, `\P[x]`, `\V[x]`, `\C[x]`, `\I[x]`, `\FS[x]`, ...
- Bracket-less VisuMZ MessageCore codes — a generic "backslash + word" rule
  is NOT sufficient, they must be enumerated:
  `LastGainObj, LastGainObjName, LastGainObjQuantity, NormalBG, DimBG,
  TransparentBG, ResetColor, ResetFont, WindowReset, WindowMoveTo,
  WindowMoveBy, ActorFace, PartyFace`.
- Single-char controls: `\G \. \| \^ \! \< \> \{ \} \\`.
- Tags: `<center>`, `</center>`, `<Wordwrap>`, `<BR>`.

The token regex must accept BOTH the raw backslash form (what
`convertEscapeCharacters` receives) and the `\x1b` form (what `drawText`
receives after conversion).

## Verification without launching the game

Run the plugin file in a Node `vm` sandbox with stubs (`PluginManager`
returning lowercased keys, `nw.App.startPath` pointing at the real game root,
stub `Game_System`/`Bitmap`/`Window_Base` prototypes), then check
`window.__NTRonpaZH` and spot-translate menu terms. Note: if the dictionary
has no usable entries the plugin intentionally returns before installing
hooks (`if (dictionary.size === 0) return;`) — `__NTRonpaZH` being undefined
then means "empty dictionary", not "plugin broken". This is how the
fully-empty-value dictionary state was correctly diagnosed.

PowerShell console prints CJK as mojibake even with `chcp 65001`; verify
Chinese output by writing it to a file and reading it back with
`Get-Content -Encoding UTF8`, never by eyeballing console output.
