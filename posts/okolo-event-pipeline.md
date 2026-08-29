---
layout: default
title: "From municipal websites to map pins: how Okolo builds its event pipeline"
header_title: "Okolo's event pipeline"
date: 2026-07-26
description: "A technical tour of the crawling, LLM extraction, geocoding, deduplication, and map-serving pipeline behind Okolo."
image: /assets/images/projects/okolo-linz.png
permalink: /posts/okolo-event-pipeline/
---

<div class="article-lead">
  <p>Building an event map is easy when every organizer gives you a clean API. Building one from small-town calendars, municipal CMS templates, RSS feeds, iCal files, and photographed posters is a very different problem.</p>
</div>

<figure class="article-figure article-figure--hero">
  <picture>
    <source srcset="/assets/images/projects/okolo-linz.webp" type="image/webp">
    <img src="/assets/images/projects/okolo-linz.png" alt="Okolo Linz identity with colorful event pins arranged around the city" width="1640" height="624" fetchpriority="high" decoding="async">
  </picture>
</figure>

I'm building [Okolo](https://okolo.events), a family-focused map for discovering what is happening nearby. It started with a simple question: *what can we do with the kids this weekend?* The useful answers are often scattered across dozens of places—a city calendar, a Gemeinde website, a museum programme, a parish page, or a poster on a kindergarten door.

That fragmentation is the product opportunity, but it is also the main engineering problem. An event only becomes useful on a map after we can reliably answer four questions:

- What is it?
- When does it happen?
- Where exactly is it?
- Can we prove where the information came from?

Okolo's pipeline is designed around those questions. It uses deterministic parsers whenever a website exposes structure, an LLM for the irregular long tail, and a deliberately conservative placement system before anything becomes a pin.

<figure class="article-figure">
  <img src="/assets/images/okolo/event-pipeline.svg" alt="Diagram of the Okolo pipeline from public event sources through crawling, extraction, validation, geocoding, deduplication, storage, and map rendering">
  <figcaption>The complete path from a public announcement to a live map pin.</figcaption>
</figure>

## The source registry is the real beginning

The crawler does not start with a list of random URLs. Every repeatable source is registered in Postgres with operational metadata: its URL, region, likely CMS, extraction route, last crawl, content hash, HTTP cache headers, yield history, and health tier.

This matters because discovering a source and crawling it once are not the same as building coverage. A useful source must survive the next run without somebody manually remembering how it worked.

When we expand into a new area, the workflow is:

1. Discover official municipal and local sources.
2. Identify the underlying format or CMS.
3. Register the source and its geographic scope.
4. Run it through the normal crawler.
5. Keep the source only if the process is repeatable.

The registry turns web research into infrastructure. It also lets the crawler make better decisions over time. Productive, frequently changing sources are revisited more often. Quiet sources are checked less often. A source that repeatedly returns nothing is quarantined, but periodically retried so it can recover automatically.

## Crawl politely, and avoid work before optimizing it

Every request goes through one network layer. It identifies the crawler, honors `robots.txt`, respects a per-host delay, and never sends two simultaneous requests to the same municipal server. Different hosts can be processed in parallel, but a small Gemeinde website should never feel that parallelism.

The next optimization is not a faster model. It is skipping extraction entirely.

The crawler uses standard HTTP conditional requests (`ETag` and `Last-Modified`) when a server supports them. A `304 Not Modified` response means there is no page body to download. If the server does not support those headers, Okolo strips the page down to its meaningful text and compares a content hash with the previous crawl. An unchanged page stops there.

That distinction is important:

- Conditional HTTP avoids the transfer.
- The page hash avoids parsing and model work.
- Source cadence avoids the request in the first place.

Most municipal calendars do not change every hour. Treating "nothing changed" as the common case is the largest cost and load reduction in the system.

## A waterfall for different website formats

When a page has changed, the extractor tries the most reliable and cheapest representations first. The first route that produces valid events wins:

```text
direct iCal or RSS feed
→ schema.org JSON-LD
→ schema.org Microdata
→ linked iCal
→ CMS-specific adapter
→ event-aware RSS/Atom
→ LLM fallback
```

Why this order? Structured data already expresses facts as fields. There is no reason to ask a probabilistic model to rediscover a start date that exists in `startDate`, or a venue already present in an iCal `LOCATION`.

