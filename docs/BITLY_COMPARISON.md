# Linky vs Bitly

An honest feature-by-feature comparison of Linky and Bitly.
Bitly facts come from Bitly's own pricing, product and API pages (read October 2026). Anything
Bitly does not state publicly is marked **not published** rather than guessed.

**The one-sentence framing:** Bitly is a link-and-QR *marketing platform* with a decade of product
breadth, teams and enterprise features. Linky is a focused, self-hostable shortener that implements
the **core engine** (shortening, redirects at speed, analytics, access control, API) with
production-grade engineering, and no artificial limits or ads.

## 1. Features

| Capability | Bitly | Linky |
|---|---|---|
| Short links | Yes. Free plan: 5 links/month; Core 100; Growth 500; Premium 3,000 | Yes, unlimited |
| Custom back-half / alias | Yes (3/mo on Free, more on paid) | Yes, unlimited |
| Custom / branded domains | Yes, from Growth | **No** (single domain) |
| Edit a link's destination later ("redirects") | Yes, from Core, with monthly limits | **Yes**, unlimited; code, QR and analytics stay the same |
| Link expiry | Not listed on the pages read | **Yes** (date picker, expired page, badge) |
| Password-protected links | Not listed on the pages read | **Yes** |
| One-time / self-destructing links | Not listed on the pages read | **Yes** (atomic, race-free) |
| Private (owner-only) links | Not listed on the pages read | **Yes** |
| QR codes | Yes, dynamic and customisable, up to 200/mo on Premium | **Yes**: custom colours, size, margin, PNG/SVG, contrast warning; unlimited |
| Dynamic QR (change target after printing) | Yes | **Yes**: the QR encodes the short link, whose destination is editable |
| Click and scan analytics | Yes; retention 30 days (Core) to 1 year (Premium), 2 years on Enterprise | Yes; browser, OS, device, referrer, time series; retained indefinitely, no tiers |
| Geography in analytics | Country and city data (city on Premium) | **Yes**: country and city, from a MaxMind GeoLite2 or DB-IP database you supply |
| UTM builder / campaign tracking | Yes | **Yes** UTM builder (no campaign-level dashboards) |
| Dynamic routing (by country, device, platform) | Yes | **Yes**: device, OS and country rules (country needs a CDN country header) |
| Mobile deep linking | Premium | No |
| Link-in-bio pages ("Pages") | Yes | **Yes**: public `/p/slug` pages whose buttons are tracked links |
| Digital business cards, 2D barcodes | Yes | No |
| Bulk creation | Yes (100 to 3,000 per upload by plan) | **Yes**: up to 100 per request, per-row results |
| Tags / organisation | Folders, campaigns, groups | Tags, favourites, archive, duplicate |
| Teams, groups, permissions, SSO | Enterprise | No (single-user accounts plus admin role) |
| Public API | Yes, v4 REST, bearer token. Free 1,000 requests/mo up to Premium 50,000 | Yes, REST plus API keys, no monthly cap (rate-limited per minute) |
| Webhooks | Yes (Enterprise-grade volume) | **Yes**: signed, retried, SSRF-safe; `link.created` and `link.milestone` events |
| Integrations (Shopify, Canva, HubSpot) | Yes | No |
| AI features (Assist, MCP server) | Yes | No |
| Browser extension, mobile app | Yes | No |
| Ads on free links | Free plan shows ads | None |
| Self-hosting / open source | No | **Yes** (MIT) |
| Uptime SLA | 99.99% (Enterprise) | None |

**Where Linky is genuinely ahead:** no usage caps, no ads, self-hostable and auditable, and
access-control link types (password, one-time, private) that Bitly's public pages do not list.
**Where Bitly is far ahead:** everything on the marketing-platform side (custom domains, editable
redirects, dynamic QR, campaigns, link-in-bio, teams/SSO, integrations, SLA).

## 2. How each is built

**Bitly**: its current production architecture is **not published**. What is public: Bitly created
the NSQ message queue (open source) for its own high-volume event pipelines, and the public API is
a versioned REST API at `api-ssl.bitly.com` with bearer tokens, grouped into bitlinks, groups,
organisations, custom domains, QR codes, campaigns, webhooks and sites. Nothing beyond that is claimed here.

