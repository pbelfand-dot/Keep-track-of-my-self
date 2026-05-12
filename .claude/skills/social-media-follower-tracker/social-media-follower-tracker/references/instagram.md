# Instagram Follower Tracking

The most hostile of the four. Meta killed the Basic Display API in Dec 2024. The Graph API only works for Business/Creator accounts you own (or are authorized by). For "track any public profile," it's unofficial methods only.

Plan for breakage. Cache aggressively.

## Option A: Public profile JSON endpoint

Instagram exposes a JSON version of profile pages at a specific URL with the right headers. It's not officially supported and Meta tightens it periodically, but as of May 2026 it still works for public profiles.

```javascript
// platforms/instagram.js
export async function getInstagramFollowers(handle) {
  const username = handle.replace(/^@/, '');
  const url = `https://www.instagram.com/api/v1/users/web_profile_info/?username=${username}`;

  const res = await fetch(url, {
    headers: {
      // The X-IG-App-ID header is what unlocks the JSON response
      'X-IG-App-ID': '936619743392459',
      'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
      'Accept': '*/*',
      'Accept-Language': 'en-US,en;q=0.9',
    },
  });

  if (res.status === 404) throw new Error(`User not found: ${username}`);
  if (res.status === 401 || res.status === 403) {
    throw new Error('Instagram blocked the request — likely rate-limited or IP-flagged');
  }
  if (!res.ok) throw new Error(`Instagram ${res.status}`);

  const data = await res.json();
  const user = data?.data?.user;
  if (!user) throw new Error('Profile data not in response');
  if (user.is_private) {
    // Private accounts: follower count is still visible
    // but other fields may be missing
  }

  return {
    followers: user.edge_followed_by.count,
    following: user.edge_follow.count,
    posts: user.edge_owner_to_timeline_media.count,
    verified: user.is_verified,
    is_private: user.is_private,
  };
}
```

**About `X-IG-App-ID: 936619743392459`** — that's Instagram's public web app ID. It's the same one their own website sends. Without it, the endpoint returns HTML instead of JSON.

## Option B: Scrape the profile page

Falls back to parsing HTML if Option A gets blocked. More fragile, but a useful backup:

```javascript
async function getInstagramFollowersHTML(handle) {
  const username = handle.replace(/^@/, '');
  const res = await fetch(`https://www.instagram.com/${username}/`, {
    headers: { 'User-Agent': 'Mozilla/5.0 ... Chrome/120.0.0.0' },
  });
  const html = await res.text();

  // Instagram embeds Open Graph meta tags with follower count in plain text
  const match = html.match(/content="([\d,]+) Followers/);
  if (!match) throw new Error('Could not find follower count in HTML');

  return { followers: parseInt(match[1].replace(/,/g, ''), 10) };
}
```

The OG meta-tag method is ironically more durable than the JSON endpoint, because Meta uses it for link previews and it'd hurt their SEO to remove it. Counts come back as approximate (e.g., "1.2M") sometimes — handle both formats.

## Option C: Free third-party (Apify)

Apify's "Instagram Followers Count Scraper" has a free tier. Same tradeoff as the TikTok version — easier to maintain, but you depend on their service.

## Rate limiting reality

Instagram WILL rate-limit you faster than the others. From a single residential IP, you can probably do **dozens of lookups per hour** before getting a temporary block. From a cloud datacenter IP (AWS, GCP, etc.), it's much less — often a few dozen total before the IP gets flagged.

**Mitigations:**
- Cache for at least 15–30 minutes per username.
- Don't poll faster than once per minute even for "live" counters.
- If running on a VPS, expect more frequent blocks. Consider running this fetcher from a residential connection or using a paid residential-proxy service if it becomes critical.

## Browser-direct: don't

Instagram explicitly blocks the JSON endpoint via CORS for non-instagram.com origins. Backend-only.

## When it breaks

Symptoms: 401/403 responses, empty JSON, or HTML returned instead of JSON.

Steps:
1. Open the same URL in a regular browser and verify the data is still there.
2. Check the request/response headers in devtools — Meta may have added a new required header (it's happened before with `X-ASBD-ID`, `X-IG-WWW-Claim`, etc.).
3. Fall back to Option B (HTML scrape).
4. Worst case, switch to a third-party service temporarily while you fix.