The generic routes cover well-behaved sites. The adapter layer handles repeated publishing systems. In Austria, for example, many municipal sites run on GEM2GO. One adapter needs to understand several real-world templates—classic tables, Bootstrap cards, card grids, and accordion lists—but once it does, it can process many municipalities without an LLM call.

Other adapters handle patterns such as:

- DVV municipal RSS feeds with embedded event fields.
- Sitepark feeds that link to one iCal file per event.
- WordPress event collections with iCal exports.
- Two-hop calendars where the listing page only contains links and the facts live on each detail page.
- Festival grids whose date has to be established from the publisher's own page state.

An adapter is more work than a one-off prompt, but it compounds. Every site on the same CMS becomes cheaper, faster, and more predictable.

<div class="metric-chart" role="img" aria-label="Extraction routes in a July 14, 2026 Upper Austria snapshot: GEM2GO 73.8 percent, LLM 23.8 percent, JSON-LD 1.4 percent, not yet classified 0.9 percent">
  <div class="metric-chart__title">Which route won? <span>Upper Austria working-source snapshot · 14 July 2026</span></div>
  <div class="metric-chart__row">
    <div class="metric-chart__label"><strong>GEM2GO adapter</strong><span>158 sources</span></div>
    <div class="metric-chart__track"><span style="width: 73.8%"></span></div>
    <strong class="metric-chart__value">73.8%</strong>
  </div>
  <div class="metric-chart__row metric-chart__row--rose">
    <div class="metric-chart__label"><strong>LLM fallback</strong><span>51 sources</span></div>
    <div class="metric-chart__track"><span style="width: 23.8%"></span></div>
    <strong class="metric-chart__value">23.8%</strong>
  </div>
  <div class="metric-chart__row metric-chart__row--muted">
    <div class="metric-chart__label"><strong>JSON-LD</strong><span>3 sources</span></div>
    <div class="metric-chart__track"><span style="width: 1.4%"></span></div>
    <strong class="metric-chart__value">1.4%</strong>
  </div>
  <div class="metric-chart__row metric-chart__row--muted">
    <div class="metric-chart__label"><strong>Not yet classified</strong><span>2 sources</span></div>
    <div class="metric-chart__track"><span style="width: 0.9%"></span></div>
    <strong class="metric-chart__value">0.9%</strong>
  </div>
</div>

That snapshot captures the reason for the waterfall: almost three quarters of the working sources in that regional pass were handled by one deterministic CMS adapter. The LLM remained essential, but it was no longer paying to solve the same template repeatedly.

## Where the LLM belongs

The web has a long tail that no practical collection of parsers will cover: hand-built pages, inconsistent date blocks, prose-heavy announcements, and layouts that change from one municipality to the next. This is where an LLM is genuinely useful.

Before the call, the HTML is reduced to text and bounded in size. The model is not asked to "understand the website" in the abstract. It receives a narrow extraction job and must return a fixed JSON shape:

```json
{
  "title": "Kinderflohmarkt",
  "date_start": "2026-08-08",
  "time_start": "09:00",
  "venue": "Pfarrzentrum",
  "address": null,
  "town": "Asten",
  "categories": ["family", "market"],
  "is_free": null,
  "age_min": null,
  "age_max": null,
  "indoor": null
}
```

Unknown means `null`. A missing or unreliable date means the event is skipped. Categories come from a closed list. Text stays in the language and script of the source. Times are interpreted as local wall-clock time for the event's region rather than casually converted through the server's timezone.

All provider routing lives behind one extraction module. The crawler and upload routes do not know whether the result came from a local model, Gemini, Claude, or another fallback. That keeps cost and quality decisions in one place and lets the high-volume common case use a cheaper model while difficult inputs can fall back to a stronger one.

The same contract also powers poster scanning. A user photographs or uploads a flyer, the image model extracts the fields with confidence scores, and the person sees an editable confirmation screen before publishing. The model accelerates data entry; it does not get unilateral authority to put an uncertain event on the map.

## Facts in, coordinates out

Extraction can tell us that an event happens at "Bühne 2" or "Pfarrsaal St. Martin." It cannot safely tell us where that is.

Okolo never asks a model for latitude and longitude. Placement runs through a separate geocoding ladder:

1. Reuse a verified venue from the venue registry.
2. Search OpenStreetMap by venue name and expected town.
3. Geocode the published street address.
4. Try the venue and town together.
5. Apply a source's default venue for single-venue publishers.
6. Fall back to the town centroid.

Every precise result is checked against the expected town. A venue name that resolves 100 kilometres away is not accepted simply because a geocoder returned it confidently. If the system cannot prove a precise location, the event stays at town precision and the interface communicates that approximation.

This is one of the most important product decisions in the pipeline: an honestly approximate pin is better than a confidently wrong one.

The venue registry makes placement improve over time. Once a venue and town pair has been resolved and verified, future events at the same place skip the external lookup and land consistently. Enrichment work becomes a reusable fact rather than a one-time correction.

## Deduplication without deleting real events

The same event often appears on a municipal site, a tourism calendar, and an organizer's page. A useful map should merge those copies, but an aggressive deduplicator can be just as damaging as no deduplicator at all.

Okolo uses two layers:

- An exact content hash catches the same normalized title, date, and town.
- A cautious fuzzy match compares candidates only within the same day and place boundary.

The fuzzy path can enrich missing fields, but it does not overwrite known facts casually. Ambiguous title substitutions are rejected. A separate dry-run tool reports wider cross-source duplicates for review before destructive changes are applied.

The underlying rule is simple: inventing a false event is bad, but silently deleting a real one is bad too.

## Storing and serving the map

Validated events live in Supabase Postgres, in a dedicated schema accessed through one data layer. PostGIS handles spatial queries. The application itself is a Next.js app hosted on Vercel, while MapLibre renders OpenStreetMap-based tiles in the browser.

The server does not send the entire catalog to every visitor. The map sends its current bounding box, zoom level, and filters to the events API:

- At neighborhood zoom, PostGIS returns full pins inside the viewport.
- At regional zoom, the server returns pre-aggregated spatial cells.
- Sparse views can still return individual events so a low-density area does not become a map full of meaningless "1" bubbles.
- MapLibre takes over fine-grained clustering and the smooth transition between clusters and pins.

This viewport-native design keeps the response proportional to what the user can see. Panning the map aborts the stale request, waits briefly for the camera to settle, and loads the new area.

The event table is also the source for individual server-rendered event pages, `schema.org/Event` JSON-LD, the sitemap, a paginated public API, and an MCP server. The same cleaned record can therefore be found by a family on the map, a search engine, or an AI assistant—with the original source still attached.

## The legal and trust boundary

The pipeline indexes facts: title, date, place, categories, and other structured attributes. It does not republish source images or copy event descriptions. When a short description is useful, it is written in new words. Every event keeps a `source_url` so the user can verify the details and the publisher receives the linkback.

That boundary is both a legal precaution and a product feature. Event data changes. Times move, venues cancel, and municipal pages get corrected. Okolo should help people discover an event, then make the provenance obvious.

The crawler's other trust rules follow the same philosophy:

- Honor `robots.txt` and explicit bot policies.
- Identify the crawler and provide a contact path.
- Rate-limit requests per host.
- Never fabricate missing facts.
- Reject events with no reliable date.
- Mark approximate placement as approximate.
- Keep anonymous submissions structured and reviewable.

## What I learned

The tempting version of this project is "give every page to an LLM and draw the answer on a map." That works for a demo. It is the wrong architecture for a dependable regional index.

The system became much better when I treated the LLM as one rung in a data pipeline:

- First detect whether anything changed.
- Prefer machine-readable facts.
- Turn repeated website families into adapters.
- Give the LLM a strict, narrow contract.
- Validate every result outside the model.
- Geocode through evidence, never model coordinates.
- Deduplicate conservatively.
- Preserve provenance all the way to the interface.

The result is not a universal web crawler. It is something more useful for this product: a growing collection of repeatable routes from local publishing systems to trustworthy map pins.

That is the technical core of Okolo—one region at a time, and one messy source made reliable at a time.

---

*Explore the live map at [okolo.events](https://okolo.events).*

---

*AI-boosted Georgi: This post was written with AI as an experiment—and because a busy family man has more ideas than uninterrupted writing time. The experience, opinions, and final editorial decisions are mine.*