**Linky:**

| Layer | Linky |
|---|---|
| API | Spring Boot 3.5 / Java 21, stateless, JWT (+ httpOnly cookie for private-link navigation) and API keys |
| Data | PostgreSQL (source of truth), Flyway migrations, Hibernate in validate mode |
| Redirect path | L1 Caffeine -> L2 Redis (+ negative cache) -> Postgres; single-flight loading; atomic one-time claim |
| Analytics | Redis Streams, consumer groups, batched JDBC inserts, at-least-once with reclaim and dead-lettering |
| Abuse control | Redis Lua rate limiter on creation, auth and unlock; trusted-proxy-aware client IP |
| Observability | Micrometer and Prometheus, request-id correlation, JSON logs |
| Frontend | Next.js 16 (App Router), React 19, Tailwind v4, TanStack Query, React Hook Form + Zod |
| Delivery | GitHub Actions: tests, real Postgres+Redis end-to-end job, image scan, CodeQL, gated deploy with smoke test |

Measured, single laptop: about 9.4k redirects/s, p99 ≤ 17 ms server-side, 100% of clicks recorded.
Bitly's scale is orders of magnitude larger and not something to compare against directly. The
honest claim is that Linky uses the **same architectural pattern** (cache-first read path, async
analytics pipeline) that large shorteners use, at a scale one machine can demonstrate.

## 3. UI and UX

Bitly's logged-in dashboard could not be inspected (it needs an account; its help pages block
automated access), so this compares what is publicly visible.

| Aspect | Bitly | Linky |
|---|---|---|
| Marketing site | Mega-menu (Platform, Solutions, Pricing, Resources), customer logos, case studies, mock dashboards, exit-intent sign-up modal | Single landing page with an **anonymous quick-shorten box** (no sign-up needed) |
| First-run friction | Sign-up pushed hard; free tier is tightly capped | Try it instantly, anonymous links allowed |
| Visual identity | Illustration- and screenshot-led, broad corporate marketing look | Distinct "Ink & Ember" theme: warm graphite neutrals, one amber accent, Manrope plus JetBrains Mono; light and dark |
| App navigation | Product areas: links, QR codes, Pages, analytics, campaigns | Sidebar: Overview, Links, Tags, API Keys, Settings, Admin |
| Create flow | Create link or QR, AI-assisted | One dialog: alias, visibility, password, one-time, tags |
| Analytics | Dashboards across all links and QR codes, with plan-gated depth | Per-link time-series chart plus top browsers, devices, referrers |
| Localisation | 10 languages | English only |
| Accessibility and responsive | Skip-navigation link, mature responsive web and mobile apps | Responsive with mobile nav; needs a formal accessibility audit |

**UI gaps worth closing, in order of impact:** show badges for one-time, password and expiry on
each link row (the API now returns the flags), a link-detail page with a world map once GeoIP is on,
a QR customisation panel, bulk create from CSV, and an empty-state "first link" walkthrough.

## 4. Why use Linky instead of Bitly?

- Bitly's free plan caps you at 5 links a month and shows ads; Linky has neither.
- You control the data: analytics are kept as long as you like, and visitor IPs are stored only as hashes.
- It is self-hostable and open source, so you can read and audit every layer.
- Honest limits: no custom domains, no teams or SSO, no integrations, no SLA. The click pipeline, cache invalidation and webhooks were designed to support those next.

## 5. What is still missing

Implemented since the first version of this comparison: editable destinations, link expiry, UTM
builder, device/OS/country routing, bulk create, QR customisation, link-in-bio pages, geo analytics
(country and city) and webhooks.

1. **Custom domains** per user: the biggest product gap (needs DNS verification and host routing).
2. **Teams** with roles, then SSO.
3. **Integrations** (Zapier, Shopify) and a browser extension. Webhooks are the foundation for these.
4. **Campaign-level analytics** across links, and analytics export (CSV).
5. **More webhook events** (`link.expired`, per-click with sampling) and encrypting webhook secrets at rest.
