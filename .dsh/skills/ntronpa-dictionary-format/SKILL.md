---
name: ntronpa-dictionary-format
description: EROonpa project only — the exact extraction and formatting rules for the NTRonpa.json translation dictionary; read before regenerating, extending, or diffing the dictionary for either game version.
whenToUse: Rebuilding NTRonpa.json, adding entries, comparing 0.3 and 0.4 dictionaries, or diagnosing "why are lines still English / full of garbage entries".
---

# NTRonpa.json dictionary format and extraction rules

`NTRonpa.json` in each game root is a flat `{"english": "中文"}` map. The 0.3
file is the historical MTool cache; the 0.4 file is generated. Key facts that
took several failed iterations to establish — each rule below was a real
defect.

## Key hygiene (user-enforced)

- **No leading/trailing whitespace.** The 0.3 MTool cache has zero untrimmed
  keys. Game text carries 4-space indents and huge runs of trailing padding
  spaces for alignment — these are layout garbage, strip them.
- **No runs of 3+ internal spaces.**
- **Multi-line messages must be merged.** One message block = consecutive
  `code:401`/`405` commands joined with `\n`. A block is terminated ONLY by
  `code 101` (new speaker), `code 0` (event end), `code 102`, `code 402`.
  Codes like `250` (SE), `231`/`235` (pictures) do NOT terminate a message —
  flushing on them splits one dialogue into orphan fragments.
- Values may be empty string (`""`) — that is the intentional "not yet
  translated" state for keys awaiting human translation.

## What counts as an entry (source field whitelist)

Text lives in exactly these places:

- Event commands: `401`/`405` (body), `101` `parameters[4]` (speaker name),
  `102`/`402` (choices), and `357` (plugin command) parameters containing a
  `text` sub-field — the kitchen/cooking messages (`Message = "Got Beef!"`)
  live ONLY there; scanning just `parameters[0]` loses ~58 entries.
- Database fields: `description`, `note`, `profile`, `nickname`,
  `message1`–`message4`, and `name` — but `name` only in
  `Items/Skills/States/Armors/Weapons/Actors/Classes/Enemies/MapInfos.json`.

## What must NOT be translated or extracted (user-enforced exclusions)

- Character names: `Matthias, Kaede, Marcus, Seika, Katie, Mia, Sam, Stacy,
  Chise, Keiko, Hina, Angelina, Bakugen, Dave, Touka, Toka, Luke, Celes,
  Kazama, Miko, Sarah, Saya, Duokuma, Halfbear, ChiroGPT` — used as
  `Actors.json` name keys and `101` speaker parameters.
- Internal identifiers: `SawQuestVRLVL1, ScavengerMap1, WorktimeMap, testmap,
  K4I-B4, N4N4`, anything that is both a `CommonEvents` name and a
  `System.json` switches/variables name — translating these breaks game logic.
- Resource/audio names: anything matching `^\d\d_…` (e.g.
  `03_離人症(Depersonalizatiion)`), `name/name` paths
  (`chiaki/moan`, `ero_kitakamitsubasa/01_02_fera`), SE names (`Battle1`,
  `Buzzer1`, `Bait&Switch`), `Animations.json` `effectName` values (`Breath`,
  `Poison`, `SonicWave`), image `characterName`/`faceName` etc.
- Pure escapes (`\N[4].`), booleans (`Yes`/`No`), `A`–`D` self-switch names.

## System.json terms ARE game UI — do not exclude them

A major earlier mistake: `terms` (commands/params/basic/messages) was
classified as "engine defaults" and dropped, leaving the main menu, options
panel, and battle UI in English while dialogue was translated. `commands` →
menu/battle/shop buttons, `params` → stat names, `messages` → options panel
labels and battle messages. All 83 unique terms must be present and
translated (only `HP/MP/TP/EXP/Lv` intentionally stay as-is). Placeholder
`%1` `%2` `\G` must be preserved verbatim.

## 0.3 vs 0.4 dictionaries are not comparable by string equality

The 0.3 MTool cache stores TRUNCATED fragments (keys cut at punctuation, e.g.
`「"I need your help` without closing quote, `...you can well` missing
`.. Reset!」`), while 0.4 extraction produces complete lines. Compare only
after normalizing: collapse all whitespace runs, strip quotes both sides —
and even then use substring containment, not equality. Roughly: of 0.3's
~6.4k translated entries, only ~782 match 0.4 exactly; ~1900 more are
fragments of 0.4's complete lines; the rest are plugin docs / engine error
strings / wasm blobs that MTool captured at runtime.

## Workflow rule the user enforced

When asked to extract English entries: keys only, values empty — do not
invent, borrow, or machine-translate values unless explicitly asked.
"Translate non-dialogue entries" meant game UI text, and each round of
scope must be confirmed against what actually shows on screen, not against
a precomputed classification.
