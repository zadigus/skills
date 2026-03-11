---
name: hungarian-verb-trainer
description: Use when the user wants to practice Hungarian verb conjugation, when they ask to train magyar igék, or when they want to drill verb forms in Hungarian
---

# Hungarian Verb Trainer

## Overview

Train the user on Hungarian verb conjugation through interactive quiz sessions. The agent picks random verbs and asks the user to conjugate them in specific tenses, persons, and conjugation types (definite/indefinite). Verbs the user doesn't know are added to their Anki deck.

## Prerequisites: AnkiConnect Setup

The skill requires the **AnkiConnect** addon to add cards to Anki programmatically.

**To install AnkiConnect:**
1. Open Anki
2. Go to **Tools > Add-ons > Get Add-ons...**
3. Enter addon code: **2055492159**
4. Click OK, restart Anki

AnkiConnect runs a REST API on `http://localhost:8765` while Anki is open. **Anki must be running** during training sessions for card additions to work.

**Verify it works:**
```bash
curl -s http://localhost:8765 -X POST -d '{"action":"version","version":6}'
```
Expected response: `{"result":6,"error":null}`

## Session Flow

```dot
digraph session {
    rankdir=TB;
    start [label="User requests\nverb training", shape=doublecircle];
    check_anki [label="Check AnkiConnect\nis reachable", shape=box];
    ensure_deck [label="Ensure deck\n'magyar igek' exists", shape=box];
    ask_tense [label="Ask: which tense(s)?", shape=box, style=filled, fillcolor="#ccffcc"];
    ask_count [label="Ask: how many\nquestions?", shape=box, style=filled, fillcolor="#ccffcc"];
    pick_verb [label="Pick random verb +\nperson + conj. type", shape=box];
    ask_user [label="Present question\nto user", shape=box];
    check_answer [label="User answers", shape=diamond];
    correct [label="Confirm correct\nMove to next", shape=box, style=filled, fillcolor="#ccffcc"];
    wrong [label="Show correct form\nExplain pattern", shape=box, style=filled, fillcolor="#ffcccc"];
    add_anki [label="Add verb to Anki\n(if not already there)", shape=box, style=filled, fillcolor="#ffffcc"];
    more [label="More questions?", shape=diamond];
    results [label="Save results to\n~/Documents/magyar/\nverb-training-results/", shape=box];
    done [label="Session complete", shape=doublecircle];

    start -> check_anki;
    check_anki -> ensure_deck;
    ensure_deck -> ask_tense;
    ask_tense -> ask_count;
    ask_count -> pick_verb;
    pick_verb -> ask_user;
    ask_user -> check_answer;
    check_answer -> correct [label="correct"];
    check_answer -> wrong [label="wrong"];
    wrong -> add_anki;
    correct -> more;
    add_anki -> more;
    more -> pick_verb [label="yes"];
    more -> results [label="no"];
    results -> done;
}
```

### 1. Check AnkiConnect

Before starting, verify AnkiConnect is reachable:

```bash
curl -s http://localhost:8765 -X POST -d '{"action":"version","version":6}'
```

If not reachable, tell the user to open Anki and install AnkiConnect (see Prerequisites).

### 2. Ensure Deck Exists

Create the deck "magyar igek" if it doesn't exist:

```bash
curl -s http://localhost:8765 -X POST -d '{
  "action": "createDeck",
  "version": 6,
  "params": {"deck": "magyar igek"}
}'
```

### 3. Bulk-Add Verbs to Anki (First Session)

On the first session (or when verbs are missing from the deck), add all verbs from verbs.md to Anki. Use the `multi` action to batch:

```bash
curl -s http://localhost:8765 -X POST -d '{
  "action": "addNote",
  "version": 6,
  "params": {
    "note": {
      "deckName": "magyar igek",
      "modelName": "Basic (and reversed card)",
      "fields": {
        "Front": "lenni",
        "Back": "to be\n\nÉn magyar vagyok."
      },
      "options": {
        "allowDuplicate": false
      },
      "tags": ["magyar", "ige"]
    }
  }
}'
```

**Card format:**
- **Front:** Hungarian infinitive (e.g., `lenni`)
- **Back:** English translation + blank line + A1 example sentence
- **Model:** `Basic (and reversed card)`
- **Tags:** `magyar`, `ige`

Check if a verb already exists before adding:
```bash
curl -s http://localhost:8765 -X POST -d '{
  "action": "findNotes",
  "version": 6,
  "params": {"query": "deck:\"magyar igek\" front:lenni"}
}'
```

### 4. Ask Tense Selection

Present these options to the user:

| Option | Hungarian | Description |
|--------|-----------|-------------|
| 1 | Jelen idő | Present tense |
| 2 | Múlt idő | Past tense |
| 3 | Jövő idő | Future tense (fog + infinitive, except lenni) |
| 4 | Feltételes mód | Conditional mood |
| 5 | Felszólító mód | Imperative / subjunctive mood |
| 6 | Minden | All tenses (random selection each question) |

### 5. Ask Session Length

Ask the user how many questions they want (e.g., 10, 20, 50).

### 6. Run the Quiz

