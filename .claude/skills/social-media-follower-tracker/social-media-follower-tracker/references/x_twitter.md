# X (Twitter) Follower Tracking

The free tier of X's official API is essentially write-only since 2023 — paid tiers start at $100/month for Basic and $5,000/month for Pro. For a free tracker, we have to go around it.

The X-specific challenge is that Twitter has actively shut down most unofficial endpoints since the Musk takeover. What worked in 2023 doesn't work now. The current best free options are below; expect to swap between them over time.

## Option A: Self-hosted Nitter instance

[Nitter](https://github.com/zedeus/nitter) is a privacy-friendly Twitter frontend that scrapes public profile data. Public instances exist but are unreliable (frequently rate-limited or shut down by X). The robust setup is to self-host your own Nitter instance, which gives you a stable JSON endpoint.

If self-hosting is too much, see Option B for a quicker (but more fragile) path.

```javascript
// platforms/x.js — using a Nitter instance you control
const NITTER_HOST = process.env.NITTER_HOST || 'https://nitter.yourdomain.com';

export async function getXFollowers(handle) {
  const username = handle.replace(/^@/, '');
  const res = await fetch(`${NITTER_HOST}/${username}`);
  if (!res.ok) throw new Error(`Nitter ${res.status}`);
  const html = await res.text();

  // Nitter renders follower count in a known span
  const match = html.match(/<span class="profile-stat-num">([\d,]+)<\/span>\s*<\/li>\s*<li class="followers"/);
  if (!match) throw new Error('Could not parse follower count from Nitter HTML');

  return { followers: parseInt(match[1].replace(/,/g, ''), 10) };
}
```

Nitter setup is a few hours' work and requires a server, but once running it's the most stable free option.

## Option B: Twitter's syndication endpoint

X has a `cdn.syndication.twimg.com` endpoint used to embed tweets/profiles on third-party sites. It returns JSON for some public profile data without auth. It's been around for years and is more durable than the main API because Twitter uses it for their own embed widgets.

```javascript
export async function getXFollowersSyndication(handle) {
  const username = handle.replace(/^@/, '');
  const url = `https://cdn.syndication.twimg.com/timeline/profile?screen_name=${username}`;

  const res = await fetch(url, {
    headers: {
      'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    },
  });

  if (!res.ok) throw new Error(`Syndication ${res.status}`);
  const data = await res.json();

  // Profile data is in the headers section
  const user = data?.headers?.user || data?.user;
  if (!user) throw new Error('No user data in syndication response');

  return {
    followers: user.followers_count,
    following: user.friends_count,
    tweets: user.statuses_count,
    verified: user.verified,
  };
}
```

This endpoint's shape changes occasionally, so log the raw response on errors and adjust.

## Option C: Reverse-engineered libraries

Open-source Node and Python libraries exist that wrap X's internal GraphQL endpoints (the same ones the website itself uses):

- Python: `tweety-ns` (no auth, scraping-based) or `twikit` (requires X account credentials)
- Node: `the-convocation/twitter-scraper`

These work but break frequently when X updates their internal APIs (which happens often). Useful as a fallback layer in your fetcher hierarchy.

## Option D: Free third-party services

Several services offer free tiers for X data scraping (SociaVault, GetXAPI, Apify Twitter scrapers). Free credits typically cover a few hundred to a few thousand lookups before requiring payment. Useful as a fallback.

## Recommended strategy: fetcher chain

Because each option breaks independently, the most robust approach is a chain that tries methods in order:

```javascript
const FETCHERS = [
  getXFollowersSyndication,  // most reliable when it works
  getXFollowersNitter,        // your self-hosted instance
  getXFollowersThirdParty,    // free third-party fallback
];

export async function getXFollowers(handle) {
  let lastError;
  for (const fetcher of FETCHERS) {
    try {
      return await fetcher(handle);
    } catch (err) {
      lastError = err;
      console.warn(`X fetcher ${fetcher.name} failed:`, err.message);
    }
  }
  throw lastError;
}
```

## Browser-direct: definitely not

CORS blocked, anti-bot detection is heavy, and X aggressively rate-limits unauthenticated requests by IP. Must be backend.

## Rate limit warning

Even via Nitter or syndication, X rate-limits by IP. From a cloud server, expect to manage a few dozen unique lookups per hour at most before getting flagged. Cache for at least 15 minutes per username, longer for non-critical use cases.

## When it breaks (which is often, for X)

X breaks unofficial methods more aggressively than the other platforms. Expect to revisit this fetcher every 3–6 months.

When it fails:
1. Check if Nitter/syndication is returning HTML instead of JSON (usually means anti-bot wall).
2. Check the unofficial-libraries' GitHub issues — someone usually posts a fix within days.
3. Worst case, accept that X tracking might be down for a few days and rely on cached values + a warning UI to your users.
