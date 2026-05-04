# the-world-now-architecture

# The World Now — Architecture Notes

> How we built an AI-powered news platform that ships 6,000+ articles, ranks #1.2 on Bing for "what wars are going on today," and runs on one developer's laptop. This is the engineering writeup — what we use, why, and the parts that surprised us.

**Live site:** https://the-world-now.com

---

## What it is

[The World Now](https://the-world-now.com) is a real-time, AI-powered news platform that tracks global conflicts, disasters, and market-moving events. It launched in December 2025. As of May 2026:

- **6,061 articles published**
- **DR 15** on Ahrefs (up from 0.2 in three months)
- **#1.2 on Bing** for "what wars are going on today"
- **~57% of traffic** comes from Bing-powered search (Bing, Edge, Yahoo, DuckDuckGo, ChatGPT browse, Copilot)
- Two evergreen trackers — [/current-wars](https://the-world-now.com/current-wars) and [/catalyst](https://the-world-now.com/catalyst) — built on top of a global event database

Built and operated by one person. This document walks through the parts that matter.

---

## The high-level shape

```
                ┌──────────────────────────────────────────┐
                │  Source ingestion (cron + on-demand)     │
                │  • 60+ news APIs / RSS feeds             │
                │  • USGS earthquakes, GDELT, ACLED        │
                │  • CoinGecko / market data               │
                └──────────────┬───────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────────┐
                │  AWS Lambda fleet                        │
                │  • news-scraper                          │
                │  • news-classifier (LLM)                 │
                │  • article-generator (LLM)               │
                │  • event-relations (cause→effect graph)  │
                │  • aircraft-tracker / ship-tracker       │
                └──────────────┬───────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────────┐
                │  MongoDB Atlas                           │
                │  • articles, events, sources             │
                │  • subscribers, A/B test variants        │
                └──────────────┬───────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────────┐
                │  Next.js (Vercel) — App Router, SSG/ISR  │
                │  • SEO-first rendering                   │
                │  • Catalyst predictions                  │
                │  • Premium tier (Stripe)                 │
                └──────────────────────────────────────────┘
```

That's the whole stack. No managed CMS, no headless platform, no media pipeline. Every box is one of three things: a Lambda, a Mongo collection, or a Next.js route.

---

## Where the articles come from

The article-generator Lambda runs on a schedule and does this loop:

1. **Pull** unprocessed events from the events collection.
2. **Cluster** related events (cause → effect chains — e.g. "Houthi missile attack" → "Red Sea shipping rerouted" → "oil futures spike").
3. **Score** each cluster for newsworthiness — recency, geographic spread, severity, and topical match against our keyword pipeline.
4. **Generate** a long-form article per cluster using a prompt-cached system prompt + the clustered event payload.
5. **SEO-pass** — generate title variants, meta description, schema.org `NewsArticle` markup, internal links, and a slug.
6. **Persist** to MongoDB with full provenance (source URLs, event IDs, model version, prompt version).
7. **Submit** to IndexNow so Bing knows the article exists within seconds.

The output is currently 6,061 articles. Roughly 30 of those are indexed on Google. About 877 of them are getting clicks on Bing in any given week. Both numbers grow weekly. The Google number grows slowly. The Bing number grows fast.

A few engineering decisions that were not obvious going in:

**Schema-first generation.** The model output is a strict JSON schema: `headline`, `dek`, `body_md`, `tags`, `entities`, `geo`, `confidence`, `sources[]`. That means we never have to "parse" the article — we just persist the JSON. The body field is markdown so it round-trips cleanly through the database and the renderer.

**Prompt versioning in the row.** Every article row stores `prompt_version` and `model_version` next to the content. When we change the system prompt, we don't lose the ability to ask "did the new prompt actually produce better articles" — we A/B over the prompt, not just the UI.

**Source provenance is not optional.** Every article links back to its source events, which link back to the raw API responses they came from. This matters legally (we're aggregating, not scraping) and it matters for SEO (every article has 3–10 outbound citations to high-authority sources, which Bing rewards).

---

## The Bing thing

The thing nobody warned us about: in 2026, **Bing ships AI search**. ChatGPT's web browse runs on Bing. Copilot runs on Bing. Yahoo runs on Bing. DuckDuckGo runs on Bing. If you rank on Bing, you rank in the AI tools that millions of people now use as their default search.

We learned this empirically. Last week we got 1,000 sessions from Bing-powered surfaces and 11 from Google. The site is identical for both — same content, same schema, same internal linking. Bing rewards new, structured, frequently-updated content with high recency authority. Google does not (or at least not without years of E-E-A-T runway).

So the engineering implication is: every Lambda we built was implicitly Bing-optimized, even though we were aiming for Google. The shape of what works on Bing turned out to be:

- **Submit via [IndexNow](https://www.bing.com/indexnow)** the moment an article is written. Sub-second indexing.
- **Schema.org `NewsArticle` and `LiveBlogPosting`** for trackers. Bing parses these heavily.
- **Internal linking density.** We auto-generate "related events" and "cause/effect" sidebars. Each article averages 8 internal links.
- **Update timestamps that are real.** When the underlying events change, the article gets a new `dateModified`. Bing trusts this; Google ignores it.

If you're building anything content-heavy in 2026, build for Bing first and let Google catch up.

---

## Programmatic pages: trackers and locations

The site has two kinds of pages:

**Articles** — what the article-generator produces. ~6,000 of them, one per event cluster.

**Trackers** — programmatic pages built on top of the events collection. The big ones:

- [/current-wars](https://the-world-now.com/current-wars) — global conflict map, severity-ranked, with embedded child pages per conflict
- [/earthquakes-today](https://the-world-now.com/earthquakes-today) — USGS-backed earthquake feed with 15 location-specific child routes (e.g. `/earthquakes-today/california`)
- [/oil-price-forecast](https://the-world-now.com/oil-price-forecast) — daily-regenerated forecast page tied to market data
- [/safest-countries](https://the-world-now.com/safest-countries) — composite ranking page

Each tracker is a single Next.js dynamic route that builds N pages from data, where N is anything from 1 to 200. They are server-rendered with revalidation, not statically generated, because the data changes every few minutes.

This is "programmatic SEO" in its most defensible form: the pages aren't templated lorem-ipsum. They are real-time views over a real database. The earthquake tracker's California page tells you what actually happened in California today, with USGS-traceable provenance.

---

## A/B testing in middleware

Most of our hero variants live in `middleware.ts`. The pattern:

```ts
// app/middleware.ts (simplified)
export function middleware(req: NextRequest) {
  const url = req.nextUrl
  if (url.pathname === '/current-wars') {
    const cookie = req.cookies.get('cw_ab')?.value
    const variant = cookie ?? pickVariant(['A', 'B', 'C'])  // 33/33/33
    const res = NextResponse.rewrite(new URL(`/current-wars/${variant}`, req.url))
    if (!cookie) res.cookies.set('cw_ab', variant, { maxAge: 60 * 60 * 24 * 30 })
    return res
  }
}
```

The middleware rewrites to a variant subpath, sets a sticky 30-day cookie, and reports the variant assignment to the analytics layer (DataFast). The variants are real component trees, not just copy swaps — variant C of /current-wars is a stats-first typographic hero with no map at all, because the map was a Largest Contentful Paint disaster on slow connections.

The A/B framework is ~80 lines of TypeScript including the cookie logic and the hash-based assignment. There is no third-party experimentation tool. We ship variants in the same PR as the feature.

---

## Catalyst: predictions as a feature

[/catalyst](https://the-world-now.com/catalyst) is an experimental product — AI-generated predictions about geopolitical and market events, scored against outcomes after the fact. It exists because the same event database that drives the articles can also drive forward-looking analyses, and because predictions are inherently shareable in a way that retrospective news isn't.

Engineering-wise, it's a separate Lambda that runs once a day:

1. Cluster open / unresolved event chains.
2. Generate prediction text + a confidence score + a resolution date.
3. Store as `predictions` rows linked to source events.
4. When the resolution date passes, a second Lambda evaluates outcome and updates a `score` field.

Catalyst is the part of the platform that keeps getting cited by ChatGPT and Copilot, because the predictions are timestamped, scored, and easy for an LLM to reference. Building "AI-friendly" content is a real lane and we underweighted it for the first three months.

---

## Stripe + soft paywall

Free everything until we hit 5,000 daily active users, then a soft paywall on Catalyst (3 free predictions per day) at the 5K mark. The paywall component is gated by a simple feature flag tied to the user's subscription tier. Stripe webhook updates the tier on `customer.subscription.updated` and `customer.subscription.deleted`. The implementation is intentionally boring — we don't want to debug paywalls, we want to debug news.

---

## What we'd do differently

In rough order of regret:

1. **Start link building on day one.** We sat at DR 0.2 for three months because we assumed quality content would attract links. It does not. The short version: submit to 100 directories in week one, not month four.
2. **Optimize for Bing earlier.** We added IndexNow at month two. It should have been at hour two.
3. **Cache LLM prompts from day one.** We added Anthropic prompt caching three weeks in. It cut article-generation cost by ~70%.
4. **Schema markup before pretty UI.** The `NewsArticle` schema affected SEO before the visual redesign affected anything.
5. **Track the AI assistant referrers separately.** ChatGPT, Copilot, and Perplexity all show up as direct or referral traffic by default. Tagging them properly took a week of analytics surgery; we should have done it on launch.

---

## Stack summary

| Layer | What we use |
|------|-------------|
| Frontend | Next.js (App Router) on Vercel, Tailwind, Mapbox |
| Database | MongoDB Atlas |
| Compute | AWS Lambda (Node.js), Vercel functions |
| LLM | Claude (Sonnet for article gen, Opus for editorial passes) with prompt caching |
| Email | Resend + custom briefing Lambdas |
| Payments | Stripe Checkout |
| Search submission | IndexNow (Bing), Google Search Console |
| Analytics | GA4 + DataFast + Bing Webmaster Tools + custom event collection |
| Monitoring | CloudWatch alarms on Lambda error rates |

Most of these are obvious choices. The interesting one is using Claude on both ends of the article pipeline — Sonnet for cheap, fast generation; Opus for the editorial pass that catches hallucinations and rewrites weak intros. The cost difference disappears once prompt caching kicks in, because the editorial pass shares 90% of its prompt with the generation pass.

---

## Try the site

If you want to see what 6,000 AI-generated news articles plus a real-time conflict tracker plus market-event predictions look like in practice:

- **Homepage:** https://the-world-now.com
- **Conflict tracker:** https://the-world-now.com/current-wars
- **Earthquake feed:** https://the-world-now.com/earthquakes-today
- **Catalyst predictions:** https://the-world-now.com/catalyst
- **Newsletter:** sign up from the homepage; weekly digest

Questions, ideas, or pushback — open an issue on this repo or DM me on X.

---

*Built and maintained by one developer. Last updated May 2026.*
