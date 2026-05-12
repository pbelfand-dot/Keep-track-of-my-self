---
name: social-media-follower-tracker
description: Fetch and track public follower counts from YouTube, TikTok, Instagram, and X (Twitter) for use in websites and apps. Use this skill ANY time the user asks to track, fetch, display, monitor, or integrate follower counts, subscriber counts, or social media stats for public accounts — including dashboards, follower counters, "live count" widgets, growth trackers, leaderboards, or any app/website that needs a number-of-followers metric from these platforms. Trigger this skill even if the user only mentions one platform (e.g. "show my YouTube subs in my app") or uses casual phrasing like "pull followers," "auto-track my socials," or "build a follower widget." Covers free-tier API setup, free third-party scrapers, code patterns for web and mobile, caching strategies, and what realistically breaks in 2026.
---

# Social Media Follower Tracker

A practical, opinionated guide to fetching public follower counts from YouTube, TikTok, Instagram, and X — using only free options, and being honest about which ones are actually reliable.

## The honest landscape (read this first)

In 2026, follower-count access ranges from "easy and rock-solid" to "duct tape and prayers." Set expectations with the user before they build:

| Platform | Reliability | Method | Free? |
|---|---|---|---|
| **YouTube** | ★★★★★ Excellent | Official Data API v3 with API key | Yes (10k units/day) |
| **TikTok** | ★★★ Decent | Unofficial library or free scraper | Yes (may break) |
| **Instagram** | ★★ Fragile | Unofficial scraper or third-party service | Yes (rate-limited) |
| **X (Twitter)** | ★★ Fragile | Unofficial scraper or community proxies | Yes (rate-limited) |

**Official APIs for TikTok, Instagram, and X effectively don't help here** — they're either gated behind approval, restricted to your own authenticated account, or so expensive they're irrelevant for a hobby/indie project. So for "track any public account," the practical path is YouTube official API + unofficial methods for the rest.

Tell the user this upfront. Don't oversell — overpromising "auto-tracking" for TikTok/IG/X sets them up for breakage.

## When to use this skill

Trigger this whenever the user wants to:
- Display a follower/subscriber count on a website or app
- Build a "live count" widget or growth tracker
- Pull stats from a social profile they don't own
- Track multiple creators or competitor accounts
- Integrate any of the four platforms' follower data into a backend or frontend

If the user names one specific platform, only load that platform's reference file. If they want multiple, load each one as needed.

## Workflow

### 1. Clarify scope before writing code

Ask (briefly, only if not already answered):
- Which platforms? (YouTube, TikTok, Instagram, X — any subset)
- Public profiles only, or their own authenticated account? (This skill assumes public.)
- Frontend (browser JS), backend (Node/Python), or both?
- One-shot fetch, or continuous polling/caching?

### 2. Pick the method per platform

Read the relevant reference file(s) for full code and setup:

- **YouTube** → `references/youtube.md` (official Data API v3, API key only, very reliable)
- **TikTok** → `references/tiktok.md` (unofficial Python library, or free Apify actor)
- **Instagram** → `references/instagram.md` (public web endpoint scrape, fragile)
- **X / Twitter** → `references/x_twitter.md` (Nitter proxies or syndication endpoint, fragile)

### 3. Build with these defaults

- **Backend over frontend.** Most of these endpoints don't allow direct browser calls (CORS blocks them). Put the fetch logic on a server (Node/Express, Python/Flask, or a serverless function) and have the frontend call your own endpoint.
- **Cache aggressively.** Follower counts don't change second-to-second for most accounts. Cache for at least 5–15 minutes to avoid hitting rate limits and to keep your app fast. For "live count" UIs that look real-time, fetch on the server every 30–60s and stream/poll from the cache.
- **Handle failures gracefully.** Unofficial methods *will* break occasionally. Always wrap calls in try/catch, return a cached value as fallback, and log so the user knows when something needs a fix.
- **Normalize the output.** All four platforms should return the same shape so the frontend doesn't care which one:
  ```json
  { "platform": "youtube", "username": "MrBeast", "followers": 324000000, "fetched_at": "2026-05-12T22:00:00Z" }
  ```

