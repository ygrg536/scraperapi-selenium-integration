# web scraping python selenium: why your scraper gets blocked, and how to route it through ScraperAPI without breaking your code

If you searched for "web scraping python selenium", you're most likely in one of two places: writing your first Selenium scraper and wondering why it keeps getting blocked, or maintaining one that worked last month and now returns 403s and CAPTCHA pages. This article covers both: how to wire a scraping API into Selenium correctly (including the integration method that officially doesn't work, which most tutorials still teach), and what ScraperAPI actually costs once you factor in how its credit system really counts requests.

Quick verdict first: for most people scraping with Python and Selenium, **ScraperAPI's Hobby plan ($49/month, or $44.10 billed annually) is the realistic entry point** — the free tier is enough to verify a target site works, and every paid plan includes the same feature set, so you only pay for volume and concurrency. You can test it without a credit card: 👉 start the 7-day trial with 5,000 credits and see if your target site passes.

> **Fast lane**
> - Recommended plan for solo developers: Hobby, $49/month (100,000 credits, 20 concurrent threads)
> - Free to try: 7-day trial, 5,000 credits, no credit card required
> - Best for: Selenium/Scrapy/Requests scrapers that break on CAPTCHAs, IP bans, or JavaScript-rendered pages
> - Not for: pure static-site scraping (requests + a parser is cheaper), or tiny one-off scripts you run once a year

## Why a plain Selenium scraper stops working

Selenium drives a real browser, so it can run JavaScript, click buttons, and fill forms — that's exactly why people use it for pages where plain `requests` gets an empty shell of HTML. The trade-off is that a real browser also produces a very visible fingerprint: the default headless Chrome user agent, automated-driver flags, and repeated requests from a single residential IP all make it easy for a site to tell what's happening.

Once a site flags you, the usual symptoms are CAPTCHA walls, 403 responses, or HTML that renders fine but contains none of the data you wanted. The standard DIY responses — rotating residential proxies, headless-browser stealth patches, retry logic, CAPTCHA-solving services — each work partially, and managing all four together is a part-time job.

ScraperAPI collapses that stack into one endpoint. You send a URL; it routes the request through a rotating pool that includes premium residential and mobile IPs, renders JavaScript with a headless browser when asked, handles CAPTCHAs and anti-bot systems (Cloudflare, DataDome, PerimeterX, Akamai are the named ones), and returns the HTML or parsed JSON. You pay per request, not per GB of bandwidth.

The relevant part for Selenium users: ScraperAPI offers a **proxy port mode**, which means your Selenium code keeps running exactly as it does now — you just hand the webdriver a proxy. No rewriting your scraping logic.

## How to connect ScraperAPI to your Python Selenium scraper

Use the proxy port method, not the API endpoint. Here's the working setup, following ScraperAPI's own integration guide.

Install Selenium Wire first — vanilla Selenium doesn't handle authenticated proxies gracefully, and Selenium Wire does:

python
pip install selenium-wire


Then configure the proxy and pass it into the webdriver:

python
API_KEY = 'YOUR_API_KEY'

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=option,
    seleniumwire_options=proxy_options
)


The proxy address is `proxy-server.scraperapi.com:8001`, the username is always the literal string `scraperapi`, and the password is your API key. From that point on, every request Selenium makes goes through ScraperAPI's pool, and proxy rotation, CAPTCHA handling, and retries happen automatically — the docs state the proxy port delivers the same features and performance as the API endpoint.

ScraperAPI's optional features work in proxy mode too, by appending parameters to the username with dots. For example, `scraperapi.render=true.country_code=us` turns on JavaScript rendering and routes the request through a US exit IP at the same time. This is how you'd combine Selenium's own browsing behavior with server-side rendering on tough pages.

