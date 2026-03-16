---
name: hungarianpod101-to-lingq
description: Use when scraping audio or dialogue transcripts from HungarianPod101 lesson pathways, when downloading dialogue MP3s from Innovative Language sites, when uploading audio lessons to LingQ via API, or when dealing with CDN bot protection on hungarianpod101.com
---

# HungarianPod101 to LingQ Pipeline

## Overview

Download dialogue MP3s from HungarianPod101 lesson pathways, scrape Hungarian-only dialogue transcripts from lesson pages, and upload both to LingQ as a private course with proper text content.

**Do NOT rely on Whisper for Hungarian transcripts** - it hallucinates on ~60% of short dialogue clips. Scrape transcripts from the website instead.

## Key Challenges

1. **Bot protection**: HungarianPod101 triggers reCAPTCHA after ~8-16 requests
2. **CDN requires Referer header**: Without `Referer: https://www.hungarianpod101.com/`, CDN returns 403
3. **LingQ API v2 is obsolete**: Must use v3
4. **Headless browser fails**: The SPA detects headless mode and shows "Loading" forever. Must use `headless=False`
5. **Whisper is unreliable for Hungarian**: See "Whisper Learnings" section

## Architecture

```
HungarianPod101 lesson pages
  → Playwright (headless=False) + pycookiecheat (extract dialogue URLs + transcripts)
  → CDN direct download (with Referer header)
  → LingQ API v3 upload (multipart form with Hungarian transcript text)
```

## Step 1: Extract Chrome Cookies

```python
from pycookiecheat import chrome_cookies
cookies = chrome_cookies("https://www.hungarianpod101.com")
# Key cookies: PHPSESSID, guid, guidmember, _amember_ru, _amember_rp
```

User must be logged into HungarianPod101 in Chrome first.

## Step 2: Scrape Lesson Pages for Dialogue URLs and Transcripts

### Playwright with Non-Headless Mode (Required)

The site is a JavaScript SPA. Headless mode (including with playwright-stealth) gets stuck on "Loading" due to bot detection. **Must use `headless=False`**.

```python
from playwright.async_api import async_playwright
from pycookiecheat import chrome_cookies

cookies = chrome_cookies("https://www.hungarianpod101.com")

async with async_playwright() as p:
    browser = await p.chromium.launch(headless=False)
    context = await browser.new_context()

    # Inject cookies from Chrome
    await context.add_cookies([
        {"name": k, "value": v, "domain": ".hungarianpod101.com", "path": "/"}
        for k, v in cookies.items()
    ])

    page = await context.new_page()
    await page.goto(lesson_url, wait_until="networkidle", timeout=60000)
```

### Batch Processing (Critical)

CAPTCHA triggers after ~8-16 page loads per browser context. Strategy:
- Process in batches of 8
- Create a **new browser context** for each batch
- Wait 10-15 seconds between batches
- Save progress after each batch (resume on failure)

### Extracting Dialogue URL from Page

The dialogue MP3 link is in the lesson page HTML. Look for links containing `_dialog.mp3`:

```python
links = await page.query_selector_all("a[href*='_dialog.mp3']")
# Or search audio player elements for dialog URLs
```

### Extracting Dialogue Text from Page

Dialogue text is in `<td>` elements on the lesson page. The structure is always **4 blocks of N lines**:
1. Hungarian dialogue lines
2. English translation lines
3. Hungarian dialogue (repeated)
4. English translation (repeated)

So the first `total_td_count / 4` text elements are the Hungarian dialogue.

```python
tds = await page.query_selector_all('td')
td_texts = []
for td in tds:
    text = (await td.inner_text()).strip()
    if text and len(text) > 1:
        td_texts.append(text)

# First quarter = Hungarian dialogue
hungarian_count = len(td_texts) // 4
hungarian_lines = td_texts[:hungarian_count]
```

**Important**: The `<td>` elements include speaker labels (A:, B:) as separate elements. You need to merge them into proper dialogue format:

```python
# Merge speaker labels with their lines
dialogue_lines = []
i = 0
while i < len(hungarian_lines):
    if hungarian_lines[i] in ('A:', 'B:', 'C:'):
        if i + 1 < len(hungarian_lines):
            dialogue_lines.append(f"{hungarian_lines[i]} {hungarian_lines[i+1]}")
            i += 2
        else:
            i += 1
    else:
        dialogue_lines.append(hungarian_lines[i])
        i += 1
dialogue = "\n".join(dialogue_lines)
```

**Alternative**: If the first-quarter approach produces too many lines (includes vocab/examples), manually verify against the audio and trim.

## Step 3: CDN URL Patterns (Fallback)

If scraping fails, dialogue URLs follow predictable patterns:

| Series | Pattern |
|--------|---------|
| Absolute Beginner | `https://mdn.illops.net/hungarianpod101/ABS_S1L{N}_{MMDDYY}_{hungpod101\|hpod101}_dialog.mp3` |
| Beginner | `https://mdn.illops.net/hungarianpod101/B_S1L{N}_{MMDDYY}_{hungpod101\|hpod101}_dialog.mp3` |

- Absolute Beginner dates: weekly Tuesdays starting Jan 2012
- Beginner dates: weekly Mondays starting June 2013 (some share dates)
- Host suffix alternates between `hungpod101` and `hpod101`

Verify with HEAD request + Referer header before downloading.

## Step 4: Download MP3s

```python
headers = {"Referer": "https://www.hungarianpod101.com/"}
r = requests.get(dialog_url, headers=headers, stream=True)
# No auth cookies needed for CDN, just the Referer header
```

