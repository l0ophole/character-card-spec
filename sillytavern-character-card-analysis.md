# How SillyTavern implements the Character Card specifications

This document describes how SillyTavern (ST) loads, stores, uses and saves character cards and lorebooks, and where that behaviour differs from the Character Card V1, V2 and V3 specifications. Each statement links to the code or specification text it rests on, pinned to the commits below.

## Sources

| Repository | Commit | Commit date | Role |
|---|---|---|---|
| [malfoyslastname/character-card-spec-v2](https://github.com/malfoyslastname/character-card-spec-v2) | `8083fb3` | 2023-06-22 | V1 and V2 specifications, V2 explainer |
| [kwaroran/character-card-spec-v3](https://github.com/kwaroran/character-card-spec-v3) | `f3a86af` | 2024-07-20 | V3 specification |
| [SillyTavern/SillyTavern](https://github.com/SillyTavern/SillyTavern) (`release` branch, v1.19.0) | `06bde93` | 2026-09-14 | Implementation |
| [SillyTavern/SillyTavern-Docs](https://github.com/SillyTavern/SillyTavern-Docs) (`main` branch) | `ed5423d` | 2026-10-03 | User and contributor documentation |

**Method.** The findings come from reading the code at these commits. SillyTavern itself was not run. To confirm the on-disk formats, the default character (`default/content/default_Seraphina.png`) and default lorebook (`default/content/Eldoria.json`) that ship in the repository were decoded and inspected. One behaviour that depends on a third-party image library, the handling of animated PNGs, is marked as untested where it appears.

**Terms.** A *card* is the JSON object that describes a character. *World Info* (WI) is SillyTavern's name for a lorebook; a WI file is a JSON file in the user's `worlds/` folder. The *embedded lorebook* is the card's `data.character_book`. A card's *linked WI* is the WI file named by `data.extensions.world`. *Chat Completion* and *Text Completion* are SillyTavern's two families of model backends; they assemble prompts differently. Normative keywords (MUST, SHOULD, MAY) are quoted from the specifications.

## Contents

- [Summary](#summary)
- [1. What the specifications define](#1-what-the-specifications-define)
- [2. How SillyTavern models a card](#2-how-sillytavern-models-a-card)
- [3. Loading character cards](#3-loading-character-cards)
- [4. Saving character cards](#4-saving-character-cards)
- [5. Lorebooks and World Info](#5-lorebooks-and-world-info)
- [6. Prompt-building behaviour](#6-prompt-building-behaviour)
- [7. Conformance tables](#7-conformance-tables)
- [8. Documentation compared with the code](#8-documentation-compared-with-the-code)
- [9. What survives a round trip](#9-what-survives-a-round-trip)
- [10. Consequences for card authors and other tools](#10-consequences-for-card-authors-and-other-tools)
- [Appendix: code index](#appendix-code-index)

## Summary

SillyTavern is a V2 implementation at its core. It stores every card as a combined V1 + V2 object inside a PNG, writes a second copy of that same object labelled as V3 without implementing any V3 feature, and treats a card's embedded lorebook as something to import into its own World Info format, not as data it uses directly.

The differences that matter most:

1. **One storage format, one hybrid object.** Every character is a PNG holding two text chunks, `chara` and `ccv3`. The JSON in them is a hybrid: the V2 `spec` / `spec_version` / `data` wrapper, plus the six V1 fields repeated at the top level, plus SillyTavern's own top-level fields (`creatorcomment`, `avatar`, `chat`, `talkativeness`, `fav`, `tags`, `create_date`). V3 says an application must not add its own fields outside `extensions`, and the V2 explainer says fields were nested under `data` precisely so that V1-only editors would fail to load V2 cards. ([2.1](#21-storage), [2.2](#22-the-hybrid-object))
2. **V3 support is a label.** The `ccv3` chunk is the same JSON with `spec` set to `chara_card_v3` and `spec_version` to `3.0`. `nickname`, `creator_notes_multilingual`, `source`, `group_only_greetings`, `creation_date` and `modification_date` are never read or written, and `assets` is read only while importing a CHARX file. Cards created in SillyTavern therefore lack `group_only_greetings`, which V3 requires. ([4.2](#42-writing-the-png), [7.3](#73-character-card-v3))
3. **Exported files are labelled V3 in both chunks.** Stored files usually carry `chara_card_v2` in `chara` and `chara_card_v3` in `ccv3`. The export path reads the `ccv3` chunk and writes it back into both, so an exported PNG has `chara_card_v3` in the `chara` chunk too, and a JSON export says `chara_card_v3`. ([4.3](#43-export))
4. **The embedded lorebook is not used until the user imports it.** Prompt building reads only WI files. A card's `character_book` has no effect until the user accepts an import dialog (or uses "Import Card Lore"), which converts it into a WI file and links it. V2 and V3 both say the character lorebook MUST be used by default. ([5.4](#54-life-cycle-of-an-embedded-lorebook))
5. **Once linked, the embedded lorebook is regenerated on every save.** The card's `character_book` is rebuilt from the WI file. The rebuilt book contains only `name` and `entries`: the book-level `description`, `scan_depth`, `token_budget`, `recursive_scanning` and `extensions` are gone, as are each entry's `name`, `priority` and top-level `case_sensitive`. Every entry gets `use_regex: true`. ([5.2](#52-field-mapping-between-the-two-formats))
6. **Several lorebook fields are ignored at run time.** Book-level `scan_depth`, `token_budget` and `recursive_scanning` are replaced by the user's global WI settings (recursion is on in the default settings even when a book says `false`, which V3 forbids). Entry-level `case_sensitive`, `use_regex` and `priority` are not read. ([5.5](#55-run-time-behaviour-compared-with-the-specifications))
7. **Only two V3 decorators work.** `@@activate` and `@@dont_activate` are implemented, and the `@@@` fallback syntax is parsed. All other decorators are removed from the content and otherwise ignored. ([5.6](#56-decorators))
8. **Nothing is validated on import.** Any JSON with a `spec` property is treated as V2-shaped, whatever its value; otherwise any JSON with `name` is treated as V1. The only validator runs on one internal update endpoint and accepts SillyTavern's own cards as V1 before the V2 or V3 checks are reached. ([3.3](#33-version-detection-and-validation), [4.4](#44-other-write-paths-and-the-validator))
9. **Card prompt overrides are on by default but conditional.** `system_prompt` and `post_history_instructions` replace the user's prompts by default and `{{original}}` works, but both depend on user settings, on the user's System Prompt being enabled (Text Completion), and on the target prompt not being disabled or protected (Chat Completion). ([6.1](#61-system_prompt-and-post_history_instructions))
10. **CHARX is import-only and partial.** The main `icon` becomes the avatar and other image assets are extracted into SillyTavern's own folders; `user_icon` assets and all non-image assets are skipped without telling the user. There is no CHARX export, and the card keeps `assets` entries whose `embeded://` paths no longer resolve. ([3.5](#35-charx))
11. **World Info files use SillyTavern's own format.** Import accepts SillyTavern, NovelAI, Agnai and RisuAI lorebooks. It does not accept a standalone V2 `CharacterBook` or the V3 `lorebook_v3` wrapper, and export always produces the SillyTavern format. ([5.3](#53-loading-and-saving-world-info-files))
12. **Macros mostly match V3.** `{{char}}`, `{{user}}`, `{{random}}`, `{{pick}}`, `{{roll}}`, `{{// }}` and `{{reverse}}` behave as specified. `{{char}}` never uses `nickname`, `{{hidden_key}}` is not implemented and is left in the text, and `{{comment}}` is removed but not shown as an inline comment. ([6.5](#65-macros-curly-braced-syntaxes))
13. **The documentation says V2 is fully supported and does not mention V3.** The contributor guide states that SillyTavern "fully supports" V2. V3, CHARX and decorators are not documented. ([8](#8-documentation-compared-with-the-code))

## 1. What the specifications define

| | V1 | V2 | V3 |
|---|---|---|---|
| Object | Six string fields at the top level: `name`, `description`, `personality`, `scenario`, `first_mes`, `mes_example` ([`spec_v1.md:27-38`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L27-L38)) | `{ spec: 'chara_card_v2', spec_version: '2.0', data: {...} }`; the V1 fields move under `data`, which gains `creator_notes`, `system_prompt`, `post_history_instructions`, `alternate_greetings`, `character_book`, `tags`, `creator`, `character_version`, `extensions` ([`spec_v2.md:45-70`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L45-L70)) | `{ spec: 'chara_card_v3', spec_version: '3.0', data: {...} }`; adds `assets`, `nickname`, `creator_notes_multilingual`, `source`, `group_only_greetings`, `creation_date`, `modification_date` ([`SPEC_V3.md:75-112`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L75-L112)) |
| Embedding | JSON file, or base64 JSON in the PNG/APNG `chara` field ([`spec_v1.md:17-21`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L17-L21)) | Same as V1 | PNG/APNG `ccv3` tEXt chunk, JSON file, or CHARX zip with `card.json` and assets ([`SPEC_V3.md:22-59`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L22-L59)) |
| Lorebook | None | `character_book` with entries ([`spec_v2.md:84-111`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L84-L111)) | Same shape plus `use_regex`, decorators in `content`, and a standalone `lorebook_v3` export format ([`SPEC_V3.md:217-259`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L217-L259)) |
| Macros | `{{char}}`, `{{user}}`, `<BOT>`, `<USER>` ([`spec_v1.md:40-42`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L40-L42)) | Adds `{{original}}` in the two prompt override fields ([`spec_v2.md:141`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L141)) | Adds `{{random}}`, `{{pick}}`, `{{roll}}`, `{{// }}`, `{{hidden_key}}`, `{{comment}}`, `{{reverse}}` ([`SPEC_V3.md:564-605`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L564-L605)) |

## 2. How SillyTavern models a card

### 2.1 Storage

Each character is one PNG file in the user's `characters/` folder. There is no JSON or CHARX storage: every import format is converted to a PNG, and the list of characters is built by reading every `.png` in that folder ([`characters.js:1466-1478`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1466-L1478)).

The card JSON is base64-encoded into two `tEXt` chunks. `write()` removes any existing `chara` and `ccv3` chunks, adds `chara` with the JSON it was given, then adds `ccv3` with the same JSON after setting `spec` to `chara_card_v3` and `spec_version` to `3.0` ([`character-card-parser.js:15-46`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js#L15-L46)). `read()` returns the `ccv3` chunk if there is one and otherwise `chara` ([`character-card-parser.js:54-78`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js#L54-L78)). Only `tEXt` chunks are examined, and the keywords are compared case-insensitively.

The default character that ships with SillyTavern shows the result. Its PNG contains exactly `IHDR`, `IDAT`, two `tEXt` chunks and `IEND`. The two chunks are the same size and differ only in the `spec` and `spec_version` values.

### 2.2 The hybrid object

This is the `chara` chunk of the default character, shortened and with its keys grouped for reading:

```jsonc
{
  // V1 fields, repeated at the top level
  "name": "Seraphina",
  "description": "[Seraphina's Personality= ...",
  "personality": "",
  "scenario": "",
  "first_mes": "*You wake with a start, ...",
  "mes_example": "",
  // SillyTavern's own top-level fields
  "avatar": "none",
  "chat": "Seraphina - 2023-5-12 @21h 32m 29s 224ms",
  "create_date": "2023-5-12 @21h 32m 29s 228ms",
  "talkativeness": "0.5",
  "fav": false,
  "creatorcomment": "ST Default Bot contest winner: roleplay bots category",
  "tags": [],
  // V2 wrapper
  "spec": "chara_card_v2",        // "chara_card_v3" in the ccv3 chunk
  "spec_version": "2.0",          // "3.0" in the ccv3 chunk
  "data": {
    "name": "Seraphina",
    "description": "[Seraphina's Personality= ...",
    "personality": "", "scenario": "", "first_mes": "...", "mes_example": "",
    "creator_notes": "ST Default Bot contest winner: roleplay bots category",
    "system_prompt": "", "post_history_instructions": "",
    "tags": [], "creator": "OtisAlejandro", "character_version": "1.0.0",
    "alternate_greetings": [],
    "extensions": {
      "talkativeness": "0.5",
      "fav": false,
      "world": "Eldoria",
      "depth_prompt": { "prompt": "", "depth": 4, "role": "system" }
    },
    "character_book": { "entries": [ /* 4 entries */ ], "name": "Eldoria" },
    "group_only_greetings": []
  }
}
```

The object is assembled by `charaFormatData()`, which is called for every create and edit ([`characters.js:565-657`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L565-L657)):

| Part | Keys | Notes |
|---|---|---|
| V1 mirror | `name`, `description`, `personality`, `scenario`, `first_mes`, `mes_example` | Written with the same values as `data.*` ([`characters.js:579-585`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L579-L585)). |
| Legacy SillyTavern fields | `creatorcomment`, `avatar`, `chat`, `talkativeness`, `fav`, `tags`, `create_date` | The code comment calls these "Old ST extension fields (for backward compatibility, will be deprecated)" ([`characters.js:587-593`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L587-L593)). `create_date` is added by the import and edit handlers. |
| V2 wrapper | `spec`, `spec_version`, `data.*` | Always written as `chara_card_v2` / `2.0` ([`characters.js:595-612`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L595-L612)). |
| SillyTavern extensions | `data.extensions.talkativeness`, `.fav`, `.world`, `.depth_prompt` | Always written, including when empty ([`characters.js:614-626`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L614-L626)). |

The default character carries `group_only_greetings: []`, but at this commit no code path in the client or server reads or writes that key, so characters created in SillyTavern do not get it.

Two specification statements bear on this layout. V3 says that being able to keep unknown fields "MUST not taken to mean that the application can add their own fields to the CharacterCard object", and that application data belongs in `extensions` ([`SPEC_V3.md:117`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L117)). The V2 explainer says the nesting under `data` exists "to prevent V1-only editors ... from successfully loading a V2 card, but silently destroying its V2-only fields" ([`README.md:193`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/README.md#L193)). A card saved by SillyTavern has all six V1 fields at the top level, so a V1-only editor can load it, which is the situation the nesting was meant to prevent.

### 2.3 Which copy of a field is used

When a stored card is read, `readFromV2()` copies `data.name`, `data.description`, `data.personality`, `data.scenario`, `data.first_mes`, `data.mes_example`, `data.extensions.talkativeness`, `data.extensions.fav` and `data.tags` over the top-level fields, logging a warning if the two disagree ([`characters.js:504-557`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L504-L557)). The `data` copy therefore wins.

The client then reads the six V1 fields from the top level and everything else from `data`:

| Used for | Read from | Code |
|---|---|---|
| Character name (`{{char}}`) | top-level `name` | [`script.js:7685`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L7685) |
| Description, personality, scenario, example messages | top-level fields | [`script.js:3432-3453`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L3432-L3453) |
| First message | top-level `first_mes` | [`script.js:7711`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L7711) |
| Alternate greetings | `data.alternate_greetings` | [`script.js:7712`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L7712) |
| System prompt and post-history overrides | `data.system_prompt`, `data.post_history_instructions` | [`script.js:3413-3421`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L3413-L3421) |
| Creator notes | `data.creator_notes`, falling back to `creatorcomment` | [`script.js:8792`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L8792) |
| Linked lorebook | `data.extensions.world` | [`world-info.js:4481`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4481) |
| Character's Note | `data.extensions.depth_prompt` | [`script.js:4482-4485`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L4482-L4485) |

The client never looks at `spec` or `spec_version`.

### 2.4 SillyTavern's keys in `extensions`

SillyTavern writes `talkativeness`, `fav`, `world`, `depth_prompt` (on every save), `regex_scripts` and `sd_character_prompt` (when those features are used). It reads `chub.full_path`, `pygmalion_id`, `github_repo`, `source_url`, `risuai.source` and `perchance_data.slug` to build a "source" link ([`script.js:1247-1292`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L1247-L1292)). V2 recommends namespacing extension keys ([`spec_v2.md:204-205`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L204-L205)); none of SillyTavern's own keys are namespaced. `world` holds the name of a file on the exporting user's machine and is included in exported cards.

## 3. Loading character cards

### 3.1 Import formats

The client accepts files with the extensions `json`, `png`, `yaml`, `yml`, `charx` and `byaf` ([`script.js:10536-10539`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L10536-L10539)) and posts them to `/api/characters/import`, which chooses an importer by extension ([`characters.js:1567-1574`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1567-L1574)).

| Extension | Importer | How the content is interpreted | Avatar |
|---|---|---|---|
| `png` | [`characters.js:968-1020`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L968-L1020) | JSON from the `ccv3` chunk, else `chara`. If `spec` is defined, the object is kept as it is. Otherwise, if `name` is defined, it is treated as V1. | The uploaded image, re-encoded |
| `json` | [`characters.js:883-959`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L883-L959) | Same two cases, plus a third: if `char_name` is defined, the Pygmalion/Gradio format (`char_persona`, `char_greeting`, `example_dialogue`, `world_scenario`). | Default avatar |
| `yaml`, `yml` | [`characters.js:731-755`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L731-L755) | `name`, `context` (becomes `description`) and `greeting` (becomes `first_mes`). | Default avatar |
| `charx` | [`characters.js:765-801`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L765-L801) | `card.json` from the zip; see [3.5](#35-charx). | The `icon` asset, else default |
| `byaf` | [`characters.js:803-874`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L803-L874) | Backyard AI archive, converted to a V2 object ([`byaf.js:248-272`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/byaf.js#L248-L272)). | First image in the archive |

Importing from a URL downloads from a set of known sites and feeds the result into the same importers. For Chub and Perchance the server assembles a V2 object itself from the site's API response; for RisuRealm it requests the site's `png-v3` download ([`content-manager.js:440-499`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/content-manager.js#L440-L499), [`content-manager.js:735-749`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/content-manager.js#L735-L749)).

WEBP cards are not accepted. V1 leaves WEBP outside the specification ([`spec_v1.md:21`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L21)).

### 3.2 Reading the PNG

`ccv3` takes precedence over `chara`, as V3 recommends ([`SPEC_V3.md:28`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L28)). V3 also says a backfilled `chara` chunk "SHOULD be trimmed on import" ([`SPEC_V3.md:26`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L26)). In SillyTavern, when both chunks exist, `chara` is never read, and both chunks are rewritten from the `ccv3` content.

V3 also allows assets in extra `tEXt` chunks named `chara-ext-asset_:{path}` ([`SPEC_V3.md:30`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L30)). SillyTavern does not read them, and they do not survive the import (see [3.4](#34-what-import-changes)).

### 3.3 Version detection and validation

The importers decide the card version from two property checks ([`characters.js:980-990`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L980-L990)):

- `spec !== undefined`: the card is treated as V2-shaped. The value of `spec` is not examined, so `chara_card_v2`, `chara_card_v3` and any other string take the same path. `spec_version` is not examined either.
- otherwise, `name !== undefined`: the card is treated as V1.

No structural validation follows. `TavernCardValidator` is not called by any importer. Consequences:

- V3 says an application that does not support a newer `spec_version` SHOULD alert the user ([`SPEC_V3.md:129`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L129)). There is no such check or alert.
- A card with `spec` but no `data` is stored as it is, with a console warning ([`characters.js:505-508`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L505-L508)).
- A V1 card needs only `name`. The other five fields default to the empty string when missing or null ([`characters.js:997-1011`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L997-L1011)). V1 requires all six to be present ([`spec_v1.md:38`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L38)); SillyTavern is lenient when reading and always writes all six.

### 3.4 What import changes

For a card with `spec`, the importer keeps the object but makes these changes before storing it:

- **Name.** `data.name` and the top-level `name` are passed through `sanitize-filename`, which removes characters that are not allowed in file names ([`characters.js:974-977`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L974-L977)). A name such as `AC/DC` is altered in the card itself, not only in the file name.
- **Private fields.** `fav` and `data.extensions.fav` are set to `false`, and `chat` (the name of the character's current chat file) is discarded; the PNG and JSON importers then assign a new one ([`characters.js:498-502`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L498-L502), [`characters.js:554`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L554)).
- **Dates.** `create_date` is set to the time of import, as an ISO string at the top level ([`characters.js:985`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L985)).
- **Top-level mirror.** `readFromV2()` copies the six V1 fields and `tags` from `data` to the top level. Its comments describe a back-fill of default values for `talkativeness` and `fav` when they are missing from `data.extensions`, but the default it assigns is overwritten with `undefined` at the end of the same loop step, so nothing is back-filled ([`characters.js:525-552`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L525-L552)).
- **RisuAI sprites.** If `data.extensions.risuai.additionalAssets` or `.emotions` hold base64 images, they are written to the character's sprite folder and then deleted from the card ([`sprites.js:48-114`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/sprites.js#L48-L114)). This removes data from `extensions`, which V2 says MUST NOT be destroyed ([`spec_v2.md:200`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L200)). The images end up as files on disk, but they are no longer in the card and so are absent from any later export.
- **Image.** The image is decoded and re-encoded as PNG by the Jimp library before the card chunks are added ([`characters.js:239-256`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L239-L256), [`characters.js:283-306`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L283-L306)). Re-encoding keeps only the pixels, so any other metadata in the source PNG, including V3 `chara-ext-asset_:` chunks, is dropped. If Jimp cannot decode the file, the raw bytes are used instead; the code comment names APNG as the case it expects ([`characters.js:325-334`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L325-L334)). Whether an animated PNG keeps its animation depends on what Jimp does with it, which was not tested.
- **Chunk labels.** The `ccv3` chunk is regenerated with `spec_version` forced to `3.0`. A card that arrived with a higher version loses that information immediately, because `ccv3` is the chunk read from then on.

For a V1 card, a new object is built from the known fields only: the six V1 fields plus `creatorcomment` or `creator_notes`, `talkativeness`, `creator` and `tags` if present ([`characters.js:997-1012`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L997-L1012)). Other top-level keys are discarded. The result goes through `charaFormatData()` and is stored as a hybrid V2 object.

Nothing that is dropped or ignored during import is reported to the user.

### 3.5 CHARX

`CharXParser` handles `.charx` files ([`charx.js:51-110`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L51-L110)).

| V3 requirement | SillyTavern |
|---|---|
| `card.json` MUST be at the root of the zip ([`SPEC_V3.md:38`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L38)) | Takes the first entry whose name ends with `card.json`, at any depth ([`util.js:207-259`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/util.js#L207-L259)). Skips to the first zip signature in the file, so archives with data in front of the zip, such as self-extracting ones, work ([`charx.js:23-30`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L23-L30)). |
| Card object | Requires `spec` to be defined; nothing else is checked ([`charx.js:74-78`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L74-L78)). |
| Asset URIs use `embeded://` (the specification notes this spelling is intentional) | Accepts `embeded://`, `embedded://` and `__asset:`, case-insensitively ([`charx.js:10`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L10), [`charx.js:112-131`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L112-L131)). Assets with `ccdefault:`, `http(s):` or `data:` URIs are ignored. |
| The `icon` named `main` is the main icon ([`SPEC_V3.md:184`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L184)) | Uses the embedded image `icon` named `main`, else the first embedded image `icon`, else the default avatar ([`charx.js:202-210`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L202-L210)). |
| `emotion` assets | Types `emotion` and `expression` are saved as sprite files in `characters/{name}/` for the Character Expressions extension ([`charx.js:355-366`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L355-L366)). |
| `background` assets | Saved in `characters/{name}/backgrounds/` ([`charx.js:368-381`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L368-L381)). |
| Other asset types | Other image assets go to the character's image gallery folder ([`charx.js:383-392`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L383-L392)). `user_icon` assets and every non-image asset (audio, video, fonts, models, code) are skipped ([`charx.js:240-253`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js#L240-L253)). |
| If asset data is not saved, the application "MUST alert the user that the asset is not saved" ([`SPEC_V3.md:191`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L191)) | No alert. A summary of imported assets is written to the server console. |

After import, the card's `data.assets` array is stored unchanged. Its `embeded://` paths refer to a zip that no longer exists, and because SillyTavern cannot export CHARX, a later PNG or JSON export carries those unresolvable entries.

### 3.6 Reading stored cards

To list or open characters, the server reads the card string from each PNG (with a memory and disk cache), parses it, and passes it to `getCharaCardV2()` ([`characters.js:406-461`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L406-L461)):

- If `spec` is undefined, the card is converted in memory through `charaFormatData()`. This covers V1 PNGs that were copied into the folder by hand.
- Otherwise `readFromV2()` refreshes the top-level mirror from `data`.

The server attaches the raw card string to the response as `json_data`. The client keeps it and sends it back with every edit; this is how unknown fields survive (see [4.1](#41-create-and-edit)).

## 4. Saving character cards

### 4.1 Create and edit

The character editor posts a form to `/api/characters/create` or `/api/characters/edit`. Both call `charaFormatData()` ([`characters.js:565-657`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L565-L657)), which works in this order:

1. Parse `json_data`, the raw card string the client received when it loaded the character, and use it as the starting object. For a new character this is an empty object. The code comment explains the purpose: "This is supposed to save all the foreign keys that ST doesn't care about" ([`characters.js:566-567`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L566-L567)).
2. Overwrite the fields SillyTavern manages with the values from the form: the six V1 fields at both levels, the legacy fields, `spec` and `spec_version`, `data.creator_notes`, `data.system_prompt`, `data.post_history_instructions`, `data.tags`, `data.creator`, `data.character_version`, `data.alternate_greetings`, and the four extension keys.
3. If the form names a linked WI and that WI file exists, replace `data.character_book` with a conversion of the file (see [5.4](#54-life-cycle-of-an-embedded-lorebook)).
4. If the request carries an `extensions` object, deep-merge it into `data.extensions`. The client sends one when creating a character.

Everything not listed in step 2 or 3 is carried over from `json_data` unchanged. This is what preserves unknown keys in `data.extensions` (required by V2), unknown keys in `data` such as the V3 fields (permitted by V3, [`SPEC_V3.md:115`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L115)), and unknown top-level keys.

`spec` is always reset to `chara_card_v2`. A card imported as V3 therefore has `chara_card_v3` in its `chara` chunk until the first edit, and `chara_card_v2` afterwards.

### 4.2 Writing the PNG

Every write goes through `writeCharacterData()`, which re-encodes the image and calls `write()` ([`characters.js:220-265`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L220-L265)). The `ccv3` chunk is produced by relabelling, not by conversion ([`character-card-parser.js:31-42`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js#L31-L42)). Compared with what V3 says about the two chunks:

| V3 statement | SillyTavern |
|---|---|
| The `ccv3` chunk MUST hold a CharacterCardV3 object ([`SPEC_V3.md:24`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L24)) | Holds the hybrid object with a V3 label. It has extra top-level fields and, unless the card was imported with one, no `group_only_greetings`. |
| A `chara` chunk backfilled for V2 readers SHOULD carry a warning in `creator_notes` ([`SPEC_V3.md:26`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L26)) | No warning is added. The `chara` chunk is the primary copy, not a backfill. |
| When backfilling V2, decorators SHOULD be removed from lorebook content ([`SPEC_V3.md:418`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L418)) | Decorators are left in place in both chunks. |

The JSDoc on `write()` still says it "Writes only 'chara', 'ccv3' is not supported and removed", which no longer describes the code ([`character-card-parser.js:8-14`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js#L8-L14)).

### 4.3 Export

The client saves the character and then requests the export ([`script.js:12049-12057`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L12049-L12057)). Two formats are offered, PNG and JSON ([`index.html:6419-6421`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/index.html#L6419-L6421)). There is no CHARX export.

- **PNG** ([`characters.js:1659-1668`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1659-L1668)). The server reads the card with `read()`, which returns the `ccv3` chunk; clears the private fields; and passes the result to `write()`. `write()` stores its input in `chara` and a relabelled copy in `ccv3`. Since the input already carries the V3 label, both chunks end up identical.
- **JSON** ([`characters.js:1669-1679`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1669-L1679)). The server reads the card the same way, refreshes the top-level mirror, clears the private fields, and returns the object with four-space indentation.

The `spec` value in each place:

| Situation | `chara` chunk | `ccv3` chunk |
|---|---|---|
| Stored file, created or edited in SillyTavern | `chara_card_v2` / `2.0` | `chara_card_v3` / `3.0` |
| Stored file, V1 card just imported | `chara_card_v2` / `2.0` | `chara_card_v3` / `3.0` |
| Stored file, V2 card just imported | as imported | `chara_card_v3` / `3.0` |
| Stored file, V3 card just imported | as imported | `chara_card_v3` / `3.0` |
| Exported PNG | `chara_card_v3` / `3.0` | `chara_card_v3` / `3.0` |
| Exported JSON | single object: `chara_card_v3` / `3.0` | |

V2 says `spec` MUST be `chara_card_v2` ([`spec_v2.md:127`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L127)). A reader that checks the `chara` chunk for that value will not recognise a PNG exported by SillyTavern.

An exported card is the full hybrid object. Apart from `chat` being removed and `fav` being cleared, it contains everything in [2.2](#22-the-hybrid-object), including `avatar: "none"`, `create_date`, and `data.extensions.world`.

### 4.4 Other write paths and the validator

- `/api/characters/merge-attributes` deep-merges a partial object into the stored card. Extensions use it through `writeExtensionField()` ([`extensions.js:2070-2120`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/extensions.js#L2070-L2120)). This is the only place `TavernCardValidator` runs ([`characters.js:1276-1304`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1276-L1304)).
- `/api/characters/edit-attribute` sets one field at the top level and under `data` to the same value ([`characters.js:1193-1232`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L1193-L1232)).
- `/api/characters/rename` and `/edit-avatar` rewrite the PNG with an updated or unchanged card.

The validator tries V1 first, then V2, then V3 ([`TavernCardValidator.js:32-48`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js#L32-L48)):

- The V1 check passes if the six V1 keys exist at the top level; values and types are not examined ([`TavernCardValidator.js:55-64`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js#L55-L64)). Cards stored by SillyTavern have those keys, because they are mirrored from `data` on import and written on every save, so validation returns at this step and the V2 and V3 checks never run for them. The repository's own test states this as intended: "prefers V1 when card satisfies both V1 and V2" ([`tavern-card-validator.test.js:59-63`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/tests/tavern-card-validator.test.js#L59-L63)).
- The V2 check requires the exact `spec` and `spec_version`, the fourteen `data` keys, arrays for `alternate_greetings` and `tags`, and, if `character_book` is present, its `extensions` and `entries` ([`TavernCardValidator.js:104-141`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js#L104-L141)). The lorebooks SillyTavern embeds have no `extensions` key (see [5.2](#52-field-mapping-between-the-two-formats)) and would fail this check if it were reached.
- The V3 check requires `spec` to be `chara_card_v3`, `data` to be an object, and `spec_version` not to be below 3.0 or at least 4.0 as a number ([`TavernCardValidator.js:143-168`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js#L143-L168)). A missing or non-numeric `spec_version` converts to `NaN`, for which both comparisons are false, so it passes. No `data` field is checked, including `group_only_greetings`.

## 5. Lorebooks and World Info

### 5.1 Two formats

SillyTavern's lorebook engine works only on WI files. A WI file is an object whose `entries` property is a map from entry id to entry, using SillyTavern's own field names. This is the first entry of the default lorebook:

```jsonc
{
  "entries": {
    "0": {
      "uid": 0,
      "key": ["eldoria", "wood", "forest", "magical forest"],
      "keysecondary": [],
      "comment": "eldoria",
      "content": "...",
      "constant": false, "selective": true,
      "order": 100, "position": 0, "disable": false,
      "probability": 100, "useProbability": true, "depth": 4,
      "group": "", "groupOverride": false, "groupWeight": 100,
      "sticky": 0, "cooldown": 0, "delay": 0,
      "scanDepth": null, "caseSensitive": null, "matchWholeWords": null
      // ...
    }
  }
}
```

The full set of entry fields and their defaults is defined in [`world-info.js:4082-4125`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4082-L4125). The specification format (`CharacterBook` in V2, `Lorebook` in V3) appears only inside cards. Two functions convert between them:

- `convertCharacterBook()` on the client turns an embedded lorebook into a WI object ([`world-info.js:5617-5674`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5617-L5674)).
- `convertWorldInfoToCharacterBook()` on the server turns a WI file's entries into an embedded lorebook ([`characters.js:663-722`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L663-L722)).

### 5.2 Field mapping between the two formats

| Specification field | WI field | Card to WI | WI to card |
|---|---|---|---|
| `keys` | `key` | Copied | Copied |
| `secondary_keys` | `keysecondary` | Copied, default `[]` | Copied |
| `content` | `content` | Copied | Copied |
| `enabled` | `disable` | Inverted | Inverted |
| `insertion_order` | `order` | Copied | Copied |
| `constant`, `selective` | same names | Copied, default `false` | Copied |
| `comment` | `comment` | Copied | Copied |
| `id` | `uid` | Used as the entry key; the array index if absent | Written from `uid` |
| `position` | `position` (0 to 7) | `extensions.position` if present, else `before_char` is 0 and anything else is 1 | 0 becomes `before_char`, every other value becomes `after_char`; the real value goes in `extensions.position` |
| `case_sensitive` | `caseSensitive` | **Not read.** Only `extensions.case_sensitive` is read | Not written at the top level; written as `extensions.case_sensitive` |
| `name` | none | Not read; `comment` is the entry title | Dropped |
| `priority` | none | Not read | Dropped |
| `use_regex` (V3) | none | Not read | Always written as `true` |
| `extensions` | `extensions`, plus individual fields | Kept whole; SillyTavern's keys are also unpacked into fields | Spread back, then SillyTavern's keys are written over it |
| Book `name` | WI file name | Used as the file name | Set to the file name |
| Book `description`, `scan_depth`, `token_budget`, `recursive_scanning`, `extensions` | none | Not converted | Dropped |

Points that follow from the table:

- **The regenerated book has no `extensions`.** `convertWorldInfoToCharacterBook()` returns `{ entries, name }` ([`characters.js:664-665`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L664-L665)). Both specifications make `extensions` a required property of the book, and V2 says unknown keys in it must never be destroyed ([`spec_v2.md:78-82`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L78-L82)).
- **Optional fields are destroyed.** V2 says optional properties may be ignored but "must never be destroyed if already in the data" ([`spec_v2.md:73-76`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L73-L76)), and V3 says an application that does not implement the optional entry fields SHOULD still save them ([`SPEC_V3.md:348`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L348)). `name`, `priority`, top-level `case_sensitive` and the book-level fields do not survive regeneration.
- **`use_regex: true` is written on every entry**, with the comment "ST keys are always regex" ([`characters.js:681`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L681)). In SillyTavern a key is a regular expression only if it is written as `/pattern/flags`; all other keys are plain text ([`world-info.js:337-366`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L337-L366)). V3 defines `use_regex: true` as meaning every key is a pattern, an invalid pattern means no match, and both `secondary_keys` and `constant` SHOULD be ignored ([`SPEC_V3.md:320-324`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L320-L324), [`SPEC_V3.md:304`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L304), [`SPEC_V3.md:344`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L344)). A V3 reader that follows this would treat SillyTavern's plain-text keys as patterns (a key such as `C++` is not a valid one) and would disregard the constant flag and secondary keys of every entry SillyTavern exports.
- **Entry ids must be unique.** The WI object is keyed by `entry.id` ([`world-info.js:5620-5626`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5620-L5626)). If two entries share an id, the later one replaces the earlier one. V3 allows string ids; they are used as they are.
- **`extensions` on entries is preserved**, including keys SillyTavern does not know. SillyTavern adds about thirty keys of its own to it, such as `probability`, `depth`, `selectiveLogic`, `group`, `scan_depth`, `match_whole_words`, `role`, `sticky`, `cooldown`, `delay` and `triggers` ([`characters.js:682-715`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L682-L715)). The WI entry's character filter is not written to the card.

When a WI object is created from a card, the original lorebook is also stored inside the WI file as `originalData` ([`world-info.js:5618`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5618)). It is not used when the card is saved: `charaFormatData()` assigns `originalData` to `character_book` and then, because a WI file always has `entries`, immediately overwrites it with the conversion ([`characters.js:632-640`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L632-L640)). The helpers meant to keep `originalData` in step with edits look entries up by `uid`, a property that entries in the specification format do not have ([`world-info.js:2756-2766`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L2756-L2766)), so `originalData` stays as it was at import time.

### 5.3 Loading and saving World Info files

**Import** ([`world-info.js:5850-5931`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5850-L5931)). The client recognises these inputs and converts them before upload:

| Input | Recognised by | Conversion |
|---|---|---|
| SillyTavern WI JSON | none of the markers below | Uploaded unchanged |
| NovelAI lorebook JSON | `lorebookVersion` property | `convertNovelLorebook()` |
| Agnai memory book | `kind === 'memory'` | `convertAgnaiMemoryBook()` |
| RisuAI lorebook | `type === 'risu'` | `convertRisuLorebook()` |
| PNG | JSON read from the image's `naidata` chunk (NovelAI lorebook image) | Then treated like a JSON file |

The server's only check is that the JSON has an `entries` property ([`worldinfo.js:114-121`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/worldinfo.js#L114-L121)).

The specification formats are not among the recognised inputs:

- The V3 standalone format, `{ spec: 'lorebook_v3', data: Lorebook }` ([`SPEC_V3.md:252-259`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L252-L259)), has no top-level `entries` and is rejected as "not a valid world info file".
- A bare V2 `CharacterBook` has `entries` and is accepted, but nothing converts it. Its entries are in an array with specification field names, so they have no `uid` or `key`, and its string `position` values match none of the numeric positions the prompt builder handles ([`world-info.js:5212-5262`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5212-L5262)). The file is stored, but its entries are not usable as they are.

**Reading** ([`worldinfo.js:17-35`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/worldinfo.js#L17-L35)). The server returns the file's JSON as it is. The client caches it. Missing entry fields are filled in from the defaults when the editor opens the file ([`world-info.js:2104-2136`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L2104-L2136)).

**Saving** ([`worldinfo.js:134-157`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/worldinfo.js#L134-L157)). The editor posts the whole WI object and the server writes it with four-space indentation. The only check is again the presence of `entries`.

**Export** ([`world-info.js:2620-2626`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L2620-L2626)). The export button downloads the WI object as JSON, including `originalData` if present. There is no export to `lorebook_v3` or to a bare `CharacterBook`. A lorebook leaves SillyTavern in specification format only by being embedded in a card.

### 5.4 Life cycle of an embedded lorebook

1. **Import.** The card's `character_book` is stored unchanged. It has no effect on prompts: the character's lore is collected from the WI file named in `data.extensions.world` and from any additional WI files the user has attached to the character in settings ([`world-info.js:4475-4525`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4475-L4525)). Nothing reads `character_book` at generation time.
2. **Prompt to import.** When the character is opened and it has an embedded lorebook but no existing linked WI, SillyTavern asks once whether to import it. If the "Lorebook Import Dialog" setting is off, it shows a notice instead ([`world-info.js:5691-5729`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5691-L5729)). The setting is on by default ([`power-user.js:210`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/power-user.js#L210)). The user can also choose "Import Card Lore" later.
3. **Conversion.** Accepting converts the lorebook with `convertCharacterBook()` and saves it as a WI file named after the book's `name`, or `"{character}'s Lorebook"` if it has none ([`world-info.js:5731-5770`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5731-L5770)). An existing WI file with that name is replaced; the confirmation that mentions this is skipped when the import is started from the dialog in step 2.
4. **Linking.** The new WI is entered in the character editor as the linked WI. The link is written into the card, and the lorebook becomes active, when the character is next saved. The "Import Card Lore" menu item and the globe button trigger that save themselves ([`script.js:12366-12369`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L12366-L12369), [`world-info.js:6328-6330`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L6328-L6330)); the dialog in step 2 does not.
5. **Regeneration.** On every save while the link exists, `charaFormatData()` reads the linked WI file and replaces `character_book` with its conversion ([`characters.js:628-644`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js#L628-L644)). Export saves first, so an exported card always holds the current WI content. Only the linked WI is embedded; additional WI files attached to the character are not.
6. **Unlinking.** Removing the link deletes `character_book` from the card, with a notice ([`world-info.js:6108-6122`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L6108-L6122)).

One side effect of linking by name: if a card arrives with `data.extensions.world` set to a name that already exists among the user's WI files, the card counts as linked to that local file. No import prompt appears, and the next save replaces the card's embedded lorebook with the local file's content.

V2 says "Frontends **MUST** use the character lorebook by default" ([`spec_v2.md:165`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L165)) and V3 repeats it ([`SPEC_V3.md:137`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L137)). In SillyTavern the lorebook is used only after steps 3 and 4, which need the user's agreement.

### 5.5 Run-time behaviour compared with the specifications

The scan is implemented in `checkWorldInfo()` ([`world-info.js:4709-5282`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4709-L5282)).

| Topic | Specification | SillyTavern |
|---|---|---|
| Stacking and precedence | Character lorebook SHOULD stack with the user's lorebook and take precedence ([`spec_v2.md:169`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L169)) | **Conforms.** Entries from chat lore, persona lore, character lore and global lore are combined. Chat and persona lore come first; the order of character and global lore follows a setting whose shipped default is character first ([`world-info.js:4606-4625`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4606-L4625), [`settings.json:21`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/default/content/settings.json#L21)). |
| `enabled` | If false, never a match "IN ANY CASE" ([`SPEC_V3.md:328`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L328)) | **Conforms.** Disabled entries are skipped before any other test ([`world-info.js:4801-4804`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4801-L4804)). |
| `constant` | Always a match ([`SPEC_V3.md:344`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L344)) | **Conforms**, subject to SillyTavern's own filters (probability, timed effects, inclusion groups, budget) ([`world-info.js:4893-4897`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4893-L4897)). |
| `keys` | Match if the chat contains a key; details up to the application ([`SPEC_V3.md:298`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L298)) | **Conforms.** Plain keys are substring matches, with optional whole-word matching. Keys written as `/pattern/flags` are regular expressions. Macros in keys are expanded first ([`world-info.js:4913-4917`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4913-L4917)). |
| `selective`, `secondary_keys` | If `selective`, require a key from both lists ([`spec_v2.md:106-107`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L106-L107)) | **Conforms** by default. An extension field can change the rule to AND ALL, NOT ANY or NOT ALL ([`world-info.js:4924-4989`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4924-L4989)). |
| `insertion_order` | Lower values are inserted earlier ([`spec_v2.md:96`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L96), [`SPEC_V3.md:332`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L332)) | **Conforms** ([`world-info.js:5203-5217`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5203-L5217)). |
| `position` | `before_char` or `after_char` ([`spec_v2.md:109`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L109)) | **Conforms**, with six more positions available through `extensions.position` ([`world-info.js:855-864`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L855-L864)). |
| `content` | Added once; empty content adds nothing ([`SPEC_V3.md:312-316`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L312-L316)) | **Conforms** ([`world-info.js:5207-5210`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L5207-L5210)). |
| `scan_depth` (book) | Only the most recent `scan_depth` messages are searched ([`SPEC_V3.md:275`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L275)) | **Ignored.** The global Scan Depth setting is used (2 by default), or a per-entry override from `extensions.scan_depth` ([`world-info.js:279-297`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L279-L297)). |
| `token_budget` (book) | Lowest-priority entries are removed when the total exceeds the budget ([`SPEC_V3.md:280`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L280)) | **Ignored.** The budget is a percentage of the context size from global settings, 25% by default, with an optional cap ([`world-info.js:4736-4741`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4736-L4741)). Entries are taken source by source and, within a source, from the highest `insertion_order` down; once the budget is reached the rest are left out. Within a source the lowest values are therefore dropped first, which V3 permits when `priority` is absent ([`SPEC_V3.md:334`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L334)). |
| `recursive_scanning` (book) | If false, entry content MUST NOT trigger other entries ([`SPEC_V3.md:284`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L284)) | **Ignored.** A global setting decides, and the shipped default settings turn recursion on ([`settings.json:17`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/default/content/settings.json#L17)). A book with `recursive_scanning: false` is scanned recursively under default settings. |
| `case_sensitive` (entry) | If true, keys SHOULD be case-sensitive ([`SPEC_V3.md:340`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L340)) | **Ignored at the top level.** Matching uses `extensions.case_sensitive`, else the global setting, which is off by default ([`world-info.js:268-271`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L268-L271)). |
| `use_regex` (entry) | If true, keys are patterns ([`SPEC_V3.md:322`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L322)) | **Ignored.** Each key decides for itself by its `/.../` form. |
| `priority` (entry) | MAY decide which entries are dropped at the budget limit ([`SPEC_V3.md:372`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L372)) | **Ignored.** |

### 5.6 Decorators

`parseDecorators()` runs on every entry's content ([`world-info.js:4652-4698`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4652-L4698)). If the content begins with `@@`, the leading lines that begin with `@@` are taken as decorators and removed from the content. The list of known decorators has two members ([`world-info.js:100`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L100)).

| Decorator | SillyTavern |
|---|---|
| `@@activate` | **Implemented.** Activates the entry without checking keys ([`world-info.js:4875-4879`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4875-L4879)). It runs after the disabled, trigger, character-filter, timed-effect and recursion checks, and the entry is still subject to probability and budget. |
| `@@dont_activate` | **Implemented.** Suppresses the entry unless `@@activate` is also present, as specified ([`world-info.js:4881-4884`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L4881-L4884)). |
| `@@@` fallback lines | **Parsed.** A fallback is considered only when the decorator before it was not recognised, with no limit on chain length. |
| `@@depth`, `@@role`, `@@scan_depth`, `@@position`, `@@activate_only_after`, `@@activate_only_every`, `@@keep_activate_after_match`, `@@dont_activate_after_match`, `@@is_greeting`, `@@additional_keys`, `@@exclude_keys`, `@@ignore_on_max_context`, `@@is_user_icon`, `@@disable_ui_prompt`, the `instruct` and `reverse` variants | **Not implemented.** Removed from the content and ignored, which is what V3 asks for unrecognised decorators ([`SPEC_V3.md:380`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L380)). |

Three details of the parser differ from the specification or from what an author might expect:

- V3 says the newlines after the decorators SHOULD be trimmed ([`SPEC_V3.md:318`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L318)). A blank line between the decorators and the text is kept, so the content starts with an empty line.
- If every line of the content is a decorator, nothing is removed and the decorator text itself is used as the content.
- Recognition is by prefix. `@@activate_only_after 4`, `@@activate_only_every 4` and `@@dont_activate_after_match` begin with a known name and are therefore treated as recognised, which makes the parser skip any `@@@` fallback written after them. They still have no effect, because activation tests for the exact strings `@@activate` and `@@dont_activate`.

SillyTavern has its own features that overlap with several decorators: depth and role (position "at depth"), per-entry scan depth, delay, sticky, cooldown, secondary-key logic, and ignoring the budget. They are stored as keys in the entry's `extensions`. Nothing translates between decorators and these keys in either direction, so a lorebook written with decorators does not gain these behaviours in SillyTavern, and one written in SillyTavern does not express them as decorators. The V3 concepts document describes `@@depth` as the replacement for "unstandardized `depth_prompt` field" ([`concepts.md:127-129`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/concepts.md#L127-L129)); SillyTavern continues to use `extensions.depth_prompt` for the Character's Note.

## 6. Prompt-building behaviour

### 6.1 `system_prompt` and `post_history_instructions`

V2 requires that, by default, a non-empty `system_prompt` replaces the user's system prompt and a non-empty `post_history_instructions` replaces the user's post-history ("jailbreak") prompt, and that `{{original}}` inserts the prompt that would otherwise have been used ([`spec_v2.md:137-151`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L137-L151)).

SillyTavern has two user settings, "Prefer Char. Prompt" and "Prefer Char. Instructions", both on by default ([`power-user.js:203-204`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/power-user.js#L203-L204)). With them on:

- **Chat Completion.** The card's text replaces the content of the Main Prompt and of the Post-History Instructions prompt, with `{{original}}` set to the replaced content. The replacement is skipped if the target prompt is marked as not overridable or is disabled for the character ([`openai.js:1495-1513`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/openai.js#L1495-L1513)).
- **Text Completion.** The card's `system_prompt` replaces the user's System Prompt text, and the card's post-history text is appended to the chat as a final user message. Both happen only if the user's System Prompt is enabled; if it is disabled, neither card field is used ([`script.js:4686-4697`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L4686-L4697), [`script.js:4748-4765`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L4748-L4765)).

In both modes `{{original}}` is replaced on its first occurrence only; later occurrences become empty ([`MacroEnvBuilder.js:144-151`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/engine/MacroEnvBuilder.js#L144-L151)). A per-chat system prompt override, if the user has set one, takes precedence over the card's ([`script.js:3415`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L3415)).

### 6.2 Greetings

- **Solo chats.** A new chat starts with `first_mes`, and `alternate_greetings` become swipes on that message, as V2 requires ([`script.js:7710-7742`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L7710-L7742)). If `first_mes` is empty, the first alternate greeting takes its place.
- **Empty greeting.** If `first_mes` is empty and there are no alternates, no opening message is created and the user writes first ([`script.js:7687-7692`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L7687-L7692)). V1 says the chatbot MUST send the first message ([`spec_v1.md:88-89`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L88-L89)).
- **Group chats.** Each member posts one greeting chosen at random from its `first_mes` and `alternate_greetings`, without swipes ([`group-chats.js:578-585`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/group-chats.js#L578-L585)). V3's `group_only_greetings` is not used.

### 6.3 Example messages

`mes_example` is split on `<START>` (case-insensitively), and the marker is added if the text does not begin with one ([`script.js:3501-3515`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L3501-L3515)). The marker is replaced by the example separator in Text Completion and by the "New Example Chat" prompt in Chat Completion. By default, examples are removed block by block as the chat fills the context; the user can instead always include or never include them ([`power-user.js:121-122`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/power-user.js#L121-L122)). This matches V1 ([`spec_v1.md:105-110`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L105-L110)).

### 6.4 Fields that must stay out of the prompt

V2 says `creator_notes`, `creator` and `character_version` MUST NOT be used in prompts and `tags` SHOULD NOT be ([`spec_v2.md:133-192`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L133-L192)). By default SillyTavern uses none of them. It does provide ways to opt in: the macros `{{charCreatorNotes}}` and `{{charVersion}}` ([`env-macros.js:135-141`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/env-macros.js#L135-L141), [`env-macros.js:168-177`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/env-macros.js#L168-L177)), and a per-entry WI option to scan the creator notes for keys ([`world-info.js:314-316`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L314-L316)). Creator notes are shown in the character list and on the character panel, which meets the discoverability recommendation.

Embedded `tags` are not applied automatically. On import the user is asked whether to add them to SillyTavern's own tag list, which is stored in settings, not in the card ([`power-user.js:213`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/power-user.js#L213)). Tag names are compared case-insensitively.

### 6.5 Macros (Curly Braced Syntaxes)

SillyTavern has two macro engines. The newer one is on by default ([`power-user.js:302`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/power-user.js#L302)); the table describes it and notes where the older engine differs.

| Macro | Specification | SillyTavern |
|---|---|---|
| `{{char}}`, `<BOT>` | Character name, case-insensitive ([`spec_v1.md:40-41`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L40-L41)). V3: `nickname` if set, else `name` ([`SPEC_V3.md:572`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L572)) | **Partial.** Replaced case-insensitively, as is `<CHAR>` ([`MacroEngine.js:285-293`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/engine/MacroEngine.js#L285-L293)). Always the character's `name`; `nickname` is never read ([`MacroEnvBuilder.js:133-134`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/engine/MacroEnvBuilder.js#L133-L134)). |
| `{{user}}`, `<USER>` | The user's display name; a default MUST exist ([`spec_v1.md:42-44`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L42-L44)) | **Conforms.** The default is `User` ([`script.js:407`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L407)). |
| `{{original}}` | See [6.1](#61-system_prompt-and-post_history_instructions) | **Conforms**, first occurrence only. |
| `{{random:A,B,C}}` | Random item; `\,` escapes a comma ([`SPEC_V3.md:580`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L580)) | **Conforms** ([`core-macros.js:340-360`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L340-L360), [`core-macros.js:410-422`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L410-L422)). |
| `{{pick:A,B,C}}` | Random item, stable for the same prompt ([`SPEC_V3.md:584`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L584)) | **Conforms.** Seeded by chat, content and position ([`core-macros.js:363-407`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L363-L407)). |
| `{{roll:N}}`, `{{roll:dN}}` | Random number from 1 to N ([`SPEC_V3.md:588`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L588)) | **Conforms.** A bare number becomes `1dN`; other input is passed to the `droll` dice library, which accepts `dN` in either case ([`core-macros.js:303-337`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L303-L337)). |
| `{{// A}}` | Empty string ([`SPEC_V3.md:592`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L592)) | **Conforms** ([`core-macros.js:282-299`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L282-L299)). |
| `{{hidden_key:A}}` | Empty string, but usable for recursive lorebook matching ([`SPEC_V3.md:596`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L596)) | **Not implemented.** Unknown macros are left in the text as written ([`MacroEngine.js:216-218`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/engine/MacroEngine.js#L216-L218)). |
| `{{comment: A}}` | Empty in the prompt; SHOULD be shown to the user as an inline comment ([`SPEC_V3.md:600`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L600)) | **Partial.** Registered as an alias of `{{// }}`, so it is removed; it is not displayed as a comment. The older engine does not recognise it. |
| `{{reverse:A}}` | Reversed text ([`SPEC_V3.md:605`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L605)) | **Conforms** ([`core-macros.js:266-279`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js#L266-L279)). |

Macro names are case-insensitive, as V3 recommends.

## 7. Conformance tables

Each row gives a requirement, its normative level where the specification states one, and what SillyTavern does. The earlier sections hold the detail and the code references.

### 7.1 Character Card V1

| Requirement | Level | SillyTavern |
|---|---|---|
| JSON file embedding | | **Conforms.** Imported and exported. |
| PNG embedding: base64 JSON in `chara` ([`spec_v1.md:20`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L20)) | | **Conforms.** |
| APNG | | **No specific support.** See [3.4](#34-what-import-changes). |
| Six fields present, defaulting to the empty string ([`spec_v1.md:38`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L38)) | MUST | **Conforms when writing.** Lenient when reading: only `name` is needed. |
| `{{char}}`, `<BOT>`, `{{user}}`, `<USER>` replaced case-insensitively ([`spec_v1.md:40-42`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L40-L42)) | MUST | **Conforms.** |
| Default user name ([`spec_v1.md:44`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L44)) | MUST | **Conforms.** |
| `description`, `personality`, `scenario` in every prompt by default | SHOULD | **Conforms** with the default templates. |
| The character sends the first message, and it is `first_mes` ([`spec_v1.md:88-89`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L88-L89)) | MUST | **Conforms** unless `first_mes` is empty; see [6.2](#62-greetings). |
| `<START>` format for examples; pruning when the context fills ([`spec_v1.md:93-110`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v1.md#L93-L110)) | MUST / SHOULD | **Conforms.** |

### 7.2 Character Card V2

| Requirement | Level | SillyTavern |
|---|---|---|
| `spec` is `chara_card_v2`, `spec_version` is `2.0` ([`spec_v2.md:125-131`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L125-L131)) | MUST | **Partial.** True of the `chara` chunk of stored files after an edit. Exports carry the V3 label; see [4.3](#43-export). |
| Fields nested under `data` | | **Partial.** Present under `data`, and also repeated at the top level with additional fields. |
| All required `data` fields present | | **Conforms.** Written on every save. |
| `creator_notes` not used in prompts; easy to find ([`spec_v2.md:135`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L135)) | MUST NOT / SHOULD | **Conforms** by default; opt-in macro exists. |
| `system_prompt` replaces the user's by default; `{{original}}` ([`spec_v2.md:139-143`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L139-L143)) | MUST | **Conforms with conditions.** See [6.1](#61-system_prompt-and-post_history_instructions). |
| `post_history_instructions`, same rules ([`spec_v2.md:147-151`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L147-L151)) | MUST | **Conforms with conditions.** |
| `alternate_greetings` offered as swipes ([`spec_v2.md:157`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L157)) | MUST | **Conforms** in solo chats; random choice in groups. |
| Character lorebook used by default ([`spec_v2.md:165`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L165)) | MUST | **Does not conform.** Requires an import step; see [5.4](#54-life-cycle-of-an-embedded-lorebook). |
| Editors save lorebooks in the specified format ([`spec_v2.md:167`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L167)) | MUST | **Partial.** The entry fields are in the specified shape; the book lacks `extensions` and loses optional fields. |
| Character lorebook stacks with and precedes the world lorebook ([`spec_v2.md:169`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L169)) | SHOULD | **Conforms** with default settings. |
| `tags` not in prompts; case-insensitive filtering ([`spec_v2.md:177-180`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L177-L180)) | SHOULD NOT / SHOULD | **Conforms.** |
| `creator`, `character_version` not in prompts ([`spec_v2.md:184-190`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L184-L190)) | MUST NOT | **Conforms** by default; opt-in macro for the version. |
| `extensions` defaults to `{}` ([`spec_v2.md:196`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L196)) | MUST | **Conforms.** |
| Unknown `extensions` keys not destroyed ([`spec_v2.md:200`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L200)) | MUST NOT | **Mostly conforms** for the card's `extensions` (exception: RisuAI sprite data, [3.4](#34-what-import-changes)) and for entry `extensions`. **Does not conform** for the lorebook's own `extensions` once linked. |
| Keys in `extensions` namespaced ([`spec_v2.md:204`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L204)) | SHOULD | **Does not conform.** |
| Optional lorebook properties never destroyed ([`spec_v2.md:73-76`](https://github.com/malfoyslastname/character-card-spec-v2/blob/8083fb388615ccbce768e97cbbd49d2b3214632c/spec_v2.md#L73-L76)) | must | **Does not conform** once the lorebook is linked. |

### 7.3 Character Card V3

**Embedding**

| Requirement | Level | SillyTavern |
|---|---|---|
| Card in a `ccv3` tEXt chunk as base64 JSON ([`SPEC_V3.md:24`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L24)) | MUST | **Partial.** The chunk is written; its content is the relabelled hybrid object. |
| Prefer `ccv3` over `chara` ([`SPEC_V3.md:28`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L28)) | SHOULD | **Conforms.** |
| Warning in `creator_notes` of a backfilled `chara` chunk ([`SPEC_V3.md:26`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L26)) | SHOULD | **Not done.** |
| `chara-ext-asset_:` chunks ([`SPEC_V3.md:30`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L30)) | MAY | **Not supported**; dropped on import. |
| A JSON file is a CharacterCardV3 object ([`SPEC_V3.md:34`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L34)) | MUST | **Partial.** Same content as the `ccv3` chunk. |
| CHARX ([`SPEC_V3.md:38`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L38)) | | **Import only, partial.** See [3.5](#35-charx). |

**Card object**

| Field or rule | Level | SillyTavern |
|---|---|---|
| Unknown fields ignored, not rejected; may be kept ([`SPEC_V3.md:115`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L115)) | SHOULD / MAY | **Conforms.** Kept through `json_data`. |
| No application fields outside `extensions` ([`SPEC_V3.md:117`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L117)) | MUST | **Does not conform.** |
| `spec_version`; alert on a newer version ([`SPEC_V3.md:125-129`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L125-L129)) | SHOULD | **Not done.** Not checked on import, and rewritten as `3.0`. |
| `nickname` ([`SPEC_V3.md:145`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L145)) | SHOULD | **Not implemented.** Preserved if present. |
| `creator_notes_multilingual` ([`SPEC_V3.md:149`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L149)) | MUST if present | **Not implemented.** `creator_notes` is always shown. Preserved if present. |
| `source` ([`SPEC_V3.md:153-157`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L153-L157)) | SHOULD | **Not implemented.** Preserved if present; SillyTavern does not append to it and derives its own source link from `extensions`. |
| `assets` ([`SPEC_V3.md:163-193`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L163-L193)) | | **Not implemented** apart from CHARX import. The card's own PNG is always the icon, which matches the default `ccdefault:` behaviour. |
| `group_only_greetings` present ([`SPEC_V3.md:197`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L197)) | MUST | **Does not conform.** Never written or used; preserved if present. |
| `creation_date`, `modification_date` ([`SPEC_V3.md:203-207`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L203-L207)) | SHOULD | **Not implemented.** SillyTavern keeps its own top-level `create_date` string and does not update a modification date on export. |

**Lorebook**

| Field or rule | Level | SillyTavern |
|---|---|---|
| Standalone `lorebook_v3` import and export ([`SPEC_V3.md:252-259`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L252-L259)) | SHOULD | **Not implemented.** |
| `use_regex` ([`SPEC_V3.md:322`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L322)) | MUST be boolean | **Not read; always exported as `true`.** |
| `constant` ([`SPEC_V3.md:344`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L344)) | required to implement | **Conforms.** |
| `scan_depth`, `token_budget`, `recursive_scanning` | SHOULD / MUST NOT | **Ignored**; see [5.5](#55-run-time-behaviour-compared-with-the-specifications). |
| `case_sensitive` ([`SPEC_V3.md:340`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L340)) | SHOULD | **Ignored** at the top level. |
| Optional entry fields saved even if unused ([`SPEC_V3.md:348`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L348)) | SHOULD | `id` and `comment` are used. `name` and `priority` are **dropped** once the lorebook is linked. |
| Decorators ([`SPEC_V3.md:374-418`](https://github.com/kwaroran/character-card-spec-v3/blob/f3a86af019fbd99f788f7a1155f399655b34ab35/SPEC_V3.md#L374-L418)) | SHOULD | **Two of them**; see [5.6](#56-decorators). |

## 8. Documentation compared with the code

| Documentation | Code |
|---|---|
| "SillyTavern fully supports Character Cards V2 Specification" ([`Writing-Extensions.md:365`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/For_Contributors/Writing-Extensions.md#L365)) | The V2 deviations are listed in [7.2](#72-character-card-v2). The most significant is that the embedded lorebook is not used by default. |
| No page mentions V3, the `ccv3` chunk, CHARX, BYAF or decorators | The code writes a `ccv3` chunk on every save, imports `.charx` and `.byaf`, and implements two decorators. |
| The character lore insertion strategy "Sorted Evenly" is marked as the default ([`worldinfo.md:73`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/worldinfo.md#L73)) | The shipped default settings and the code default are "Character Lore First" ([`settings.json:21`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/default/content/settings.json#L21), [`world-info.js:80`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js#L80)). |
| A PNG card that is really a WEBP file can be imported by "renaming the card to `.webp`" ([`faq.md:71`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/faq.md#L71)) | The importer accepts only `json`, `png`, `yaml`, `yml`, `charx` and `byaf` ([`script.js:10536-10539`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js#L10536-L10539)). |
| Prompt overrides apply "if the 'Prefer Char. Prompt' user setting is enabled" ([`characterdesign.md:105-106`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/Characters/characterdesign.md#L105-L106)) | Accurate. The further conditions in [6.1](#61-system_prompt-and-post_history_instructions) are not mentioned. |
| "When exporting the character, this file will also get embedded"; "only the primary World Info file gets exported" ([`worldinfo.md:39-41`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/worldinfo.md#L39-L41)) | Accurate. The embedding happens on every save, not only on export, and the fields lost in the conversion are not mentioned. |
| "Character Name is the only required field" ([`characterdesign.md:10`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/Characters/characterdesign.md#L10)) | Accurate for the editor. The stored card always contains all required fields with empty-string defaults. |
| Alternate greetings are swipes; in a group one is chosen at random ([`characterdesign.md:91`](https://github.com/SillyTavern/SillyTavern-Docs/blob/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d/Usage/Characters/characterdesign.md#L91)) | Accurate. |

Two code comments are also out of date: the header of `TavernCardValidator` says "Supported specs: V1, V2" although it has a V3 check ([`TavernCardValidator.js:1-7`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js#L1-L7)), and the JSDoc of `write()` says `ccv3` is not written ([`character-card-parser.js:8-14`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js#L8-L14)). The server's type definitions cover V2 only ([`spec-v2.d.ts:1-52`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/types/spec-v2.d.ts#L1-L52)).

## 9. What survives a round trip

"Linked" means the embedded lorebook has been imported as a WI file and linked to the character. An empty cell means nothing further changes at that stage.

| Data in the imported card | After import | After an edit, lorebook not linked | After an edit, lorebook linked | In an export |
|---|---|---|---|---|
| V1 and V2 fields | Kept | Kept or updated from the editor | Same | Present at both levels |
| `spec` / `spec_version` | `chara` as imported; `ccv3` set to V3 / `3.0` | `chara` reset to V2 / `2.0` | Same | V3 / `3.0` everywhere |
| Unknown keys in `data.extensions` | Kept (RisuAI sprite arrays moved to files) | Kept | Kept | Kept |
| V3 fields in `data` (`nickname`, `source`, `assets`, `group_only_greetings`, dates, multilingual notes) | Kept, unused | Kept | Kept | Kept |
| Unknown top-level keys | Kept (V2/V3 cards); dropped (V1 cards) | Kept | Kept | Kept |
| `name` containing characters not allowed in file names | Altered | | | |
| `character_book` as a whole | Kept verbatim, inactive | Kept verbatim | **Replaced** by the conversion of the WI file | Whatever the last save produced |
| Book `description`, `scan_depth`, `token_budget`, `recursive_scanning`, `extensions` | Kept | Kept | **Lost** | |
| Entry `name`, `priority`, top-level `case_sensitive` | Kept | Kept | **Lost** | |
| Entry `use_regex` | Kept | Kept | **Set to `true`** | |
| Entry `extensions` | Kept | Kept | Kept, with SillyTavern keys added | |
| Decorators in entry content | Kept | Kept | Kept (they are part of `content`) | Kept in both chunks |
| PNG metadata other than the card; asset chunks | **Lost** (image re-encoded) | | | |
| CHARX assets | Image assets extracted to folders, except `user_icon`; non-image assets **lost** | | | Not exported |

## 10. Consequences for card authors and other tools

- **Reading SillyTavern exports.** Expect extra top-level keys beside `spec`, `spec_version` and `data`. Do not rely on the `chara` chunk saying `chara_card_v2`. Do not assume a card labelled `chara_card_v3` has `group_only_greetings`.
- **Reading SillyTavern lorebooks.** The book may have no `extensions` property. Treat `use_regex: true` as unreliable: keys are regular expressions only when written as `/pattern/flags`. The entry's real position, case sensitivity, depth, role and similar settings are in `extensions` under SillyTavern's key names.
- **Writing cards for SillyTavern users.** The embedded lorebook works only after the user imports it. Book-level scan depth, token budget and recursion settings are not honoured, and top-level `case_sensitive` is not read; to control case sensitivity or scan depth per entry, the values must be in `extensions.case_sensitive` and `extensions.scan_depth`. Of the decorators, only `@@activate` and `@@dont_activate` do anything. `nickname` and `group_only_greetings` have no effect.
- **Exchanging standalone lorebooks.** SillyTavern neither reads nor writes `lorebook_v3`. A lorebook must be in SillyTavern's WI format, or embedded in a card, to be usable.

## Appendix: code index

**Server**

| File | Contents |
|---|---|
| [`character-card-parser.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/character-card-parser.js) | `read()`, `write()`, `parse()`: PNG `chara` and `ccv3` chunks |
| [`characters.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/characters.js) | Import, create, edit, export, list; `charaFormatData()`, `readFromV2()`, `convertToV2()`, `convertWorldInfoToCharacterBook()` |
| [`TavernCardValidator.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/validator/TavernCardValidator.js) | V1, V2 and V3 structural checks |
| [`charx.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/charx.js) | CHARX parsing and asset extraction |
| [`byaf.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/byaf.js) | Backyard AI archive import |
| [`worldinfo.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/worldinfo.js) | WI file list, read, import, save, delete |
| [`sprites.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/sprites.js) | `importRisuSprites()` |
| [`content-manager.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/endpoints/content-manager.js) | Import from external sites |
| [`spec-v2.d.ts`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/src/types/spec-v2.d.ts) | V2 type definitions |

**Client**

| File | Contents |
|---|---|
| [`script.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/script.js) | Character editor, import and export requests, greetings, prompt assembly, macro entry points |
| [`world-info.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/world-info.js) | WI engine and editor; `convertCharacterBook()`, `checkWorldInfo()`, `parseDecorators()`, `importWorldInfo()` |
| [`openai.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/openai.js) | Chat Completion prompt assembly and card prompt overrides |
| [`group-chats.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/group-chats.js) | Group greetings and combined card fields |
| [`core-macros.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/core-macros.js), [`env-macros.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros/definitions/env-macros.js) | Macro definitions (newer engine) |
| [`macros.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/macros.js) | Older macro engine |
| [`char-data.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/char-data.js) | Type definitions for the client-side character object |
| [`extensions.js`](https://github.com/SillyTavern/SillyTavern/blob/06bde939fb1e9c4c8d8641d810f0a916b5bce127/public/scripts/extensions.js) | `writeExtensionField()` |