One detail that trips people up: **through the proxy port, you must disable SSL verification** (`verify=False` in requests-style libraries, and the equivalent setting in Selenium Wire's context). ScraperAPI's docs are explicit that requests fail otherwise. If you're not comfortable disabling SSL checks, the alternative is downloading ScraperAPI's proxy CA certificate from `api.scraperapi.com/proxyca.pem` and trusting it at the system level — the docs walk through the steps for Windows (via the Microsoft Management Console, importing into Trusted Root Certification Authorities), macOS (Keychain Access → set the `ScraperAPI CA` entry to Always Trust), and Linux.

If you want a reference implementation before writing your own, ScraperAPI maintains full Selenium examples on GitHub, including a working scraper for `quotes.toscrape.com` built on this exact proxy setup.

### The integration mistake that quietly breaks Selenium

Here's something most Selenium scraping tutorials never mention, and it's the single most useful thing in ScraperAPI's own guide: there are two ways to plug ScraperAPI into Selenium, and the more "obvious" one is broken by design.

The obvious approach is rewriting your target URL into ScraperAPI's API endpoint — urlencode your URL and API key into an `api.scraperapi.com` address and call `driver.get()` on it. It feels simpler, and it half-works. But it produces two failures the official documentation spells out:

1. **Static resources 404.** When Selenium loads `quotes.toscrape.com`, the page also requests `/static/main.css`. Normally that resolves to `quotes.toscrape.com/static/main.css`. Through the API endpoint, Selenium instead requests `api.scraperapi.com/static/main.css`, which errors out. The result: Selenium only ever gets the first HTML response, which defeats the entire point of driving a headless browser.
2. **Relative links die.** Any simulated user behavior — clicking through pages, navigating the site — relies on relative links, and those resolve to the API domain for the same reason.

ScraperAPI's own wording on this: *"we highly recommend that you don't use the API endpoint if you want to integrate a Python Selenium scraper with the API."* The guide still publishes an API-endpoint example for the stubborn, but the proxy port is the method that actually preserves what Selenium is for.

## What ScraperAPI costs — all plans side by side

ScraperAPI has eight tiers, and the feature list is identical across every paid plan: JavaScript rendering, premium residential and mobile IPs, CAPTCHA and anti-bot bypassing, auto-parsing APIs, the DataPipeline batch runner, crawler access, custom headers and sessions, automatic retries, and unlimited bandwidth. What changes is credits, concurrency, geotargeting reach, and analytics history retention. Annual billing takes 10% off every paid plan.

| Plan | Monthly price | Annual (−10%) | API credits/mo | Concurrent threads | Purchase |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | $0 | 1,000 | 5 | [ 免费注册获取 1,000 credits](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ 开通 Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ 开通 Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ 开通 Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ 开通 Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ 开通 Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ 开通 Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22M+ | 500+ | [ 联系销售获取报价](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

Two limits worth flagging before you pick: **Hobby and Startup are locked to US & EU geotargeting only** — country-level targeting outside those regions requires Business ($299) or higher. And analytics history on Hobby and Startup is capped at 30 days; Business and up keep it indefinitely.

The trial path is generous enough to actually evaluate the product: 7 days with 5,000 credits, no credit card. After that you drop to the permanent free plan (1,000 credits, 5 concurrent connections), and ScraperAPI says you can contact support for extra test credits if you need more headroom. Paid plans carry a 7-day refund policy, and you can cancel from the dashboard at any time. If you want to compare the annual math against monthly before committing, 👉 the pricing page shows both billing cycles side by side.

## The real cost: why "1 million credits" is not 1 million pages

This is the part most reviews gloss over, and it changes which plan makes sense. ScraperAPI doesn't charge 1 credit per request uniformly — harder targets cost more:

| Request type | Credits per request |
| --- | --- |
| Normal page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google, Bing | 25 |
| LinkedIn | 30 |
| Cloudflare / DataDome / PerimeterX / Akamai bypass | +10 |
| `render=true` (JavaScript rendering) | 10 |
| `premium=true` (residential/mobile IPs) | 10 |
| `premium=true` + `render=true` | 25 |
| `ultra_premium=true` | 30 |
| `ultra_premium=true` + `render=true` | 75 |

`country_code`, `session_number`, `device_type`, `output_format`, and `autoparse` add nothing extra.

The practical consequence: a Startup plan's 1,000,000 credits is 1,000,000 plain pages, or 100,000 rendered pages, or 40,000 Google searches, or roughly 13,000 ultra-premium rendered pages. If your Selenium targets are mostly e-commerce and search pages, divide the headline credit number by 5 to 25 before comparing it against competitors' "request" counts.

Three guardrails keep this under control, and none of them are advertised on the pricing page itself:

- **Domain Cost Estimator** — in the dashboard, check any URL's exact credit cost before you scale up a scrape.
- **`max_cost` parameter** — set a per-request spending cap; if a request would exceed it, the API returns a 403 instead of silently charging more.
- **Charging only on outcomes** — you're billed for successful requests (HTTP 200 and 404) and requests you actively cancel within a 70-second window. Failed fetches don't burn credits.

## Billing rules that surprise people later

Four rules from the docs and terms determine what the second and third month actually look like:

1. **Credits don't roll over.** Every billing period resets the balance; unused credits vanish.
2. **Running out pauses scraping by default.** Requests stop until your next renewal. On Hobby through Professional you can opt into auto-upgrade, which bumps you to the next tier automatically when credits run dry. On Scaling and above, pay-as-you-go lets you keep scraping past your allowance at a fixed per-credit rate, capped by a monthly limit you set.
3. **Concurrency is hard-capped.** Exceed your plan's thread limit and the API returns HTTP 429 — the request is refused, not queued, and not billed.
4. **Overage billing only exists on Scaling and up.** On Hobby, Startup, or Business, an empty balance means upgrading or waiting; there's no metered overflow.

## Who this setup is for — and who should skip it

Not every Selenium project needs this. A quick sorting:

- **You're scraping mostly static, unprotected pages.** Keep `requests` plus a parser. Paying a scraping API to fetch pages your own library handles fine wastes money.
- **Your target blocks bots, serves CAPTCHAs, or needs JavaScript rendering, and you're already committed to Selenium.** This is the core use case. The proxy port means zero restructuring of your code.
- **You're scraping Google, Amazon, or LinkedIn at volume.** ScraperAPI works, but budget by real credit costs, not headline credits. For LinkedIn-heavy workloads the math gets steep fast — 30 credits per request means even the Startup plan covers about 33,000 LinkedIn pages.
- **You need geotargeting outside the US and EU on a small budget.** Business is the floor for true country-level targeting; Hobby and Startup won't get you there.
- **You want no-code scraping.** The DataPipeline runner (up to 100,000 URLs per project, CSV/webhook input, scheduled runs) exists, but that's a different product experience than writing Selenium scripts. There are also structured-data endpoints returning parsed JSON for Amazon, Google Search/News/Jobs/Shopping/Maps, Walmart, eBay, Home Depot, Redfin, and several AI answer engines, plus integrations with LangChain, LlamaIndex, n8n, Zapier, and an MCP server.

One more option if your budget is zero and your targets are easy: the free plan's 1,000 monthly credits and 5 concurrent connections will confirm whether a site is scrapeable at all, which is often the answer you need before deciding anything.

## FAQ

**Does ScraperAPI work with headless Chrome and Selenium 4?**
Yes — the proxy port sits between your webdriver and the network, so it's independent of your Selenium version or headless mode. The integration only requires Selenium Wire for clean proxy authentication, and the official examples target current Selenium setups.

**What happens if I exceed the concurrent thread limit?**
The API returns an HTTP 429 and refuses the request. You're not charged, and requests don't queue — you simply need fewer parallel threads or a higher plan.

**Can I keep my SSL verification enabled with the proxy port?**
Not out of the box. The docs require `verify=False` through the proxy, or manually trusting ScraperAPI's proxy CA certificate, which is downloadable from the API domain and installable on Windows, macOS, and Linux.

**Do unused credits carry over to next month?**
No. Credits reset each billing period. If that worries you, the pay-as-you-go mode on Scaling and higher plans, or auto-upgrade on lower tiers, are the two official ways to avoid losing throughput mid-month.

**Is there a refund if the product doesn't work for my target sites?**
Paid plans come with a 7-day refund window, and cancellation is self-serve from the dashboard. Since the trial requires no credit card, the cheapest way to find out is testing first and paying second.

## Picking a plan before you click buy

Map your situation to a tier:

- Testing whether your target site is even scrapeable → free trial, 5,000 credits, no card.
- Solo developer, mostly regular pages, occasional rendering → Hobby at $44.10 annually or $49 monthly is enough; the 20-thread ceiling suits one Selenium instance comfortably.
- Production workload, 500K+ pages monthly, or targets outside US/EU → Startup if you stay in US/EU, Business if country-level geotargeting matters — that's the real reason to jump to $299, not the credit volume alone.
- Sustained multi-million-page operation with overflow needs → Scaling or above, where pay-as-you-go and 200+ threads start to matter.

Before paying anything, run your actual target URLs through the dashboard's Domain Cost Estimator — a "100,000 credits" plan means something very different for a Google-scraping project than for a blog-crawling one. When you're ready to test with your own scraper, 👉 the free trial signup takes about a minute and asks for no payment details, which is the honest way to find out whether your Selenium blocks disappear.
