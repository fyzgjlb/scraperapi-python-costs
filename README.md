# ScraperAPI Python: SDK, API endpoint or proxy port? What a render call actually costs, and why `client.get()` stalls on 50 URLs (credit table + working code)

The shortest working setup is three lines: `pip install scraperapi-sdk`, put your key in an environment variable called `SCRAPERAPI_API_KEY`, then call `client.get(url="https://example.com/")`. ScraperAPI rotates proxies and headers, retries failed attempts, detects bans and handles CAPTCHAs on its side, and returns the page HTML. New accounts start with 5,000 free credits and no credit card.

What the quick-start pages skip is the part that decides whether your script runs for a week or dies on day two: which of the three integration routes fits your code, what a request actually costs in credits, how to run batches when the SDK has no async mode, and the handful of settings that produce an exception or a silently empty response.

> **Quick pick:** Hobby, $49 a month for 100,000 credits. That is 100,000 plain page requests, or 10,000 once you switch JavaScript rendering on. If you are still evaluating, the free 5,000 credits cover 500 rendered pages or 5,000 plain ones. 👉 [Start with the 5,000 free credits](https://www.scraperapi.com/?fp_ref=coupons)

## ScraperAPI plans at a glance

ScraperAPI sells monthly credits, and the parameter you send decides how many credits a single page burns. Read this table together with the credit table further down, because the two multiply.

| Plan | Credits per month | Price per month | Cost per 1,000 standard pages | Sign up |
| --- | --- | --- | --- | --- |
| Free trial | 5,000 (one-off) | $0 | — | [Free credits](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | $49 | $0.49 | [Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 | $149 | $0.15 | [Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 | $299 | $0.10 | [Business](https://www.scraperapi.com/?fp_ref=coupons) |

Above Business, ScraperAPI lists Scaling, Professional, Advanced and Enterprise. Those tiers add pay-as-you-go overage, higher concurrency and priority support, with the monthly figures published on the pricing page. Annual billing works out cheaper per month than the numbers above.

Two pricing traps are worth modelling before you commit.

The Startup-to-Business step is steep. A workload that needs 1.1 million credits misses Startup's 1 million allowance, and there is no overage option below the Scaling tier, so the bill moves from $149 to $299 for 10% more usage. Round your monthly credit estimate up before choosing.

Rendered pages are where budgets actually break. At 10 credits each, Startup's 1 million credits buy 100,000 rendered pages, which is $1.49 per 1,000. Compare that figure against your real page volume rather than against the headline plan price. 👉 [Check the current plan ladder](https://www.scraperapi.com/?fp_ref=coupons)

## Three ways to call ScraperAPI from Python

All three routes reach the same proxy pool and accept the same request parameters. Pick by how your code is already structured.

### The Python SDK: least code

python
import os
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient(os.environ["SCRAPERAPI_API_KEY"])
html = client.get(url="https://example.com/", params={"render": True})


The package needs Python 3.7 or newer. Retries are built in, three by default, and you can raise the count with `retry=5` on the call. Keep the key in an environment variable; ScraperAPI's own guidance is blunt about never hardcoding it.

One warning for anyone following an older tutorial: the module name changed. Guides from 2021 import `from scraper_api import ScraperAPIClient`, while the current package is `scraperapi-sdk` with `scraperapi_sdk` as the module. Copy the old line and you get an `ImportError` before a single request goes out.

### The API endpoint: most portable

python
import os
import requests

API_KEY = os.environ["SCRAPERAPI_API_KEY"]

def fetch(url, **params):
    r = requests.get(
        "https://api.scraperapi.com/",
        params={"api_key": API_KEY, "url": url, **params},
        timeout=70,
    )
    r.raise_for_status()
    return r.text


Use `https://`. Some 2021 examples still show `http://api.scraperapi.com/`, and the documentation has moved on since.

### The proxy port: drop-in for existing code

python
import requests

proxy = "https://scraperapi:YOUR_API_KEY@proxy-server.scraperapi.com:8001"
r = requests.get(
    "https://example.com/",
    proxies={"http": proxy, "https": proxy},
    verify=False,
    timeout=70,
)


`verify=False` is not optional in Requests here. Without it, certificate validation fails against the proxy and the request never reaches the target. Scrapy skips certificate checks by default, so the same proxy string inside a spider's `meta` needs no such flag.

Inside a Scrapy spider, the SDK route is one method call:

python
yield scrapy.Request(client.scrapyGet(url=url), callback=self.parse)


## What each request costs, and how to escalate

| Request type | Credits | What 5,000 free credits buy |
| --- | --- | --- |
| Standard | 1 | 5,000 pages |
| `render=True` | 10 | 500 |
| `premium=True` | 10 | 500 |
| `premium=True` + `render=True` | 25 | 200 |
| `ultra_premium=True` | 30 | 166 |
| `ultra_premium=True` + `render=True` | 75 | 66 |
| Structured Data (most verticals) | 1–5 | 1,000–5,000 |

The official advice is to start without `render=True` and add it only when the response is missing content you expected. Rendering multiplies cost by ten and adds latency, which makes it the single most expensive habit in a scraper.

When a target still blocks you, climb one rung at a time and stop at the first tier that returns real HTML:

python
def scrape_with_escalation(url):
    for params in [
        {},                                   # 1 credit
        {"render": True},                     # 10
        {"premium": True},                    # 10
        {"premium": True, "render": True},    # 25
        {"ultra_premium": True},              # 30
    ]:
        result = client.get(url=url, params=params)
        if result and "<html" in result.lower():
            return result
    return None


Log which tier succeeded for each domain. The next run can start there instead of paying for four failed attempts.

Three parameter rules cause quiet waste. `premium` and `ultra_premium` are mutually exclusive, so never set both. `ultra_premium` ignores `keep_headers`, which means custom headers and the top proxy tier cannot be combined. `country_code` costs nothing extra, so there is no reason to strip geotargeting to save credits.

For tuning a new target, cap the damage per request with `max_cost`:

python
client.get(url=url, params={"render": True, "max_cost": 10})


If the request would exceed 10 credits, ScraperAPI returns 403 instead of spending them.

## Batches: the SDK has no async mode

`client.get()` blocks on every call, for up to 70 seconds per URL, and the SDK ships with no async or batch support at all. Past roughly 20 URLs, or for any single URL that regularly times out, switch to the Async Jobs API using plain `requests`:

python
payload = {"apiKey": API_KEY, "url": url, "apiParams": {"render": True}}
job = requests.post("https://async.scraperapi.com/jobs", json=payload, timeout=10).json()
# {"id": "...", "status": "running", "statusUrl": "..."}


Poll the returned `statusUrl` until `status` reads `finished`, then take the body from `data["response"]["body"]`. A `failed` status means the job is dead and should be raised, not retried blindly. If you run your own server, add `"callback": {"type": "webhook", "url": "https://yourapp.com/cb"}` to the payload and drop polling entirely.

For volume, `https://async.scraperapi.com/batchjobs` accepts a `urls` array of up to 50,000 entries in a single call.

Threads still work for small jobs. The older Requests-plus-BeautifulSoup pattern runs a `ThreadPoolExecutor` with five workers and retries each URL three times, which is fine for a few dozen pages. Above that you are rebuilding queueing, backoff and a concurrency ceiling that the jobs endpoint already handles, and exceeding your plan's concurrent-thread limit just returns 429.

## Structured endpoints: skip the parser

If your target is one of the supported sites, the structured endpoints return finished JSON instead of HTML you have to parse, at 1 to 5 credits rather than 10 for a rendered page.

Supported verticals: `google/search`, `google/news`, `google/jobs`, `google/shopping`, `google/maps`, `amazon/product`, `amazon/search`, `amazon/offers`, `walmart/product`, `walmart/search`, `walmart/category`, `ebay/product`, `ebay/search`, `redfin/listing`, `redfin/search`.

python
def structured_get(vertical, params):
    r = requests.get(
        f"https://api.scraperapi.com/structured/{vertical}",
        params={"api_key": API_KEY, **params},
        timeout=70,
    )
    r.raise_for_status()
    return r.json()

structured_get("google/search", {"query": "best python web scrapers"})
structured_get("amazon/product", {"asin": "B09V3KXJPB"})


On the raw endpoint, `autoparse=True` does something similar for supported sites and hands back fields like title, price and reviews as JSON. For anything feeding a language model, `output_format="markdown"` returns clean Markdown instead of tag soup.

If your scraper lives inside an agent, `pip install -U langchain-scraperapi` exposes three tools that read `SCRAPERAPI_API_KEY` from the environment: `ScraperAPITool` for arbitrary pages in HTML, text or Markdown, `ScraperAPIGoogleSearchTool` for SERP data as JSON or CSV, and `ScraperAPIAmazonSearchTool` for Amazon product search. The tool parameters mirror the SDK's, so `render`, `country_code`, `premium` and `device_type` carry over.

## Five things that break your first script

**1. Missing `verify=False` in proxy mode.** Covered above, and it is the most common first failure when porting working code over.

**2. Scrapy's default settings fight the API.** Add these to `settings.py`:

python
ROBOTSTXT_OBEY = False
RETRY_TIMES = 3
CONCURRENT_REQUESTS = 10   # your plan's thread count


Scrapy fetches `robots.txt` before your first request, which interferes with endpoint mode. Remove `DOWNLOAD_DELAY` and `RANDOMIZE_DOWNLOAD_DELAY` if they are set; they throttle concurrency that ScraperAPI is already managing for you.

**3. The 2MB per-request limit.** Images, PDFs and other files scrape like any other URL, but anything over 2MB comes back truncated. Use `binary_target=True` for binary content and expect to skip the large files.

**4. Sessions expire 15 minutes after last use.** A `session_number` keeps you on the same proxy IP across requests, which is what makes pagination and logged-in flows work. Any integer is a valid ID, but a slow crawl between pages loses the session silently.

**5. Timeouts and status codes.** Current documentation describes the sync SDK blocking for up to 70 seconds per URL, and its examples pass `timeout=70`. The 2021 guides said 60 seconds, when the API retried internally for a full minute. Use 70.

| Code | Meaning | What to do |
| --- | --- | --- |
| 200 | Success | Parse it |
| 401 | Invalid API key | Check `SCRAPERAPI_API_KEY` |
| 403 | Blocked, or out of credits | Escalate proxy tier, check balance |
| 404 | Target page missing | Skip the URL |
| 429 | Concurrency limit reached | Back off, or move to async |
| 500 / 503 | Transient failure | Retry with exponential backoff |

500 responses are not billed, so a retry loop costs nothing but time. If you write your own rather than using the SDK's built-in retries, three attempts per URL is the documented baseline.

## Which setup fits your project

- One page, result needed immediately: `client.get()` from the SDK.
- Dozens to thousands of URLs: Async Jobs API with `requests`, webhook if you have a server.
- Amazon, Google, Walmart, eBay or Redfin: structured endpoint, 1–5 credits, no parser to maintain.
- React, Vue or Angular front end: `render=True`, plus `wait_for_selector` when content loads lazily.
- Standard proxies get blocked: `premium=True`, then `ultra_premium=True` alone.
- Multi-step flow or login state: `session_number`, finish inside 15 minutes.

Before you pay, work out two numbers. First, how many of your pages genuinely need rendering, because that ratio turns 100,000 credits into either 100,000 pages or 10,000. Second, your peak concurrency, since exceeding the plan limit returns 429 rather than queueing the request.

Get those two right and the tier choice is arithmetic. 👉 [Pick your tier and start with the free credits](https://www.scraperapi.com/?fp_ref=coupons)
