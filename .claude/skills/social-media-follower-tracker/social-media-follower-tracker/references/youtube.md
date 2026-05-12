# YouTube Follower (Subscriber) Tracking

The cleanest of the four. Uses the official YouTube Data API v3 with a free API key. No OAuth needed for public subscriber counts.

## Setup (5 minutes)

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (or use an existing one).
3. Search for "YouTube Data API v3" in the API library and click **Enable**.
4. Go to **Credentials** → **Create Credentials** → **API key**.
5. Copy the key. (Restrict it to "YouTube Data API v3" only, for safety.)

That's it. The key is free. Default quota: **10,000 units/day**. A subscriber-count lookup costs **1 unit**, so you can do up to 10,000 lookups per day per key.

## Important caveat: rounded counts

YouTube's public API returns subscriber counts rounded to **three significant figures** for channels with more than 1,000 subs. So MrBeast doesn't show as `324,512,847` — it shows as `324,000,000`. This applies to every third-party tool, too. Only the channel owner (using OAuth) can see exact counts.

Tell the user this if they're building a "exact live count" feature.

## Fetching by channel handle (e.g., `@MrBeast`)

The API doesn't accept handles directly — you need either a **channel ID** (starts with `UC...`) or to look up the channel via the `search` endpoint (expensive, 100 units) or `channels` endpoint with `forHandle` (1 unit, preferred).

```javascript
// platforms/youtube.js
const API_KEY = process.env.YOUTUBE_API_KEY;

export async function getYouTubeFollowers(handle) {
  // Strip leading @ if present
  const cleanHandle = handle.replace(/^@/, '');

  const url = new URL('https://www.googleapis.com/youtube/v3/channels');
  url.searchParams.set('part', 'statistics');
  url.searchParams.set('forHandle', `@${cleanHandle}`);
  url.searchParams.set('key', API_KEY);

  const res = await fetch(url);
  if (!res.ok) throw new Error(`YouTube API ${res.status}`);
  const data = await res.json();

  if (!data.items || data.items.length === 0) {
    throw new Error(`Channel not found: ${handle}`);
  }

  const stats = data.items[0].statistics;
  if (stats.hiddenSubscriberCount) {
    throw new Error('Subscriber count is hidden by the channel owner');
  }

  return {
    followers: parseInt(stats.subscriberCount, 10),
    views: parseInt(stats.viewCount, 10),
    videos: parseInt(stats.videoCount, 10),
  };
}
```

## Fetching by channel ID (faster, more reliable)

If you already have the channel ID (e.g., `UCX6OQ3DkcsbYNE6H8uQQuVA`), use `id` instead of `forHandle`:

```javascript
url.searchParams.set('id', 'UCX6OQ3DkcsbYNE6H8uQQuVA');
```

You can also pass multiple IDs comma-separated to look up many channels in one call (still 1 quota unit total):

```javascript
url.searchParams.set('id', 'UCX6OQ3D...,UCq-Fj5jknLsUf-MWSy4_brA,...');
```

This batches up to 50 channels per request — useful for leaderboards.

## Browser-direct calls

YouTube's API supports CORS, so you *can* call it directly from a browser. **But don't ship the API key in client-side code** — anyone can grab it from devtools and burn your quota. Always proxy through your own backend in production.

For a dev-only prototype, browser-direct is fine.

## Common errors

- `403 quotaExceeded` — You've hit 10k units today. Wait until midnight Pacific Time, or request a quota increase in the Cloud Console (free, typically approved in 1–2 business days).
- `400 keyInvalid` — Key is wrong or restricted to a different API/referrer.
- `404` / empty `items` — Channel doesn't exist, or you used a username/handle the API can't resolve. Try `forHandle` with the `@` prefix, or look up the channel ID manually.
