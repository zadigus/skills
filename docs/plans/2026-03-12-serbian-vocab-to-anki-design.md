# Serbian Vocab-to-Anki Skill Design

## Problem

Serbian lesson documents (.docx) contain red-highlighted words marking unknown vocabulary. These need to be extracted, lemmatized, translated, and added to Anki as flashcards with reversed cards.

## Architecture

Single-pass pipeline: Extract red words -> Lemmatize + classify -> Translate -> Check Anki duplicates -> Add cards

## Pipeline Steps

### 1. Extract red-highlighted words
- Use `python-docx` to scan all runs for `<w:highlight w:val="red"/>`
- Collect raw text of each matching run

### 2. Lemmatize + classify (agent knowledge)
- **Nouns/adjectives/adverbs:** nominative singular (dictionary form)
- **Verbs:** infinitive form. Identify aspectual pair (imperfective + perfective). Both on front card with `(imperf)` / `(perf)` annotations
- **Multi-word expressions:** keep as-is if idiomatic
- **Slang:** mark with `(sleng)` annotation

### 3. Translate to English
- Agent provides English translation for Back field

### 4. Duplicate check + add to Anki
- `findNotes` to check existing cards in "Konverzacija Srpski"
- `addNote` with model `Basic (and reversed card)`
- Tags: `srpski`, `vokabular` (+ `sleng` if applicable)
- `allowDuplicate: false` as safety net

## Card Format

| Front (Serbian) | Back (English) |
|---|---|
| `povezati (perf) / povezivati (imperf)` | `to connect, to link` |
| `smaračica (sleng)` | `annoying woman, pest (f.)` |
| `pakao` | `hell` |

## Technical Details

- Red highlight detection: `<w:highlight w:val="red"/>` in run XML properties
- AnkiConnect API: `http://localhost:8765` (must be running)
- Deck: "Konverzacija Srpski"
- Model: "Basic (and reversed card)"
- python-docx required (`pip3 install python-docx`)

## Design Decisions

- Fully automatic (no user confirmation per word)
- Trust all red highlighting (no filtering of exercise instructions)
- No modification of the .docx file
- Agent handles lemmatization using linguistic knowledge (no external NLP library)
