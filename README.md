# Ecommerce scraper guide: how much ScraperAPI costs per 1,000 product pages, which plan you need, and what it can't fetch

ScraperAPI hands you HTML. You send a URL, it handles proxies, retries and bot checks, and what comes back is the page's source, not a table of products. That single fact decides whether it fits your project: if you have someone who can write and maintain a parser, you're paying for the part of scraping that actually breaks. If you need clean product rows with SKUs and stock levels out of the box, this is one layer short of what you want.

The paid entry point is **Hobby at $49/month for 100,000 credits**, and an ordinary page costs one credit. The trial runs 7 days and gives you 5,000 credits without a card.

> **Start on Hobby.** $49/month buys 100,000 credits, which is 100,000 plain HTML pages or 10,000 JavaScript-rendered ones. That covers most price-monitoring and catalog-audit work below roughly 50,000 pages a month, provided you can parse the response. Move up only when rendering, Amazon, or bot-protection surcharges push you past 100,000 credits. 👉 [See Hobby's current price and start the 7-day trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The plans, and what each one is actually for

| Plan | Monthly credits | Price | Billing | Start |
| --- | --- | --- | --- | --- |
| Hobby | 100,000 | **$49/mo** | Monthly or annual (annual costs 10% less) | [Start on Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | Shown on the pricing page | Monthly or annual; pay-as-you-go available | [See Scaling's current allowance](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom quote | Contract | [Ask for Enterprise pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup, Business, Professional, Advanced | Intermediate tiers between Hobby and Enterprise; allowances scale with price | Shown on the pricing page | Monthly or annual | [Compare all plans side by side](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two of those cells are deliberately blank. Credit allowances and prices for the middle tiers shift, and a number copied from a blog post is worse than no number. The live pricing page and the Domain Cost Estimator inside the dashboard are the authoritative sources, and both are one click from the trial.

What's fixed across tiers is the multiplier system. Your plan size doesn't change what a request costs; the target domain does. That's the part people get wrong when they budget.

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon | 5 |
| Google or Bing, including subdomains | 25 |
| LinkedIn | 30 |
| Bot-protection bypass (Cloudflare, DataDome, PerimeterX) | +10 on top of the base cost |
| JavaScript rendering (`render=true`) | 10 |
| JS rendering with premium proxies | 25 |
| JS rendering with ultra premium proxies | 75 |

The bypass surcharge only applies when ScraperAPI actually runs its bypass logic, so an identical URL can cost 1 credit on Monday and 11 on Tuesday. Budget against the higher figure for any domain you know is protected.

## What the API hands back, and what you still have to build

A response is HTML. Everything downstream of that is your code: finding the price element, normalising currency, deciding what an empty field means, deduplicating variants, storing history.

The gap is easiest to see against a managed actor, which returns parsed rows instead. One such actor ships 29 flat columns per product: `title`, `price` as a number, `currency` as an ISO code, `originalPrice` pulled from the compare-at or strikethrough price, `discountPercent` calculated and rounded from those two, `inStock`, `condition` (New/Used/Refurbished), `seller`, `variantCount`, `specifications` as name/value pairs, plus `gtin` and `mpn` where the store publishes them. GTIN and MPN exist for one reason: matching the same physical product across retailers that name it differently. The same actor writes an `error` row with the reason when a page is blocked, so a failed fetch doesn't silently become a fake product.

That's several weeks of parser work you'd be skipping, and it's why managed actors bill per product returned (one on Apify charges around $0.003 per product plus a small per-run fee, on top of platform and proxy costs). ScraperAPI bills per request instead. Neither model is cheaper in the abstract; they're priced against different work.

## The cheap path most people miss

If your targets run on Shopify or WooCommerce, you don't need to render a single category page. Both platforms expose their catalog as JSON:

- **Shopify:** `/products.json` returns up to **250 products per call**.
- **WooCommerce:** the Store REST API at `/wp-json/wc/store/v1/products` returns up to **100 per call**.
- **Everything else:** fall back to schema.org `Product` JSON-LD and CSS selectors, which covers BigCommerce, Magento and custom builds.

A 5,000-SKU Shopify store is 20 requests. At one credit each, that's 20 credits out of 100,000. Rendering the same catalog page by page at 10 credits a request would cost 50,000. Same data, 2,500 times the bill.

Those endpoints are ordinary requests, so they bill at the standard rate unless the store sits behind bot protection and the bypass surcharge kicks in. If a price comes back empty because the store lazy-loads it with JavaScript, the fix is to request the product URL directly and let a JSON-LD extractor read the structured data rather than rendering the category listing.

## Cost per 10,000 product pages, by page type

At Hobby rates ($49 for 100,000 credits), a credit costs about $0.00049. Ten thousand pages works out like this:

| What you're scraping | Credits per page | Credits for 10,000 pages | Cost at Hobby rates |
| --- | --- | --- | --- |
| Static HTML store | 1 | 10,000 | **~$4.90** |
| Store behind Cloudflare or DataDome | 11 | 110,000 | Over Hobby; needs the next tier or pay-as-you-go |
| JavaScript-rendered store | 10 | 100,000 | **$49** (the full monthly allowance) |
| JS rendering plus premium proxies | 25 | 250,000 | ~$122.50 |
| JS rendering plus ultra premium proxies | 75 | 750,000 | ~$367.50 |
| Amazon | 5 | 50,000 | ~$24.50 |

Two things fall out of that table. Rendering is where budgets die, so it's worth confirming a site really needs it before you turn it on. ScraperAPI's own documentation recommends rendering only when nothing else works, because it adds latency and can lower success rates, which in turn cuts how many requests you get through.

The second: 10,000 pages on a static store costs about the price of a coffee, and 10,000 pages on an anti-bot-protected store can exceed your entire monthly allowance. Same page count, 11× the credits.

👉 [Start the free trial and price your own target domains before you pick a tier](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Where runs stall, and what to do about it

**A price field comes back empty.** Usually the store renders prices client-side. Add `render=true`, or use `wait_for_selector` to hold the response until the element exists, or drop to a product-level URL and parse JSON-LD. Do the cheap check first: open DevTools, hit `Ctrl+Shift+P`, type `javascript`, choose Disable JavaScript, and reload. If the price is still there, you never needed rendering.

**You get a 429.** That's the concurrency limit for your plan, and requests over it are turned away rather than queued. There's no charge for the refusal, but your code has to retry with backoff, otherwise you lose the pages silently. Slow down, or move up a tier.

**Your throughput plateaus even with credits left.** Rendering has its own ceiling: a default burst limit of 10 requests per second. At roughly 25 seconds per rendered page, that caps you at around 250 pages in flight regardless of plan size. Higher rendering concurrency is an Enterprise conversation.

**Prices look wrong across countries.** Some stores serve different currencies and languages depending on where the request originates. If you're comparing a US price to a German one and the numbers don't reconcile, check the origin before you check the parser.

**Credits vanish at renewal.** Unused credits expire each month. If your usage is bursty, run the heavy jobs early in the cycle rather than saving them up.

**You run out mid-month.** Behaviour depends on tier. On Hobby, Startup, Business, Scaling and Professional you can turn on auto-upgrade, which moves you to the next plan when credits hit zero. On Scaling, Professional, Advanced and Enterprise, pay-as-you-go keeps requests flowing at a fixed per-credit rate up to a monthly cap you set. Below Scaling, running dry means scraping pauses until renewal.

👉 [Compare the tiers that include pay-as-you-go](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Who should not buy this

**A few hundred SKUs on one Shopify store.** Twenty requests gets you 5,000 products from `/products.json` for free. Write eighty lines of code and skip the subscription entirely.

**Anyone who needs product rows and has no engineering time.** Managed actors on marketplace platforms do platform detection, variant nesting and GTIN matching for you. You pay per product, and the output is a clean table the same day.

**People targeting a marketplace the API doesn't special-case.** Amazon gets custom logic at 5 credits. Other big marketplaces vary: some are reachable, some reject non-browser clients outright, and the only way to know is to run a handful of real URLs through the estimator before committing to an annual plan.

**Anyone with a budget of zero and a JavaScript-heavy target.** Rendering is the expensive path, and there's no free tier that absorbs it.

## FAQ

**Do I need my own proxies?**
No. Proxy rotation and bot-check bypass are part of the request, priced as a surcharge when bypass activates. Premium and ultra premium proxy pools exist as options and change the credit cost, mostly in combination with rendering.

**Do I need JavaScript rendering for every store?**
No, and you shouldn't default to it. Static HTML sites and catalog JSON endpoints cover a large share of ecommerce targets. Test with JavaScript disabled before you spend 10 credits a page.

**How many products can I get per month?**
Divide your allowance by the per-request cost. On Hobby: 100,000 standard pages, 20,000 Amazon pages, 10,000 rendered pages, or roughly 25 million Shopify products if you go through `/products.json` at 250 per call. Concurrency, not credits, is usually the binding constraint on big runs.

**What happens if a page is blocked?**
You get an error back for that URL rather than invented data. Build your pipeline to log the failure with the URL and the reason, and you'll get a usable gap list instead of silent nulls in your price history.

**Is there a refund if I pick the wrong plan?**
Check the terms before you buy annually. The 7-day, 5,000-credit trial exists so you can size your usage first, and the Domain Cost Estimator will tell you what your actual URLs cost before you commit a cent.

## Picking a tier in one step

Work out your monthly page count, multiply by the credit cost of your hardest target domain, and see which side of 100,000 you land on. Under it, with no rendering involved, Hobby is the whole answer. Over it because of rendering or Amazon, go up a tier or turn on pay-as-you-go at Scaling. If your targets are Shopify or WooCommerce and you're comfortable parsing JSON, subscribe at the smallest tier or don't subscribe at all.

Before you order, run twenty real URLs from your actual targets through the Domain Cost Estimator and confirm the trial's 5,000 credits cover a meaningful test. A 10% annual discount only pays off if the plan you chose is the plan you needed.

👉 [Start the 7-day, 5,000-credit trial and size the plan against your own URLs](https://www.scraperapi.com/pricing/?fp_ref=coupons)