## Step 5: Upload to LingQ

### API v3 (v2 is obsolete)

Upload with the scraped Hungarian dialogue as the lesson text:

```python
url = "https://www.lingq.com/api/v3/{lang}/lessons/"
headers = {"Authorization": "Token {api_key}"}
data = {
    "title": f"L{num:02d} - {lesson_title}",
    "text": hungarian_dialogue,              # Scraped Hungarian-only dialogue
    "collection": "{course_id}",             # String, not int
    "status": "private",
    "level": "1",
}
with open(audio_path, "rb") as f:
    files = {"audio": (os.path.basename(audio_path), f, "audio/mpeg")}
    r = requests.post(url, headers=headers, data=data, files=files, timeout=120)
# Expect 201 on success
```

### Delete Lessons (if re-uploading)

```python
r = requests.delete(
    f"https://www.lingq.com/api/v3/{lang}/lessons/{lesson_id}/",
    headers={"Authorization": "Token {api_key}"}
)
# Expect 204 on success
```

### Create Course First

```python
r = requests.post(
    "https://www.lingq.com/api/v3/{lang}/collections/",
    headers={"Authorization": "Token {api_key}"},
    json={"title": "Course Title", "description": "..."}
)
course_id = str(r.json()["id"])
```

### Rate Limiting

Add 1-second delay between uploads. LingQ API is generally tolerant but don't hammer it.

## Quick Reference

| Item | Value |
|------|-------|
| CDN host | `mdn.illops.net` |
| CDN auth | Referer header only (no cookies) |
| LingQ API base | `https://www.lingq.com/api/v3` |
| LingQ Hungarian lang code | `hu` |
| CAPTCHA threshold | ~8-16 requests per context |
| Lesson 1 ("All About") | No dialogue audio (intro lesson) |
| Headless mode | **Does NOT work** - must use `headless=False` |
| Dependencies | `pycookiecheat`, `playwright`, `requests` |
| Working directory | `/Users/lmichel/hungarianpod101_dialogues/` |
| Level 1 course ID | `2660048` (50 lessons, L02-L51) |

## Whisper Learnings (Do Not Use for Hungarian Dialogues)

Whisper (openai-whisper) was tested for transcribing Hungarian dialogue MP3s (~20-30 second clips). **Results were too unreliable** - scraping transcripts from the website is the correct approach.

### Model Comparison for Hungarian

| Model | Quality | Speed | Notes |
|-------|---------|-------|-------|
| `base` | Poor | Fast | Garbled output |
| `small` | Reasonable | Medium | Some errors |
| `medium` | Bad | Slow | Collapsed/repeated output |
| `turbo` | Mixed | Fast | Best speed/quality ratio but **60% hallucination rate** on short clips |
| `large-v3` | Best | Very slow (CPU) | Recovered most `turbo` failures but still hallucinated on 7/50 files |

### Key Parameters

- `language='hu'` - Must be specified explicitly, auto-detect often picks wrong language
- `condition_on_previous_text=False` - Did NOT help with hallucinations
- `no_speech_threshold`, `compression_ratio_threshold` - Did NOT help
- `initial_prompt` - Can force output but risks fabricating content

### Common Hallucinations

Whisper frequently outputs these instead of actual dialogue content:
- "A TORONTOS KÖZÖN" (repeated)
- "Köszönöm szépen!" (repeated)
- Random Hungarian words unrelated to audio

### Conclusion

For short Hungarian audio clips, **always scrape transcripts from the source website** rather than using Whisper. The hallucination rate is unacceptably high even with `large-v3`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using LingQ API v2 | Returns `["API is obsolete. Use v3 instead."]`. Use v3. |
| Forgetting Referer header on CDN | 403 error. Always send `Referer: https://www.hungarianpod101.com/` |
| Using `requests` directly for lesson pages | Bot-blocked. Must use Playwright with injected cookies. |
| Using `headless=True` | SPA gets stuck on "Loading" forever. Must use `headless=False`. |
| Using `playwright-stealth` | Not needed and doesn't fix the headless issue. Just use `headless=False`. |
| Not rotating browser contexts | CAPTCHA after ~8-16 pages. Create new context per batch. |
| Sending `collection` as int to LingQ | Must be string in multipart form data. |
| Downloading full lesson audio instead of dialogue | Look for `_dialog.mp3` suffix specifically. |
| Using Whisper for Hungarian transcripts | 60% hallucination rate. Scrape from website instead. |
| Using `is_hungarian()` heuristic for dialogue extraction | Fails on names like "Anne Smith". Use structural pattern (first quarter of TD elements). |

## File Formats

### Lesson Metadata (`lesson_metadata.json`)

```json
[
  {
    "num": 2,
    "title": "Finding New Friends in Hungary",
    "dialog_url": "https://mdn.illops.net/hungarianpod101/ABS_S1L1_010312_hungpod101_dialog.mp3",
    "filepath": "/path/to/mp3s/02_Finding_New_Friends_in_Hungary_dialog.mp3"
  }
]
```

### Scraped Dialogues (`scraped_dialogues.json`)

Hungarian-only dialogue text keyed by lesson number:

```json
{
  "2": "A: Jó reggelt!\nB: Szia! Hogy hívnak?\nA: Anne Smith.\nB: Én Pál Balázs vagyok.",
  "3": "A: Honnan jöttél? Angol vagy?\nB: Nem angol vagyok. Amerikai vagyok."
}
```

Both files allow resuming downloads/uploads after failures without re-scraping.
