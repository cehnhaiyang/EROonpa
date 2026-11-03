---
name: ntronpa-asset-decryption
description: EROonpa project only — how to decrypt RPG Maker MZ encrypted assets (.png_/.ogg_ with RPGMV header) and the layout of the unpack tool; read before unpacking assets or when image/audio files "fail to open".
whenToUse: User reports images/audio from img/ or audio/ cannot be viewed, or asks to unpack a new game version's assets.
---

# RPGMV asset decryption (NTRonpa)

Encrypted assets are `.png_` / `.ogg_` files with a 16-byte literal header
`52 50 47 4d 56 00 00 00 00 03 01 00 00 00 00 00` ("RPGMV"). Renaming the
extension to `.png` does NOT work — the body's first 16 bytes are XORed with
the game key. This is why "改了后缀也看不了".

## Decryption (mirrors `Utils.decryptArrayBuffer` in rmmz_core.js)

1. Verify the 16-byte header exactly.
2. Drop the header.
3. XOR the first 16 body bytes with the key parsed as hex pairs.
4. The result is a plain PNG / OGG.

The key lives in `data/System.json` as `encryptionKey`
(`0f16948d979cd75ade7d373a814002b7` for both 0.3 and 0.4 — same key across
versions, but always read it from the version's own System.json, never
assume). `hasEncryptedImages` / `hasEncryptedAudio` flags confirm scope.
Engine reference: `Bitmap.prototype._startDecrypting` requests
`this._url + "_"` — that trailing underscore is how `.png_` files are loaded.

## Coverage in this project

- `img/`: ~1132 (0.3) / ~1177 (0.4) encrypted PNGs; a handful of plaintext
  `.txt`/`.psd` are original.
- `audio/`: all `.ogg_` encrypted.
- `movies/`, `effects/`, `fonts/`: plaintext, skip them.

## Tool

`unpack-rpgmv.js` sits in each game root (D:\EROonpa\NTRonpa 0.3 and
NTRonpa 0.4). Usage:

    node unpack-rpgmv.js                  # decrypt into ./unpacked
    node unpack-rpgmv.js <outDir>         # custom output
    node unpack-rpgmv.js <outDir> --copy  # rename only, no decrypt

It sniffs the real format from the decrypted header and fixes the extension
accordingly. Copy the existing tool into a new version folder rather than
rewriting it. Verify results by checking every output file's magic bytes
(PNG `89 50 4E 47` / OGG `OggS`) — a full pass had 0 failures; anything
else means the key or header assumption is wrong.

Decrypted output lands in `unpacked/` (~494 MB per version) and is purely
for viewing assets; the game itself decrypts in-memory at runtime and never
reads `unpacked/`.
