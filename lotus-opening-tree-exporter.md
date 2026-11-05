---
description: Extracts a user's Lotus Chess opening repertoire and training positions into legally validated, print-ready variation trees, including early deviations and user-supplied continuations.
mode: all
---

# Lotus Opening Tree Exporter

Turn the user's Lotus Chess training data into printable opening trees. Preserve
their repertoire choices and requested playing colour. Distinguish recovered
training data, selected opening prefixes, reconstructed connections, and moves
supplied by the user. Deliver the requested files, not just an explanation of how
to create them.

## Scope and defaults

- Use the conversation to identify the openings, playing colour, and desired
  output. Ask only about consequential details that are still missing.
- Treat a Lotus course as potentially broader than its strict opening name.
  Include earlier opponent deviations, not just descendants of the textbook
  defining position.
- Keep Caro-Kann for White separate from Caro-Kann for Black. Position side to
  move alone does not establish membership in a particular course.
- Do not equate discovered, trained, selected, and mastered positions. Verify
  progress-field semantics before applying a mastery filter.
- Default print layout: A4 landscape, one opening tree per page, black on white,
  small fixed-width move text, and aligned branch connectors.
- The PDF contains **only opening names and trees**: no dates, account details,
  explanations, counts, legends, page numbers, or footers. Keep extraction notes
  and missing-data details in a separate file and the final response.

## 1. Locate accessible data

Prefer the user's existing local app data or a supported export. An account email
alone does not provide account access. Do not request a password for local reads.

On macOS, the observed Lotus container is:

```text
~/Library/Containers/com.lotuschess.app/Data/Documents/app_data/
```

Discover the installed application by bundle identifier rather than assuming its
display name or executable path:

```sh
mdfind 'kMDItemCFBundleIdentifier == "com.lotuschess.app"'
```

One observed installation was `/Applications/Runner.app`; this is not a portable
constant. Its Flutter assets were under:

```text
Wrapper/Runner.app/Frameworks/App.framework/flutter_assets/assets/databases/
```

Relevant assets include `opening_moves.txt`, `opening_names.txt`, and
`database.bin`. The first two can help identify opening prefixes and named
positions. `database.bin` was observed with a `LOTUSDB6` header: do not assume it
is SQLite or pretend to have decoded it.

If reads fail with `Operation not permitted`, explain that macOS privacy access
is blocking the app container. Ask the user to grant Full Disk Access to the
application hosting the tool process, relaunch it if necessary, and retry the
actual read. Do not repeatedly request permission after access works.

Read Lotus data without changing it. Prefer a stable copy for extraction; detect
incomplete frames or changing files and retry from a consistent snapshot. Keep
account tokens and unrelated preferences out of logs, artifacts, and repositories.

## 2. Decode the stored records

Inspect these files before assuming there is no training data:

| File | Observed role |
| --- | --- |
| `opening_position_states.hive` | FEN keys and JSON position-state values |
| `system_white.hive` | Selected White repertoire prefixes |
| `system_black.hive` | Selected Black repertoire prefixes |
| `mastered_positions.hive`, `discovered_positions.hive` | May be empty legacy stores |

`position.hive`, `trainings.hive`, `pgns.hive`, and
`repetition_confidences.hive` were also empty in the investigated installation.
Empty legacy files are not proof that the account has no progress.

The observed Hive files were append-only frames:

1. A little-endian 32-bit frame length, including the frame header and checksum.
2. A key type: `0` for a 32-bit integer key, or `1` for a string key with a
   one-byte byte length followed by UTF-8 bytes.
3. A typed value, unless this is a deletion frame.
4. A four-byte checksum trailer.

Observed value types were `4` for a UTF-8 string with a 32-bit byte length and
`9` for a string list with a 32-bit count followed by length-prefixed strings.
The selected-repertoire values were string lists, **not plain strings**; the
first item held a space-separated UCI opening prefix.

Verify the format against the installed version. Check frame and string bounds,
validate checksums using the matching Hive format when available, and reject
unknown or malformed values instead of silently skipping them. Apply frames in
order, retain the latest value per key, and remove deleted keys. A `strings` scan
is useful for reconnaissance but is not an authoritative export.

Position-state JSON contained fields such as `d`, `m`, `s`, and `u`. The `u`
field held a UCI move; other fields looked like timestamps. Do not infer their
exact learning or mastery meanings merely from their names.

## 3. Reconstruct the repertoire

Use a legal-move library, such as `chess.js`, to replay positions and generate SAN.

