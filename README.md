# Japanese-to-English Game Translation: An Agent's Working Guide

This README is a reusable operating guide for AI agents translating Japanese games into English. It records lessons supported by the local **Fairy Tail: Portable Guild 2 (Japan, v1.01)** PSP translation project and turns them into a process other agents can adapt.

**Evidence boundary:** this guide was reconstructed from the project's README, tools, manifests, validation reports, and crash investigation. It does not claim a complete playthrough or independent retesting of the translation. Project-specific facts below are observations from those records; recommended practices are instructions for future work. File extensions, offsets, encodings, and layout limits must be rediscovered for each game.

The requested filename is `Agent.md`. Agents should explicitly read it when starting work; projects that require automatic discovery through `AGENTS.md` can reference or adapt this document to their own conventions.

## 1. What this project taught us

Translation is a pipeline: identify the correct assets, understand their structure, translate with context, rebuild every reference, and verify the result in play. An English sentence in a JSON file proves only that a draft exists.

| Finding in this project | General lesson |
| --- | --- |
| The project README reports 6,718 localized catalog entries and 6,618 intentionally preserved entries. | Count player-facing coverage separately from internal, test, and preserved data. |
| Some opening-scene dialogue existed only inside filesets. | A main archive directory can omit resources used by individual scenes. |
| Compressed assets appeared in multiple archive locations. | Update every relevant copy and its metadata. |
| Some English assets exceeded their original compressed allocations. | Rebuild archive layout instead of forcing longer text into adjacent data. |
| Translating `システム` to `System` broke a program comparison and crashed a reward scene. | Display text can also function as a program identifier. |
| Dialogue, item descriptions, diary pages, trophies, and quests required different limits. | Layout belongs to the destination screen, not to the language as a whole. |
| Changing the name-entry labels alone was insufficient. | A keyboard must insert the correct characters as well as display them. |
| Old emulator states and optional installed data could bypass current changes. | Runtime testing must establish which code and assets actually loaded. |

The local README labels v0.18 an experimental draft. Much wording remains unreviewed, several later changes have static checks without in-game confirmation, and a complete playthrough has not been performed. Preserve those qualifications in any handoff.

## 2. Start with a baseline and a small proof

Before bulk translation:

1. Read project instructions and existing notes. Inventory completed work before repeating extraction or translation.
2. Record the exact game title, region, revision, disc/product ID, source image size, and SHA-256. A filename is not a reliable revision identifier.
3. Keep the original image and extracted source assets unchanged. Write candidates into separate build directories.
4. Boot the unmodified game using an isolated emulator profile. Record emulator version, graphics backend, install-data setting, cheats, texture replacements, and any compatibility changes.
5. Capture representative Japanese screens: title, name entry, dialogue with a portrait, inventory, quest details, save/load, and an error or confirmation dialog.
6. Identify one low-risk string, trace it to its resource, translate it, rebuild, cold-boot, and confirm it on screen.
7. Prove a no-change extraction/rebuild before scaling up. Compare payloads and semantics where compression or padding prevents byte-identical output.

The first milestone is a repeatable edit-to-screen path. If a one-string change cannot be reproduced, bulk work will hide the problem.

Use local game dumps and keep distributable deliverables separate from source images. Patch packaging is described later; publication is a separate action from preparing a build.

## 3. Recommended tools and when to use them

These are practical choices, not requirements to install everything. The local project demonstrates Python-based extraction, rebuilding, image conversion, and PPSSPP testing. Other tools below are recommendations for similar work.

