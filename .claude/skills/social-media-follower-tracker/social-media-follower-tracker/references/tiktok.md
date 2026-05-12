# TikTok Follower Tracking

Harder than YouTube. The official TikTok API does **not** expose follower counts for arbitrary public profiles — it's gated behind an approval process and only useful for authenticated users or research access. For the "give me any username, get me the count" use case, you have to go unofficial.

## Two practical free paths

### Option A: Unofficial Python library (`TikTokApi`)

Open-source, actively maintained, scrapes TikTok's public web data. Best for backend Python projects.

```bash
pip install TikTokApi
python -m playwright install
```

```python
# platforms/tiktok.py
from TikTokApi import TikTokApi
import asyncio

async def get_tiktok_followers(username):
    username = username.lstrip('@')
    async with TikTokApi() as api:
        await api.create_sessions(ms_tokens=[None], num_sessions=1, sleep_after=3)
        user = api.user(username=username)
        user_data = await user.info()

        stats = user_data['userInfo']['stats']
        return {
            'followers': stats['followerCount'],
            'following': stats['followingCount'],
            'likes': stats['heartCount'],
            'videos': stats['videoCount'],
        }

# usage
# asyncio.run(get_tiktok_followers('mrbeast'))
```

**Tradeoffs:**
- Needs a headless browser (Playwright) — adds ~200MB to your deploy.
- May need an `ms_token` cookie value from a real browser session if TikTok tightens up.
- Can break when TikTok updates its frontend. Check the library's GitHub issues if calls start failing.

### Option B: Node.js — fetch the public web profile JSON

TikTok embeds a JSON blob in the page HTML at `/@username`. You can scrape it without a headless browser, but TikTok's bot detection is aggressive and this method breaks more often than the Python lib.

```javascript
// platforms/tiktok.js
export async function getTikTokFollowers(handle) {
  const username = handle.replace(/^@/, '');
  const url = `https://www.tiktok.com/@${username}`;

  const res = await fetch(url, {
    headers: {
      // Pretend to be a real browser; without these you'll get a wall
      'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
      'Accept-Language': 'en-US,en;q=0.9',
    },
  });

  if (!res.ok) throw new Error(`TikTok ${res.status}`);
  const html = await res.text();

  // Extract the embedded JSON
  const match = html.match(/<script id="__UNIVERSAL_DATA_FOR_REHYDRATION__"[^>]*>(.*?)<\/script>/s);
  if (!match) throw new Error('TikTok page format changed — selector broke');

  const data = JSON.parse(match[1]);
  const userInfo = data?.__DEFAULT_SCOPE__?.['webapp.user-detail']?.userInfo;
  if (!userInfo) throw new Error('User data not found in page JSON');

  return {
    followers: userInfo.stats.followerCount,
    following: userInfo.stats.followingCount,
    likes: userInfo.stats.heartCount,
    videos: userInfo.stats.videoCount,
  };
}
```

**Tradeoffs:**
- No Playwright dependency, fast.
- Breaks more often — TikTok rotates the script tag ID and JSON structure periodically. When it breaks, look at a fresh page source and update the selector.
- Will start returning 403/captcha pages if you hit it too hard from one IP. Keep request rate low and cache aggressively (15+ min).

### Option C: Free third-party scraper (Apify free tier)

If you don't want to maintain unofficial code, Apify offers a TikTok Profile Scraper with a free tier. You hit their API, they handle the scraping infrastructure.

- Sign up at apify.com (free, no credit card).
- Find "TikTok Profile Scraper" actor.
- Call their API with your token and a username.

Lower maintenance, but you're at their mercy for free-tier limits. Worth keeping in your back pocket as a fallback when your own scraper breaks.

## Browser-direct: don't bother

TikTok's pages do not allow CORS for the embedded JSON, and their public endpoints are protected. This must run server-side.

## When TikTok changes the page format

It will. The fix is usually:
1. Open `https://www.tiktok.com/@<some_user>` in a browser, view source.
2. Search for `followerCount` to find where it lives now.
3. Update the regex/JSON path in your fetcher.

Worth wrapping the scraper in a try/catch that emails or logs a clear "TikTok parser needs updating" message.
