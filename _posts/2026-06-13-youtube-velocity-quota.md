---
layout: post
title: "Tracking YouTube Video Velocity Without Burning Your API Quota"
author: Kürşat Kutlu Aydemir
---

*How I monitored views-per-hour across thousands of videos on a 10,000-unit-a-day budget, by never using the expensive endpoint.*

When I built a YouTube analytics tool for the gaming niche, the single most important metric was **velocity**: how fast a video is gaining views *right now*, measured in views per hour. Velocity is what separates "trending" from "already trended". A video doing 50,000 views/hour today is a different story than one that did 2 million views total over six months.

But velocity has an awkward property; to measure a *rate* you have to sample the same video repeatedly over time. And on YouTube, every sample costs you. The YouTube Data API gives you **10K quota units per day** by default. And if you're naive about it, you'll blow through that before lunch.

Here's how I tracked velocity across thousands of videos and hundreds of channels on that budget, and the one mental shift that made it possible.

## The wall: 10K units, and not all calls are equal

The YouTube Data API doesn't rate-limit you by request count. It rate-limits you by **quota units**, and different endpoints cost wildly different amounts. The numbers I hard-coded as constants:

```go
const (
    QuotaCostSearch        = 100  // search.list  - the expensive one
    QuotaCostVideosList    = 1    // videos.list
    QuotaCostChannelsList  = 1    // channels.list
)
```

That `search.list` cost of **100** is the trap. It's the most intuitive endpoint, "search this channel for new videos", and it's a hundred times more expensive than everything else. Your entire daily budget is **100 searches**.

Let me make the disaster concrete. Say you track 300 channels and want to catch new uploads by searching each channel hourly:

```
300 channels × 100 units × 24 hours = 720,000 units/day
```

That's **72× your entire daily quota** for *discovery alone*, before you've measured a single view count. Even searching each channel just once a day is 30,000 units: 3× over budget, dead on arrival.

So the rule writes itself; you basically can't use `search.list`. Which sounds impossible for a product whose whole job is finding and measuring videos. The way out is to stop thinking of it as one problem.

## The mental shift: discovery is not measurement

Velocity tracking is actually two separate jobs with totally different cost profiles:

1. **Discovery**: "a new video exists" Happens once per video. Should be *free*.
2. **Measurement**: "this video now has N views" Happens many times per video. Should be *cheap*.

Conflating them is what kills your quota. `search.list` does both at once and expensively. Once you split them, each half has a much better tool.

```
   ┌────────────────────────────────────────────────────────────────┐
   │             VELOCITY  =  Delta Views / Delta Hours             │
   └────────────────────────────────────────────────────────────────┘
              ▲                                          ▲
              │ needs: "a video exists"                  │ needs: "its view count, now"
              │                                          │
   ┌──────────┴───────────┐                  ┌───────────┴────────────────┐
   │   DISCOVERY          │                  │   MEASUREMENT              │
   │   once per video     │                  │   many times per video     │
   ├──────────────────────┤                  ├────────────────────────────┤
   │ PubSubHubbub push    │                  │ videos.list, batched ×50   │
   │ + RSS Atom feed      │                  │ on a tiered schedule       │
   │                      │                  │ (hot/recent/older/archive) │
   │ ►  0 quota units     │                  │ ►  1 unit per 50 videos    │
   └──────────────────────┘                  └────────────────────────────┘

   ✗ The naive way: search.list does BOTH at once, 100 units a call.
     300 channels × 100 × 24h = 720,000 units/day = 72× your budget.
```

## Part 1: Free discovery with RSS + PubSubHubbub

YouTube publishes a public Atom/RSS feed for every channel, and it's wired into **PubSubHubbub**, a publish-subscribe protocol. Instead of polling YouTube and asking "anything new?", you subscribe once and YouTube *pushes* you a webhook the moment a video goes live.

None of this touches the Data API. **Discovery cost: zero quota units.**

Subscribing is a single form POST to Google's hub:

```go
const (
    PubSubHubURL        = "https://pubsubhubbub.appspot.com/subscribe"
    YouTubeFeedURLBase  = "https://www.youtube.com/xml/feeds/videos.xml"
    DefaultLeaseSeconds = 432000 // 5 days
)

topicURL := fmt.Sprintf("%s?channel_id=%s", YouTubeFeedURLBase, channel.YoutubeID)

data := url.Values{}
data.Set("hub.callback", s.cfg.PubSub.CallbackURL)
data.Set("hub.topic", topicURL)
data.Set("hub.verify", "async")
data.Set("hub.mode", "subscribe")
data.Set("hub.lease_seconds", fmt.Sprintf("%d", DefaultLeaseSeconds))
data.Set("hub.secret", secret) // used to verify pushes are really from the hub

resp, err := s.httpClient.PostForm(PubSubHubURL, data)
```

The hub then calls your callback URL with a `hub.challenge` to confirm you actually wanted the subscription (the `hub.verify=async` handshake), and from then on it POSTs you an Atom feed every time the channel uploads. You parse out the video ID and channel ID:

```go
type YouTubeAtomFeed struct {
    XMLName xml.Name `xml:"feed"`
    Entries []struct {
        VideoID   string `xml:"videoId"`
        ChannelID string `xml:"channelId"`
        Title     string `xml:"title"`
        Published string `xml:"published"`
    } `xml:"entry"`
}
```

Two things worth getting right:

**Verify the signature**. Anyone who finds your callback URL can POST fake videos at it. The hub signs each push with the secret you provided (HMAC-SHA1), so check it before trusting anything:

```go
func verifySignature(body []byte, signature, secret string) bool {
    if !strings.HasPrefix(signature, "sha1=") {
        return false
    }
    mac := hmac.New(sha1.New, []byte(secret))
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(signature[5:]), []byte(expected))
}
```

**Subscriptions expire**. That `lease_seconds` is 5 days, leases are not forever. You need a background job that renews anything expiring soon, or your "real-time" feed quietly goes dark. Mine sweeps for subscriptions expiring in the next 24 hours and re-subscribes:

```go
expiringBefore := time.Now().Add(24 * time.Hour)
subs, _ := s.repo.GetExpiringPubSubSubscriptions(ctx, expiringBefore)
for _, sub := range subs {
    _ = s.Subscribe(ctx, sub.Channel) // re-subscribe, still free
}
```

When the webhook fires, I don't even fetch stats synchronously, I just hand the video ID off to a goroutine and return `200 OK` immediately, so a burst of uploads can't block the handler.

## Part 2: Cheap measurement with batched `videos.list`

Now the other half: actually reading view counts. This is where `videos.list` (costs **1**) shines, because it accepts **up to 50 video IDs in a single request**, still for one unit.

```go
// GetVideoStatsBatch fetches stats for up to 50 videos in a single API call.
// Cost: 1 quota unit for up to 50 videos.
func (c *Client) GetVideoStatsBatch(ctx context.Context, videoIDs []string) (map[string]VideoStats, error) {
    if len(videoIDs) > 50 {
        videoIDs = videoIDs[:50] // API hard limit
    }
    params := url.Values{
        "part": {"statistics,snippet,contentDetails,liveStreamingDetails"},
        "id":   {strings.Join(videoIDs, ",")},
    }
    return c.makeRequest(ctx, "videos", params, QuotaCostVideosList)
}
```

The arithmetic flips entirely. Measuring 50 videos one-by-one is 50 units; batched, it's **1**. A 98% reduction, and the bigger your catalog the more it matters.

With fresh view counts in hand, velocity itself is almost embarrassingly simple — it's just a delta between two timestamped snapshots. Each refresh writes a `VideoMetric` row, and the rate is computed against the previous one:

```go
if prevMetric != nil {
    timeDiff := time.Since(prevMetric.RecordedAt).Hours()
    if timeDiff > 0 { // guard against divide-by-zero on same-instant samples
        metric.ViewDelta    = newViewCount - prevMetric.ViewCount
        metric.ViewVelocity = float64(metric.ViewDelta) / timeDiff // views per hour
        video.ViewVelocity  = metric.ViewVelocity
    }
}
```