| Task | Tools | Practical use |
| --- | --- | --- |
| Inventory and binary reconnaissance | `rg`, `file`, `xxd`, `strings`, an installed 7-Zip CLI | Find notes and signatures; list image contents; inspect small byte ranges. Raw string scans generate candidates, not a trustworthy catalog. |
| Structured extraction and rebuilding | Python 3: `struct`, `pathlib`, `zlib`, `hashlib`, `json` | Make binary layout, byte order, compression, checksums, and bounds explicit. |
| Encoding checks | Python codecs | Decode and encode strictly. Python documents CP932 and other Japanese codecs in its [codec reference](https://docs.python.org/3/library/codecs.html). |
| Runtime testing | PPSSPP | Use isolated profiles, logging, debugger access, and screenshots. Its [developer tools](https://www.ppsspp.org/docs/development/developer-tools/) include remote debugging and decrypted EBOOT dumping. |
| Program analysis | Ghidra plus the PPSSPP debugger | Follow string references and inspect control flow. [Ghidra's official repository](https://github.com/NationalSecurityAgency/ghidra) provides the framework and setup instructions. Confirm processor and executable format for the target. |
| Texture inspection | Pillow and a format-specific decoder/encoder | Decode to PNG, edit lettering, restore native format, then decode again. Pillow's [image concepts](https://pillow.readthedocs.io/en/stable/handbook/concepts.html) cover palette and alpha modes. |
| Linguistic work | A bilingual glossary, Japanese dictionaries, contextual AI assistance, and a Japanese-proficient reviewer | Draft quickly, then resolve ambiguity against source and gameplay. Machine output remains a draft. |
| Changes and provenance | Git where available, JSON manifests, SHA-256 | Track scripts, catalogs, decisions, and exact build identity. Avoid committing large game images or private saves. |
| Patch delivery | xdelta3 | Produce and verify a binary delta for an exact source image. See the [official command-line guide](https://github.com/jmacd/xdelta/blob/main/site/docs/commandline.md). |

Pin versions used for release builds. Prefer documented service APIs for online translation. The local prototype uses a Google Translate endpoint; do not assume that endpoint is stable, supported, or appropriate for a different project. Check existing authorization before transmitting text or incurring costs; do not introduce redundant approval when already authorized.

## 4. Organize the project so another agent can resume

A useful structure is:

```text
project/
  Agent.md                   # process, guardrails, and known lessons
  README.md                  # current user-facing build status
  source/                    # untouched extracted disc files
  extracted/                 # untouched decompressed resources
  manifests/                 # paths, IDs, offsets, sizes, hashes, duplicates
  catalogs/                  # Japanese source, English, context, review status
  glossary/                  # names, terminology, style decisions
  overrides/                 # edited native resources
  tools/                     # extraction, encoding, rebuilding, validation
  graphics/                  # previews, editable assets, import manifests
  tests/                     # targeted fixtures and regression definitions
  emulator/                  # isolated configuration and local checkpoints
  build/                     # generated candidates and reports
  release/                   # patch, checksums, and player instructions
```

This is a suggested structure. The existing project keeps several catalogs and manifests directly in `FTPG2-English/`; do not move them without adapting the scripts.

Keep source data, editorial decisions, and generated output distinct. Record each override's source hash and rationale. Avoid undocumented manual hex edits: the next clean build must reproduce every shipped change.

## 5. Discover the complete resource graph

### Inspect contents, not just names

An extension can be misleading. Here, `data.apk` is a game PACK archive; treating it as an Android application would be wrong. Inspect magic values and container structure first.

The local parser recognizes `ENDILTLE`, `PACKTOC `, and fileset structures such as `PACKFSLS` and `PACKFSHD`. It handles zlib-compressed payloads and records archive entries. These signatures are specific to this game.

For each discovered resource, record:

- Logical path or stable resource ID, parent archive, and duplicate locations.
- Absolute or relative offset and the base used to interpret it.
- Stored size, decompressed size, compression method, and alignment.
- Flags, checksums, byte order, and any external index references.
- Source hash and the parser/version used to extract it.

Validate ranges before reading: offsets and sizes must fit their containing buffer; pointer tables must fit their chunks; terminators must be found inside valid string regions. Recognizing a signature is a hypothesis until structural checks pass.

### Search every storage layer

Inventory main archives, scene filesets, nested archives, scripts, executables, textures, fonts, movies, optional install data, and alternate menu variants. Track each as translated, intentionally preserved, out of scope, or unknown.

In this project the opening scene includes `fileset-extra/quest_001_fileset/0005.gop`. A main-directory-only catalog would miss it. See `extract_fileset_extras.py` and `append_fileset_text.py` before assuming extraction is complete.

Hashing identical stored payloads helped locate repeated copies here. Generalize carefully: the same decompressed content can use different compression, and near-identical resources may have distinct IDs or behavior. Preserve identity relationships instead of merging records solely because text matches.

## 6. Extract indexed strings and classify them

Prefer actual string tables over guessing boundaries with a Japanese-byte regex. The local GOP exporter reads `GENESTRT` tables using counts and relative offsets, then retains the file, table base, string index, source offset, Japanese text, English field, and status.

For new projects, extend that record with context:

```json
{
  "id": "quest_001:dialogue:17",
  "file": "scene/example.bin",
  "table": 4096,
  "index": 17,
  "offset": 8192,
  "source": "Japanese source retained verbatim",
  "source_hash": "hash-of-original-string-bytes",
  "english": "English draft",
  "speaker": "known speaker or unknown",
  "screen": "portrait_dialogue",
  "layout_profile": "dialogue_portrait",
  "status": "draft",
  "review_notes": "ambiguity, evidence, and next action"
}
```

The numbers above are illustrative. Use a stable logical ID when offsets can change during rebuilding. Keep pristine offsets for source verification rather than treating rebuilt offsets as permanent identity.

Classify each candidate before translating:

| Category | Default action |
| --- | --- |
| Player-facing sentence or label | Translate with context and a layout profile. |
| Internal key, script symbol, path, schema name | Preserve unless all uses are understood. |
| Text that also participates in comparisons | Trace consumers; update dependencies together or preserve it. |
| Keyboard data | Inspect display labels and inserted-character references separately. |
| Diagnostic text | Track separately; preserve formatting and test relevant paths where feasible. |
| Test data, filters, unused assets, ambiguous binary fragments | Preserve with a reason until investigated. |

The existing source exporter overwrites `text_catalog.json`. **Do not run it over reviewed translations.** Export into a separate file and merge by verified identity when refreshing a catalog. Existing extraction/export scripts are examples to inspect, not universal commands to run blindly.

## 7. Treat encoding and control syntax as part of the format

### Establish the actual encoding

The local text pipeline uses CP932. That does not establish CP932 for another game: alternatives include Shift-JIS variants, UTF-16, custom glyph IDs, and mixed binary/text structures.

Decode representative samples strictly and compare the result to what the game renders. Reject undecodable data rather than silently replacing bytes. Encoding success also does not prove that a font contains the requested glyph.

Track three different budgets:

1. **Encoded bytes** for buffers, string slots, and terminators.
2. **Visible glyph width** for lines and panels.
3. **Compressed bytes** for archive storage.

`len(text)` measures none of those reliably in every game. A CP932 character can occupy two bytes; a control token can occupy bytes while drawing no letters; proportional letters can have different widths.

The local encoder normalizes punctuation and some unsupported characters before CP932 encoding. Make such transformations explicit and reviewable. Never use lossy encoding to conceal unsupported text or modify source identifiers indiscriminately.

### Parse tokens before sending text to a translator

Local syntax includes printf arguments, face controls, voice controls, button codes, newlines, and the `#p` page delimiter. Their exact grammar must come from the engine and observed data.

A robust flow is:

1. Tokenize the source into text and typed controls.
2. Replace protected controls with unique markers that cannot collide with source text.
3. Translate text with entry ID, speaker, adjacent lines, glossary, and screen constraints.
4. Restore each marker exactly once.
5. Reject missing, duplicated, invented, or semantically misplaced controls.
6. Verify the encoded result, not just the editable English field.

For non-positional printf arguments, preserve argument order and type. Preserve positional arguments correctly when the engine supports them. Keep escaped percent signs distinct from arguments. A generic regex rarely covers every format specifier or game command.

The local validator checks several token families separately. For a new validator, also compare the **combined ordered token stream** where ordering affects behavior. Matching counts or separate per-family sequences can miss a face or voice command moved across another command. Permit new page breaks only under an explicit layout rule; do not silently remove source behavior controls.

Add small fixtures for malformed markers, escaped percent signs, repeated controls, empty strings, and commands adjacent to punctuation. Never let a failed translation fall through as a successful empty entry.

## 8. Translate meaning before shortening for layout

Build a glossary early: character names, place names, abilities, equipment classes, element names, UI verbs, and preferred abbreviations. Separate established franchise terminology from uncertain romanization. Record evidence and unresolved choices rather than inventing certainty.

For each scene or related group:

1. Read surrounding Japanese lines and establish speaker, listener, tone, and situation.
2. Draft natural English while preserving mechanics and narrative information.
3. Check omitted Japanese subjects, politeness, gender assumptions, jokes, idioms, and ambiguous names.
4. Verify quantities, negation, conditions, rewards, timing, and button actions against the source.
5. Fit the destination panel without dropping requirements or changing game rules.
6. Review wording in the game and retain separate linguistic and visual review statuses.

Japanese frequently omits subjects. Do not invent a pronoun or relationship merely because English wants one. Short labels can also be ambiguous without the menu: `セット` may mean equip, assign, or set depending on its action.

Use caching and bounded batches for drafts. Include context and glossary version in future cache keys; identical Japanese can need different English in different screens. The local cache is keyed by source text, which is convenient but insufficient for contextual ambiguity. Split failed batches to isolate bad entries, save progress, and retain failure logs.

Keep states such as `draft`, `linguistically_reviewed`, `layout_checked`, and `runtime_verified` separate. Mark intentionally preserved entries with a reason. The absence of Japanese characters is a useful check only for selected text; it is not proof of semantic accuracy or whole-game completion.

## 9. Build a layout profile for every screen family

A single wrapper failed because panels had different space. The local project records these working budgets:

| Screen | Recorded project budget | Qualification |
| --- | --- | --- |
| Story dialogue | 32 visible columns, 3 rows per page; story scale 87.5% | Controls do not consume ordinary printed columns. |
| Multiline item descriptions | 36 characters per row, 2 rows | Preserve reviewed explicit breaks. |
| Mabel diary pages | Reviewed pages use up to 24 characters, 8 rows | The encoder permits 9 rows; document and resolve this mismatch before treating it as a universal guarantee. |
| Trophy descriptions | 14 characters per row, 4 rows | Narrow columns beside artwork. |
| Quest details | 22 characters per row, 5 rows | Titles and objectives use another profile. |
| Short quest fields | 26 characters, single line | Shared labels can affect multiple screens. |

These are character-count rules from this project, not measured pixel limits for other games. Several later versions explicitly lack visual verification.

For each new game, measure available space with portraits, icons, borders, and dynamic values present. Test long names, large numbers, and maximum-length substitutions. Use glyph advances when available; otherwise adopt conservative limits and verify screenshots at native game scale.

Wrap text without splitting commands or detaching a portrait/voice control from its intended content. Check long unbroken words and token-bearing words explicitly. Preserve editorial line breaks for reviewed fixed panels. Paginate only when the engine supports it and the new page retains the intended timing and controls.

Prefer concise, faithful wording. Font reduction or renderer patches are separate changes requiring their own tests; shrinking every screen can make text unreadable and affect unrelated UI.

## 10. Rebuild structures instead of overwriting adjacent data

### String-table rebuilding

The local GOP builder reconstructs string data and offsets, preserves indices, restores null terminators, aligns chunks, updates size fields, and reparses the result. This works because relevant records refer to string indices in the examined format. Another engine may use direct addresses or offsets into strings.

Before choosing a rebuilding strategy, establish whether references are indices, file offsets, chunk-relative offsets, runtime pointers, or shared suffixes. Preserve unknown fields. Assert that untouched entries retain their content and that every replacement resolves through the rebuilt table.

### Archive rebuilding

The early in-place prototype rejected compressed overflows. The later repacker relocates payloads and rebuilds metadata with 2,048-byte alignment for this game.

Required checks include:

- Recalculate stored and decompressed sizes, offsets, and container sizes.
- Update every modified primary and fileset copy.
- Preserve resource IDs and ordering where lookup depends on them.
- Synchronize archive metadata and external indexes such as `pack.idx`.
- Recalculate format-required checksums; here filesets use MD5.
- Reopen output using its newly written metadata and decompress every relevant payload.
- Compare decoded output to the intended replacements and untouched source resources.

MD5 is used here for format compatibility; use SHA-256 for provenance. Matching lengths alone cannot prove correct content. A successful `zlib.decompress` cannot prove the correct asset was associated with the correct ID.

### ISO rebuilding

Writing a larger archive into a disc image also requires valid file extents and size metadata. Do not assume trailing zeros are available space. The local finalizer checks the next file boundary, updates file lengths and volume sizes, and rereads inserted bytes for comparison.

If growth cannot fit safely, use a disc-image rebuilding approach compatible with the target game and verify any LBA assumptions. Do not reuse this game's ISO routine for arbitrary images. Validate directory records, volume bounds, duplicated metadata where present, and neighboring file integrity.

## 11. Executable text and the bridge-crash lesson

Search executable-embedded strings separately from data catalogs. Obtain the correct executable representation, verify its hash and format, and distinguish file offsets from virtual addresses. PPSSPP's developer tools can dump decrypted EBOOT data; the local patcher expects a particular ELF dump, not an arbitrary encrypted `EBOOT.BIN`.

For every fixed-slot replacement:

1. Assert the exact original bytes at the expected location.
2. Establish slot capacity from structure and references, including the terminator.
3. Encode strictly and reject overflow.
4. Preserve argument and control semantics.
5. Pad only within verified free bytes.
6. Report the offset, old/new bytes or text, capacity, and source/output hashes.
7. Compare binary differences against the approved change ranges.

### What actually went wrong

The recorded bridge failure occurred after equipping magic gear. The translated speaker label was `System`, but the program still compared it with Japanese `システム`. The mismatch selected the character-voice path for a system reward sound and caused an invalid memory access at program counter `0x08adde7c`.

The fix changed the matching executable literal at file offset `0x30a324`. The crash report records reproduction before the correction and passage through the scene into the next area's boss afterward. These addresses identify this build only.

**Reusable rule:** before translating speaker labels, event names, or category strings, inspect comparisons and call sites. Some strings have both presentation and logic roles. Change all dependencies together or preserve the logic value and localize display separately if the engine supports that.

Renderer or code changes need additional care: processor instructions, calling convention, preserved registers, branch delay slots, relocation behavior, segment mappings, and overwritten instructions all matter. The local story-scale patch adds a MIPS wrapper and adjusts ELF metadata. Do not transplant its offsets or code into another revision. Keep code patches separate from wording edits so failures can be isolated.

## 12. Text in images, palettes, fonts, and input screens

No text search finds lettering baked into a texture. Produce contact sheets and inspect title labels, Yes/No buttons, portraits, headings, tutorials, and alternate states.

The local GIM workflow preserves texture dimensions, indexed palettes, alpha, swizzle order, and headers/footers. Some title labels use 128×16 images with 4-bit indices. The generated PNG is an intermediate asset, not the game's native texture.

A reliable artwork loop is:

1. Decode the original native texture into a viewable image.
2. Identify the exact lettering region and preserve surrounding art.
3. Create English lettering with adequate margins; inspect spelling manually.
4. Quantize to the native palette and alpha constraints.
5. Restore stride, nibble order, swizzling, mipmaps, and other required metadata.
6. Decode the newly encoded texture and inspect that result at native size.
7. Compare unchanged pixels or regions against the original and record hashes.
8. Repack every applicable copy and inspect it in the game.

For known, repeated labels, deterministic font rendering may be easier to reproduce than generated artwork. If image generation is used, preserve the prompt, original output, crop/scale recipe, and import manifest. The existing title workflow does this in `graphics/title-art-workflow.md`.

If English glyphs are absent, inspect the font atlas, encoding-to-glyph map, metrics, and fallback behavior before inserting new characters. A texture replacement pack can help prototype appearance, but it must be disabled when verifying that native assets were patched.

The name-entry screen requires an end-to-end test: type uppercase and lowercase Latin letters, switch pages and modifiers, delete characters, confirm, save, and reload. Here the fix reuses real Latin character records and redirects modifier variants. Relabeling Japanese keys would leave Japanese input underneath English-looking buttons.

## 13. Validate in layers and report each layer honestly

| Layer | What to verify | What passing does not prove |
| --- | --- | --- |
| Source identity | Exact image/executable hashes and original bytes | Translation accuracy. |
| Catalog | Stable IDs, intended coverage, encoding, controls, review status | Correct visual fit. |
| Native resources | Tables, references, decoded replacements, unchanged data | Correct runtime selection. |
| Archives | Every copy, offsets, sizes, alignment, decompression, checksums | Functional gameplay. |
| Disc image | File boundaries, embedded bytes, image structure | Boot or story completion. |
| Runtime smoke test | Cold boot, menus, loading, representative rendering | All quests or branches. |
| Regression test | Previously failing paths and affected shared screens | Untested scenes and hardware. |
| Linguistic/playthrough review | Meaning, tone, mechanics, progression | Other modes or platforms unless tested. |

Use a fresh process and normal in-game saves for executable changes. An emulator save state can restore old code, assets, or settings into memory. States are useful for targeted diagnosis only when their provenance and any debugger modifications are documented; they are insufficient as the only release test.

Keep optional install data disabled for this project: its untranslated installed archive can bypass modified disc assets. Do not delete normal saves to solve that problem. Document the exact supported install-data configuration for each game.

Keep cheat-assisted access and debugger-based experiments distinct from normal gameplay evidence. Record build hash, emulator configuration, checkpoint, action sequence, expected result, screenshot/log, and outcome for every important reproduction.

### Minimum runtime matrix

- Cold boot, title labels, new game, and normal save/load.
- Name entry, confirmation dialogs, longest expected names, and dynamic substitutions.
- Dialogue with portraits, voice controls, page transitions, and long messages.
- Equipment changes, item use, inventory descriptions, and reward sequences.
- Quest acceptance, abandonment, objectives, failure, completion, and area transitions.
- Diary, trophy, statistics, and other narrow panels.
- Multiplayer, disconnect prompts, and supported error paths if included in scope.
- Previously reported crashes plus nearby preceding and following events.
- Representative late-game scenes and ending, followed by a complete playthrough before claiming completion.

When PSP hardware compatibility is a goal, test hardware separately. Emulator success alone does not establish it.

## 14. Debug by changing one cause at a time

| Symptom | Investigate first |
| --- | --- |
| English exists in the catalog but Japanese appears | Wrong image/build; fileset-only resource; duplicate; installed data; texture lettering; old state. |
| Crash after a translated label | String comparisons, speaker/event identifiers, control sequence, format arguments. |
| Menus work but one scene stays Japanese | Scene-specific filesets or alternate resource variants. |
| Text overlaps or disappears | Destination profile, explicit breaks, portrait area, glyph widths, missing glyphs. |
| Garbled texture | Palette format, alpha, swizzle, stride, packing order, native dimensions. |
| English keys insert Japanese | Input character IDs still reference Japanese records. |
| New ISO behaves like an older build | State restore, wrong path, mounted image, installed data, active texture replacements. |
| Archive opens but a scene fails | Incorrect resource ID/reference, stale index, missed checksum or duplicate, content corruption despite correct length. |

Reproduce on a known baseline; compare logs and binary changes; isolate text, graphics, and program modifications. Keep the smallest failing case. Verify a suspected fix on the failing path and neighboring behavior before broadening it.

## 15. Continuing the existing FTPG2-English project

Read `FTPG2-English/README.md` for current status before rebuilding. Relevant implementation references are:

| Area | Existing files under `FTPG2-English/` |
| --- | --- |
| Main extraction and indexed text | `tools/extract_archive.py`, `tools/export_text.py` |
| Fileset-only discovery | `tools/extract_fileset_extras.py`, `tools/append_fileset_text.py`, `fileset_manifest.json` |
| Draft generation and glossary | `tools/translate_catalog.py`, `translation_cache.json`, `translation_failures.json` |
| Text encoding and validation | `tools/build_patch.py`, `tools/validate_catalog.py` |
| Full archive rebuild | `tools/repack_archive.py` |
| Executable localization | `tools/patch_program.py`, `tools/localize_diagnostics.py` |
| Native artwork and keyboard | `tools/inspect_gim.py`, `tools/import_*`, `tools/keyboard_layout.py` |
| Disc-image finalization | `tools/finalize_iso.py` |
| Regression evidence | `build/crash_investigation.json`, `emulator/layout-lab/` |
| Layout changes | `tools/fix_*_layout.py`, `build/*layout*json` |

The documented rebuild sequence, **after deliberate catalog/override edits**, is:

```bash
cd /home/Jazneo/Downloads/games/FTPG2-English
python3 tools/validate_catalog.py
python3 tools/repack_archive.py
python3 tools/patch_program.py
python3 tools/finalize_iso.py vNEXT --program-patch
```

Replace `vNEXT` with a new candidate version. Close the game before rebuilding and stop at any failure; run commands individually or use a runner that stops on nonzero status. The scripts rely on existing source assets, manifests, and a particular emulator-dumped executable. These commands do not create those prerequisites or perform emulator testing.

Do not run `build_patch.py` as the release builder: its standalone main is the earlier capacity-limited prototype, although other scripts import its helpers. Do not rerun extraction or translation services merely to rebuild edited text. Regenerate artwork overrides through their specific importers when artwork sources change.

Some local validators use Python `assert`; run them without optimization flags that disable assertions. For reusable production tools, prefer explicit validation exceptions and nonzero exits for release-critical checks.

After building, inspect reports, run an installed 7-Zip image test, verify the final image hash, then perform targeted cold-boot/runtime tests. A stored `emulator_tested: false` field must remain false until that exact candidate has evidence. This guide itself did not rebuild or retest the game.

## 16. Package a patch and a reproducible release

For a future release, prepare a patch against one exact source revision and independently apply it to a clean copy. Example xdelta3 usage, with illustrative paths:

```bash
xdelta3 -e -s "original.iso" "translated.iso" "english.xdelta"
xdelta3 -d -s "original.iso" "english.xdelta" "verification.iso"
sha256sum "translated.iso" "verification.iso"
```

The reconstructed and tested images must have identical hashes. Record patcher version and options. Reject wrong-source hashes before patching instead of relying on filenames. Retain the original locally; deliver the patch and project materials appropriate to the release scope.

A release README should state source revision/hash, patch instructions, resulting image hash, emulator/hardware configurations tested, install-data requirements, save/state caveats, translated and untranslated surfaces, linguistic review level, known defects, and exact runtime coverage.

Use language that matches the evidence: “static validation passed,” “cold boot verified,” “bridge regression passed,” or “complete playthrough verified.” These are different achievements. Do not label a machine draft as a finished localization.

## 17. Handoff checklist for the next agent

- [ ] Original hashes and source revision recorded; source assets preserved.
- [ ] Extraction/rebuild format documented, including unknown fields and assumptions.
- [ ] Main, fileset-only, executable, graphics, font, movie, and install-data coverage tracked.
- [ ] Catalog identities, Japanese source, English, context, and preservation reasons retained.
- [ ] Glossary, ambiguous readings, abbreviations, and style decisions documented.
- [ ] Control grammar, formatting semantics, encoding, and layout profiles validated.
- [ ] Rebuilt resources and every relevant duplicate checked using output metadata.
- [ ] Code patches guarded by exact source bytes and hashes.
- [ ] Native artwork decoded after re-encoding; actual input behavior tested.
- [ ] Exact candidate cold-booted with known configuration and no stale state.
- [ ] Crash regressions and affected shared screens tested with evidence.
- [ ] Untested modes, scenes, hardware, and linguistic review clearly identified.
- [ ] Patch round-trip produces the tested image, if preparing a release.
- [ ] Remaining work is actionable: resource/entry IDs, reproduction, owner or next step.

Leave a concise handoff recording the latest build hash, successful checks, current failures, unresolved translation questions, and the next reproducible test. The durable lesson is to keep meaning, binary structure, visual layout, and runtime behavior connected through evidence at every step.
