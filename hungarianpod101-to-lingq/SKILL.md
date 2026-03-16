---
name: hungarianpod101-to-lingq
description: Use when scraping audio from HungarianPod101 lesson pathways, when downloading dialogue MP3s from Innovative Language sites, when uploading audio lessons to LingQ via API, or when dealing with CDN bot protection on hungarianpod101.com
---

# HungarianPod101 to LingQ Pipeline

## Overview

Download dialogue MP3s from HungarianPod101 lesson pathways and upload them to LingQ as a private course. LingQ auto-generates transcripts from audio, so only audio files are needed.

## Key Challenges

1. **Bot protection**: HungarianPod101 triggers reCAPTCHA after ~8-16 requests
2. **CDN requires Referer header**: Without `Referer: https://www.hungarianpod101.com/`, CDN returns 403
3. **LingQ API v2 is obsolete**: Must use v3

## Architecture

```
HungarianPod101 lesson pages
  → Playwright + stealth (extract dialogue URLs)
  → CDN direct download (with Referer header)
  → LingQ API v3 upload (multipart form)
```

## Step 1: Extract Chrome Cookies

```python
from pycookiecheat import chrome_cookies
cookies = chrome_cookies("https://www.hungarianpod101.com")
# Key cookies: PHPSESSID, guid, guidmember, _amember_ru, _amember_rp
```

User must be logged into HungarianPod101 in Chrome first.

## Step 2: Scrape Lesson Pages for Dialogue URLs

### Playwright with Stealth (Required)

Plain `requests` and even plain Playwright get blocked. Use `playwright-stealth`:

```python
from playwright.async_api import async_playwright
from playwright_stealth import Stealth

async with async_playwright() as p:
    browser = await p.chromium.launch(headless=True)
    context = await browser.new_context()

    # Apply stealth BEFORE navigating
    stealth = Stealth()
    await stealth.apply_stealth_async(context)

    # Inject cookies
    await context.add_cookies([
        {"name": k, "value": v, "domain": ".hungarianpod101.com", "path": "/"}
        for k, v in cookies.items()
    ])

    page = await context.new_page()
    await page.goto(lesson_url, wait_until="networkidle")
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

```python
url = "https://www.lingq.com/api/v3/{lang}/lessons/"
headers = {"Authorization": "Token {api_key}"}
data = {
    "title": lesson_title,
    "text": "Dialogue placeholder.",  # LingQ generates transcript from audio
    "collection": "{course_id}",      # String, not int
    "status": "private",
    "level": "1",
}
with open(audio_path, "rb") as f:
    files = {"audio": (os.path.basename(audio_path), f, "audio/mpeg")}
    r = requests.post(url, headers=headers, data=data, files=files, timeout=120)
# Expect 201 on success
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
| Dependencies | `pycookiecheat`, `playwright`, `playwright-stealth`, `requests` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using LingQ API v2 | Returns `["API is obsolete. Use v3 instead."]`. Use v3. |
| Forgetting Referer header on CDN | 403 error. Always send `Referer: https://www.hungarianpod101.com/` |
| Using `requests` directly for lesson pages | Bot-blocked. Must use Playwright + stealth with injected cookies. |
| Not rotating browser contexts | CAPTCHA after ~8-16 pages. Create new context per batch. |
| Sending `collection` as int to LingQ | Must be string in multipart form data. |
| Downloading full lesson audio instead of dialogue | Look for `_dialog.mp3` suffix specifically. |

## Metadata File Format

Save scraped metadata as JSON for resumability:

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

This allows resuming downloads and uploads after failures without re-scraping.