### 4. Tech-stack guidance

For Paul's typical stack (single-file HTML apps, Node-friendly):

**Quick prototype (single-file HTML, dev-only)**
- Use a free CORS proxy (e.g., `corsproxy.io`) just to test.
- This works for YouTube directly because Google's API supports CORS.
- TikTok/IG/X will not work browser-direct in production — see below.

**Production (recommended)**
- Node.js + Express backend with one endpoint per platform: `/api/followers/youtube/:handle`, etc.
- Frontend fetches your own endpoint, never the social platform directly.
- Cache layer: in-memory `Map` with TTL is fine for a personal project; Redis if scaling.

**Mobile app**
- Same pattern: app calls your own backend, backend handles the social fetch.
- Don't ship API keys in the mobile binary — they can be extracted.

## Drop-in starter (Node.js backend)

This is the skeleton. Fill in per-platform fetchers from the reference files.

```javascript
// server.js
import express from 'express';
import { getYouTubeFollowers } from './platforms/youtube.js';
import { getTikTokFollowers } from './platforms/tiktok.js';
import { getInstagramFollowers } from './platforms/instagram.js';
import { getXFollowers } from './platforms/x.js';

const app = express();
const cache = new Map(); // key -> { data, expires }
const TTL_MS = 10 * 60 * 1000; // 10 min

const FETCHERS = {
  youtube: getYouTubeFollowers,
  tiktok: getTikTokFollowers,
  instagram: getInstagramFollowers,
  x: getXFollowers,
};

app.get('/api/followers/:platform/:handle', async (req, res) => {
  const { platform, handle } = req.params;
  const key = `${platform}:${handle.toLowerCase()}`;

  const cached = cache.get(key);
  if (cached && cached.expires > Date.now()) {
    return res.json({ ...cached.data, cached: true });
  }

  const fetcher = FETCHERS[platform];
  if (!fetcher) return res.status(400).json({ error: 'unknown platform' });

  try {
    const data = await fetcher(handle);
    const payload = {
      platform,
      username: handle,
      followers: data.followers,
      fetched_at: new Date().toISOString(),
    };
    cache.set(key, { data: payload, expires: Date.now() + TTL_MS });
    res.json(payload);
  } catch (err) {
    // Fall back to stale cache if we have it
    if (cached) return res.json({ ...cached.data, cached: true, stale: true });
    res.status(502).json({ error: 'fetch failed', detail: err.message });
  }
});

app.listen(3000, () => console.log('Follower API on :3000'));
```

Each `getXXXFollowers(handle)` function returns `{ followers: <number> }`. The reference files contain the full implementation per platform.

## Frontend pattern

Once the backend is running, the frontend is trivial:

```javascript
async function loadFollowers(platform, handle) {
  const res = await fetch(`/api/followers/${platform}/${handle}`);
  const data = await res.json();
  return data.followers;
}

// Poll for "live" feel
setInterval(async () => {
  const count = await loadFollowers('youtube', 'MrBeast');
  document.getElementById('count').textContent = count.toLocaleString();
}, 30000);
```

## What to do when something breaks

Unofficial scrapers break when platforms update their site or anti-bot systems. When that happens:

1. **YouTube** — Almost never breaks. If it does, check API key quota first (10k units/day; subscriber lookup is 1 unit).
2. **TikTok / Instagram / X** — If a scraper stops working, fall back to a free third-party service (see each reference file for current options) or switch to a different unofficial library. Build the system so swapping out one platform's fetcher is a one-file change.
3. **Always log failures** so the user can see "Instagram fetcher broken for 2 days" instead of silent stale data.

## Things this skill deliberately doesn't do

- Authenticated/private account access — that's a totally different problem (OAuth flows per platform).
- Follower *lists* (who specifically follows an account) — much harder, much more rate-limited, ethically dicey.
- Engagement metrics, post analytics, demographics — out of scope; can be added later per-platform.
- Paid third-party aggregators (Social Blade API, RapidAPI premium endpoints) — possible but not free.

If the user asks for any of these, note the scope and ask whether they want to expand the skill.
