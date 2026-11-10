# character-card-spec

An analysis of how [SillyTavern](https://github.com/SillyTavern/SillyTavern) implements the Character Card specifications (V1, V2 and V3): how it loads and saves character cards and lorebooks, and where its behaviour differs from what the specifications say.

**Read the analysis: [sillytavern-character-card-analysis.md](sillytavern-character-card-analysis.md)**

## Main findings

- SillyTavern stores every card as a PNG with two text chunks, `chara` and `ccv3`. Both hold the same object: the V2 structure, with the V1 fields and SillyTavern's own fields repeated at the top level.
- V3 is supported as a label only. The `ccv3` chunk is the same object with `spec` changed to `chara_card_v3`. None of the V3 card fields are used, and `group_only_greetings`, which V3 requires, is never written.
- A card's embedded lorebook is not used until the user imports it as a World Info file. After that, the embedded copy is regenerated from that file on every save, which drops several specification fields.
- Book-level lorebook settings (`scan_depth`, `token_budget`, `recursive_scanning`) and the entry fields `case_sensitive`, `use_regex` and `priority` are ignored. Of the V3 decorators, only `@@activate` and `@@dont_activate` are implemented.
- Imported cards are not validated, CHARX is import-only, and standalone lorebooks can be exchanged only in SillyTavern's own format.

## Sources

The analysis is based on reading the source at the commits below; SillyTavern was not executed. Every code and specification reference in the analysis is a permalink to one of these commits.

| Repository | Content | Commit |
|---|---|---|
| [malfoyslastname/character-card-spec-v2](https://github.com/malfoyslastname/character-card-spec-v2/tree/8083fb388615ccbce768e97cbbd49d2b3214632c) | V1 and V2 specifications | `8083fb388615ccbce768e97cbbd49d2b3214632c` |
| [kwaroran/character-card-spec-v3](https://github.com/kwaroran/character-card-spec-v3/tree/f3a86af019fbd99f788f7a1155f399655b34ab35) | V3 specification | `f3a86af019fbd99f788f7a1155f399655b34ab35` |
| [SillyTavern/SillyTavern](https://github.com/SillyTavern/SillyTavern/tree/06bde939fb1e9c4c8d8641d810f0a916b5bce127) | Implementation (`release` branch, v1.19.0) | `06bde939fb1e9c4c8d8641d810f0a916b5bce127` |
| [SillyTavern/SillyTavern-Docs](https://github.com/SillyTavern/SillyTavern-Docs/tree/ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d) | Documentation (`main` branch) | `ed5423d809c6b10b8c1e9ef4a0c8ddc7f2ef7f3d` |