That's the whole "algorithm." Velocity isn't hard math — it's `Δviews / Δhours`. The hard part is affording the samples that feed it.

## Part 3: Spend your quota where it actually matters

Even at 1 unit per 50 videos, refreshing *every* tracked video *every* hour is wasteful. A video published 8 months ago is not going to suddenly accelerate; a video published 40 minutes ago might be exploding. So I refresh on a sliding scale based on age:

```go
groups := []TieredVideoGroup{
    {Name: "hot",     MaxAge: 48 * time.Hour,       RefreshAfter: 1 * time.Hour},
    {Name: "recent",  MaxAge: 7 * 24 * time.Hour,   RefreshAfter: 4 * time.Hour},
    {Name: "older",   MaxAge: 30 * 24 * time.Hour,  RefreshAfter: 12 * time.Hour},
    {Name: "archive", MaxAge: 365 * 24 * time.Hour, RefreshAfter: 24 * time.Hour},
}
```

Fresh videos (< 48h old) get sampled hourly — fine enough resolution to catch a spike. Week-old videos every 4 hours. Anything pushing a year, once a day. The query just asks "which videos in this age band haven't been fetched since their refresh window?" and the worker processes them in batches of 50.

This is the core idea behind the whole design: **resolution should follow volatility.** Sample the things that are changing fast, fast; sample everything else lazily.

## Part 4: A hard ceiling so you never get cut off

Finally, a guardrail. Quota tracking is in-memory with a DB log, and every call checks a buffer before firing — so background jobs can't accidentally drain the budget and leave the user-facing parts of the app unable to make a call:

```go
func (c *Client) CanMakeRequest(cost int) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.checkAndResetQuota() // resets at midnight Pacific, matching YouTube
    return (c.quotaUsed + cost + c.quotaBuffer) <= c.dailyQuota
}
```

With the defaults (`dailyQuota: 10000`, `quotaBuffer: 500`), the workers voluntarily stop at 9,500 units, reserving the last 500 for live user actions like a manual refresh or a channel lookup. The counter resets at midnight Pacific — matching when YouTube actually resets, not your server's local midnight, which is a subtle bug if you skip it.

## Putting it together: the budget

Here's a back-of-envelope day for an illustrative catalog of ~300 channels / ~5,000 videos across the tiers:

| Job | How | Quota/day |
|---|---|---|
| Discover new uploads | PubSubHubbub push | **0** |
| Hot tier (~300 vids, hourly) | 6 batches × 24 | ~144 |
| Recent tier (~700 vids, /4h) | 14 batches × 6 | ~84 |
| Older tier (~1,500 vids, /12h) | 30 batches × 2 | ~60 |
| Archive (~2,500 vids, /day) | 50 batches × 1 | ~50 |
| **Total** | | **~340 / 10,000** |

Roughly **3% of the daily budget** — versus the 720,000 units the search-based version wanted. The headroom went toward channel-level stats and the occasional genuine search for onboarding new channels.

## What I'd reuse on the next thing

- **Split discovery from measurement.** It's the whole game. One is a once-per-item event (make it free, via push); the other is a recurring sample (make it cheap, via batching).
- **Push beats poll** whenever the platform offers it. PubSubHubbub turned "poll 300 channels constantly" into "wait for a webhook," for zero quota.
- **Batch endpoints are a force multiplier.** Always check whether the cheap endpoint takes a list. 50:1 is a huge lever.
- **Match sampling rate to volatility.** Tiered refresh is a couple dozen lines and cut steady-state quota by an order of magnitude.
- **Leave yourself a buffer.** A background worker that can starve your foreground requests is a bad day waiting to happen.

The funny epilogue is that solving the quota problem was, by a wide margin, the *easy* part of building this product — and I'll get to the hard parts (the ones that don't have a clean engineering answer) later in this series. But if you're building anything on top of the YouTube API, this is the architecture I'd start from.