For each question:
1. Pick a **random verb** from verbs.md
2. Pick a **random person**: én, te, ő/ön, mi, ti, ők/önök
3. Pick a **random conjugation type**: alanyi (indefinite) or tárgyas (definite)
4. Pick a **random tense** (from the user's selection)
5. Present the question clearly:

**Format:**
```
[Q3/20] Conjugate "látni" (to see)
Tense: jelen idő (present)
Person: én (I)
Conjugation: tárgyas (definite)
```

6. Wait for the user's answer
7. Check correctness:
   - If **correct**: confirm and move on
   - If **wrong**: show the correct form, briefly explain the pattern (vowel harmony, ik-verb, etc.), and add the verb to Anki if not already there

## Hungarian Conjugation Reference

### Persons

| Person | Hungarian | English |
|--------|-----------|---------|
| E/1 | én | I |
| E/2 | te | you (informal) |
| E/3 | ő / ön | he/she/it / you (formal) |
| T/1 | mi | we |
| T/2 | ti | you (plural) |
| T/3 | ők / önök | they / you (formal plural) |

### Conjugation Types

- **Alanyi (indefinite):** used when there is no definite direct object (e.g., "I read a book" -- olvasok egy könyvet)
- **Tárgyas (definite):** used when the direct object is definite (e.g., "I read the book" -- olvasom a könyvet)

### Tense/Mood Overview

**Jelen idő (present):** Base conjugation. Back-vowel vs front-vowel endings differ.

**Múlt idő (past):** Add `-t`/`-tt`/`-ott`/`-ett`/`-ött` to stem + personal endings.

**Jövő idő (future):** `fog` + infinitive for most verbs.
- **Exception: lenni** -- uses its own future forms: leszek, leszel, lesz, leszünk, lesztek, lesznek (no `fog` needed).

**Feltételes mód (conditional):** Add `-na`/`-ne`/`-ná`/`-né` to stem + personal endings.

**Felszólító mód (imperative/subjunctive):** Add `-j`/`-jj` (with consonant assimilation rules) + personal endings.

### Key Irregularities to Know

| Verb | Type | Notes |
|------|------|-------|
| lenni | highly irregular | Special forms in all tenses. Future: leszek, not *fog lenni |
| menni | irregular | megyek, mész, megy... |
| jönni | irregular | jövök, jössz, jön... |
| enni | irregular | eszek, eszel, eszik... |
| inni | irregular | iszok, iszol, iszik... |
| venni | irregular | veszek, veszel, vesz... |
| vinni | irregular | viszek, viszel, visz... |
| hinni | irregular | hiszek, hiszel, hisz... |
| tenni | irregular | teszek, teszel, tesz... |

### Important Note on Correctness

You (the agent) must **know the correct conjugation** to verify the user's answer. Hungarian conjugation has many irregular verbs and subtle rules. If you are unsure of a form:
- State clearly that you're not 100% certain
- Provide your best answer with a caveat
- Never silently accept a wrong answer or reject a correct one

## Results Tracking

At the end of each session, save a results file to `~/Documents/magyar/verb-training-results/`. Create the directory if it doesn't exist.

**Filename format:** `YYYY-MM-DD-HHmm-results.md`

**Content format:**

```markdown
# Verb Training Results - YYYY-MM-DD HH:MM

## Summary
- **Tense(s):** [selected tense(s)]
- **Questions:** [total]
- **Correct:** [count] ([percentage]%)
- **Wrong:** [count] ([percentage]%)

## Detailed Results

| # | Verb | Tense | Person | Conj. | Your Answer | Correct Answer | Result |
|---|------|-------|--------|-------|-------------|----------------|--------|
| 1 | látni | jelen idő | én | tárgyas | látom | látom | ✅ |
| 2 | menni | múlt idő | ő | alanyi | ment | ment | ✅ |
| 3 | enni | jelen idő | te | alanyi | esz | eszel | ❌ |

## Verbs to Review
- enni (to eat) -- struggled with present tense alanyi forms
```

## Adding a Verb to Anki Mid-Session

When the user gets a verb wrong and it's not already in the deck:

```bash
curl -s http://localhost:8765 -X POST -d '{
  "action": "addNote",
  "version": 6,
  "params": {
    "note": {
      "deckName": "magyar igek",
      "modelName": "Basic (and reversed card)",
      "fields": {
        "Front": "[infinitive]",
        "Back": "[english]\n\n[example sentence]"
      },
      "options": {
        "allowDuplicate": false
      },
      "tags": ["magyar", "ige"]
    }
  }
}'
```

If the verb is already in the verb list, use the translation and example from there. If it's a verb not in the list (user-provided), ask the user for the English meaning and compose a simple A1 example.

## Common Mistakes (Agent)

| Mistake | Fix |
|---------|-----|
| Not checking AnkiConnect before starting | Always verify connectivity first |
| Adding duplicate cards to Anki | Check with `findNotes` before `addNote` |
| Using `fog lenni` for future of lenni | lenni has its own future: leszek, leszel, lesz... |
| Accepting wrong answers silently | Always verify against known conjugation tables |
| Not saving results at session end | Always save to ~/Documents/magyar/verb-training-results/ |
| Asking all questions at once | One question at a time, wait for user's answer |
| Mixing up alanyi and tárgyas | Clearly state which conjugation type in each question |
