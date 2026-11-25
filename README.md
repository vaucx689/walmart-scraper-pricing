# Walmart Scraper: How Do You Pull Prices, Reviews and Best Sellers Without Getting Blocked, and What Does 1,000 Records Actually Cost?

A Walmart scrape gets priced two different ways, and the difference decides your bill. Hosted actors charge per record: $1.00 per 1,000 rows, plus $3.00 per 1,000 product pages when prices are attached. ScraperAPI charges per request: a Walmart page costs 5 credits, so the $49 Hobby plan's 100,000 credits buy 20,000 Walmart requests, which works out to $2.45 per 1,000. Which one is cheaper depends on how many records a single request returns, and that number is different for a search page, a product page and a review page.

What follows is what Walmart actually serves on each page type, what the three main routes cost once you include their limits, and the two questions worth settling before you pay anyone.

> **Short answer for most readers:** a fixed watchlist or a few thousand requests a month fits ScraperAPI's Hobby plan at $49/month (100,000 credits, 20 concurrent threads), and the free trial credits cover 200 Walmart requests at 5 credits each. Move to Advanced ($1,975/month, 21.5M credits, 500 threads, priority routing) only when you're running continuous multi-source pipelines.
> 👉 [See the current ScraperAPI plans and the credit cost of a Walmart request](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, side by side

| Plan | API credits / month | Concurrent threads | Geotargeting | Price | Sign up |
| --- | --- | --- | --- | --- | --- |
| Hobby | 100,000 | 20 | US & EU | $49/mo, or $44.10/mo billed annually ($529.20/yr) | [Hobby pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | Listed on the pricing page | [Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | Listed on the pricing page | [Business pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | Listed on the pricing page | [Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | Listed on the pricing page | [Professional pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | $1,975/mo, or $1,777.50/mo billed annually ($21,330/yr) | [Advanced pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Custom | [Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

The two ends of that range are where the unit economics become visible. Hobby costs about **$0.49 per 1,000 credits**; Advanced costs about **$0.09 per 1,000 credits**. Because Walmart bills at 5 credits per request, that's roughly $2.45 per 1,000 Walmart requests on Hobby and $0.46 on Advanced. The four tiers in between sit somewhere along that curve, priced by credit volume.

Two limits matter more than the headline allowance. Hammering past your concurrency gets a 429 and no extra charge, and once credits run out scraping pauses until renewal unless auto-upgrade is switched on. Pay-as-you-go past the cap only exists from Scaling upward, so a pipeline that can't afford to pause usually starts higher than Hobby.

## What a Walmart page actually gives you

Walmart's search results usually don't carry a price, and any scraper that shows prices in search mode is quietly fetching each product page as well. Apify's own Walmart actor documentation says it outright: search results don't include price, and its `fetch_prices` switch exists to read the product page for price, UPC, brand, model, seller and category. A price-tracking job is therefore two requests per record, not one.

What each page type is good for:

- **Search and category pages** give you item id, title, image, star rating, review count, stock status and a sponsored flag. Cheap, fast, and enough for assortment research.
- **Product pages** carry price, `was_price`, UPC/GTIN, brand, model number, seller, a `sold_by_walmart` flag and a full category path plus specifications.
- **Reviews** come from a separate endpoint or page. Real review records include the star rating, the review text, author, publish date, helpful votes and badges such as Verified Purchase.
- **Best Sellers charts** return one row per rank position per department, with price and review count beside each rank.

UPC is the field worth designing your schema around. It gives an exact join key against Amazon, Target or your own catalogue, which beats fuzzy-matching product titles across marketplaces.

## Three routes, three billing models

| Route | Billing unit | Verified price | Who maintains the parser |
| --- | --- | --- | --- |
| Hosted actor (Apify) | Per record | $1.00 per 1,000 rows + $3.00 per 1,000 priced product pages; a second actor from $1.50 per 1,000 listings | The vendor |
| Scraping API (ScraperAPI) | Per request (credits) | Walmart at 5 credits; $49 buys 20,000 requests | You |
| Self-hosted Scrapy | Your own infrastructure | Proxy traffic plus engineering time | You |

### Hosted actors: cheap to start, thin on history

Apify has several Walmart actors and their economics differ. One charges $0.001 per delivered row, $0.003 per product page actually read, and $0.001 per run start. That prices 600 unpriced search results at about $0.60, 100 results with prices at about $0.40, and 200 reviews at about $0.20. Another asks $1.50 per 1,000 product listings, with product details and reviews billed separately at $1.50 per 1,000 each.

Three limits are documented rather than hidden. The actor keeps no deduplication state between runs, so merging two pulls of the same keyword requires you to dedupe on `usItemId` with `url` as a fallback. It keeps no historical snapshots, so price history is your database's job. And when Walmart changes its markup, individual fields come back empty while the row still gets written, which means your pipeline has to tolerate partial records. Failed detail lookups don't discard the listing row already collected. The free Apify plan caps each run at 10 rows, which makes it a demo rather than a trial.

### Scraping APIs: per-request pricing rewards parser work

ScraperAPI returns HTML for any Walmart URL and also exposes structured Walmart endpoints, which changes the cost math. A search-page request can carry dozens of listings, so at 5 credits per request your cost per record falls the more you parse out of each response. You pay for that in parser maintenance, and in the fact that raw search pages are the least stable part of Walmart.

The structured side covers two jobs. The product endpoint is asynchronous: send a single `productId` or an array of `productIds`, set `tld` to `com` or `ca`, add a `country_code`, pick JSON or CSV output, and hand it a webhook callback to collect results. The reviews endpoint takes a `product_id`, a sort order, a star filter as a list from 1 to 5, a `verified_purchase` flag and a page number, and returns the product name, average rating, review count and the individual reviews.

Two habits keep the bill predictable. Every response carries a `sa-credit-cost` header showing what that call actually cost, and `max_cost` caps any single request. A Walmart page that needs JS rendering or premium proxies costs well above the 5-credit base, so check the domain cost estimator in the dashboard before a large run rather than after.
👉 [Test a Walmart product lookup with the free trial credits](https://www.scraperapi.com/pricing?fp_ref=coupons)

### Parsing APIs as a middle ground

Oxylabs' Walmart scraper takes a URL through its universal source with `parse: true` and returns structured output for product, search and offer-listing pages: SKU, GTIN, price, title, rating object, breadcrumbs, description, a specifications array (brand, size, manufacturer, dimensions) and a `parse_status_code`. Its output also includes `_warnings` entries when it can't determine shipping or seller details, which is a useful reminder that parsed output is best-effort even from a paid parser. The published sample record dates from 2022, so treat it as a description of output shape, not of current pricing.

### Self-hosted Scrapy: the proxy bill is the small part

There's a MIT-licensed Walmart Scrapy spider that ships with ScrapeOps proxy support and a 1,000-request free tier, good for development. Its own documentation rates Walmart scraping difficulty at **9 out of 10**: CAPTCHA challenges, deliberately irregular CSS class names, JavaScript-rendered content and IP rotation requirements. Defaults are one concurrent request and a three-second download delay, so a 50,000-page crawl is measured in days unless you scale concurrency and pay for proxies to match.

The legal note in that same project is the part people skip. Walmart's Terms & Conditions prohibit automated data collection without explicit permission, while robots.txt permits crawling under specific whitelisted user agents. The project labels itself educational and research-only, and puts compliance responsibility on the user. Running your own spider doesn't change those terms; it just moves the enforcement risk onto your IPs.

## Deduplication and price history are not included

Neither hosted actors nor raw APIs hand you a clean time series. `usItemId` is the stable key across runs, with `url` as a fallback, and you need it in a unique constraint before your first scheduled run. A daily priced search over a narrow query or category URL, storing `price` and `was_price` per item per day, is a complete discount-alert system; `was_price` only appears when a markdown is real, so a sudden drop in that field is the signal. Without your own snapshot table, you have yesterday's data and today's, and no way to prove what changed.

## Which route fits your job

- **A few hundred listings, once.** Use a hosted actor. The per-record price on 500 rows is under a dollar on either Apify actor, and you skip parser work entirely.
- **Daily price monitoring on a known SKU list, feeding your own database.** Use a scraping API. Per-request billing plus your own parser is the cheapest path at volume, and 20,000 Walmart requests on Hobby covers roughly 650 products checked daily for a month at two requests each.
- **Broad assortment research where prices don't matter yet.** Turn price fetching off. Ratings and review counts are on the search page already, and they tell you which items have traction before you spend a second request per product.
- **Small volume and an engineer with spare time.** Self-host. The 1,000 free proxy requests will get a prototype running, and the maintenance load is real but bounded if you only need public product fields.

One note on the search term itself: "walmart scraper" also returns actual Cricut scrapers and crafting tools sold on Walmart.com. If you landed here looking for a $7.67 mint-colored craft scraper, this isn't that page.

## Two things to confirm before you pay

Check what the meter counts. Per-record pricing and per-credit pricing only become comparable once you convert both to "cost per record I can actually use," which means knowing your requests-per-record ratio. Priced Walmart data runs at roughly two requests per record; unpriced listing data at a fraction of one.

Then check whether your target pages need extras. A Walmart page behind bot protection with JS rendering and premium proxies is billed at a multiple of the base rate, and on a 100,000-request month that multiple is the difference between one plan tier and the next. The free trial's 1,000 credits are 200 Walmart requests, enough to measure your real cost per record on your own URLs before committing to a tier.

👉 [Compare Hobby and Advanced on the ScraperAPI pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons)