1. Choose a root early enough to cover the requested course deviations. For the
   Scotch course investigated here, use `1.e4 e5 2.Nf3`, rather than requiring
   `2...Nc6 3.d4 exd4 4.Bc4` before traversal starts.
2. Use selected repertoire prefixes to bridge opening moves absent from the
   position-state store. Record their provenance separately from training states.
3. At a supported repertoire-side position, play the stored response, enumerate
   legal opponent replies, and connect replies that reach supported saved
   positions. Generalize the traversal to the requested colour.
4. Normalize position identity by board, side to move, castling rights, and
   legally relevant en-passant availability. Ignore move counters for lookup, but
   derive displayed move numbers from the actual replayed path. Do not merge
   positions with different legal en-passant options.
5. Lotus used king-to-rook castling coordinates such as `e1h1` and `e1a1`.
   Convert these to the library's standard-chess castling representation only
   after verifying the moving piece and legality. Handle Black analogously;
   do not apply this conversion blindly to Chess960.
6. Preserve transpositions as valid alternative paths with path-local cycle
   detection. Do not globally suppress a position and thereby hide another line.
7. Stop when no supported continuation exists. Do not fill gaps with engine
   recommendations or generic opening theory and call them the user's training.
8. Track uncovered saved positions, missing responses, and uncertain course
   membership. A graph reconstructed from global position states cannot prove
   full course coverage or mastery.

Merge user-provided continuations into the appropriate branch after validating
every move. Mark them as user-supplied in the intermediate data and extraction
notes; never describe them as recovered from the app. Keep existing sibling
branches when adding or correcting a line.

### Session-derived regression examples

These examples describe the original user's data, not universal repertoire
recommendations. Use them when maintaining that export; do not inject them into
another user's repertoire without evidence.

- `1.e4 e5 2.Nf3 Nc6 3.d4 d6 4.d5 Nb8 5.Nc3 Nf6 6.Be3`.
- `1.e4 e5 2.Nf3 Nc6 3.d4 d6 4.d5 Nce7 5.c4 f5 6.exf5 Bxf5
  7.Nc3 Nf6 8.Bg5 Ng6`.
- User-supplied continuation: `1.e4 e5 2.Nf3 d6 3.d4 exd4 4.Nxd4
  Nf6 5.Nc3 Be7 6.Be2`.
- Preserve the existing `3...exd4 4.Bc4` subtree when adding early deviations.
- The investigated Caro-Kann White tree included `1.e4 c6 2.d4 d5 3.e5`,
  with branches `3...Bf5 4.h4` and `3...c5 4.c3`.

## 4. Produce the printable trees

Keep a structured intermediate tree with positions, moves, children, and source
provenance. Generate the PDF from that tree; do not edit rendered move strings
as the primary source of truth.

- A practical implementation uses Node.js, `chess.js`, and `pdfkit` in an
  isolated working directory. Do not depend on a previous session's temporary
  directory or install dependencies into the Lotus application.
- Print the opening name followed by its starting moves and connected branches.
  Use standard SAN, including captures, checks, promotion, disambiguation, and
  `O-O` / `O-O-O`.
- Use a fixed-width font around 8–9 pt. Calculate available width, row height,
  and page bounds before rendering; verify the resulting layout actually fits.
- Compact consecutive single-child moves onto a line. Wrap only between moves
  or move pairs and indent continuations consistently. Align sibling connectors
  such as `+-`, `` `- ``, and `|`; a last-child continuation must not grow a
  phantom vertical connector.
- Maintain one opening per landscape page. If the tree cannot fit legibly, ask
  about larger paper or splitting the tree instead of silently clipping moves
  or reducing the type to unreadable sizes.
- An unavailable continuation ends at its last supported move. Explain the gap
  outside the PDF rather than adding commentary to the tree.
- Default output directory: `~/Documents/Lotus Opening Trees/`. Provide the PDF
  and separate extraction notes. Plain-text or HTML companions are optional;
  regenerate them consistently if they already exist.

## 5. Verify and deliver

Before reporting completion:

1. Replay every exported line and validate every move and displayed move number.
2. Check requested colours and explicit user examples, including early
   deviations and both sibling knight retreats in the regression examples.
3. Confirm user-supplied lines appear exactly as supplied, with their provenance
   retained separately.
4. Confirm the expected page count, landscape dimensions, readable font size,
   and absence of clipped branches. Inspect the generated PDF, not just the
   generation script or its success message.
5. Confirm the PDF has only opening names and trees, with no notes or footers.
6. Report the output path or clickable file link and any remaining missing
   continuation briefly. Do not claim the export covers every course line or
   identifies mastered positions unless that was independently established.
