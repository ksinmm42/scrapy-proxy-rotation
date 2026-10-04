# scrapy rotating proxies: how to wire up rotation middleware without burning your proxy budget

A spider that runs clean for 400 requests and then starts returning 403 on every single URL is not a parsing bug. Nothing in `parse()` changed. What changed is that the exit IP collecting those requests got scored, and Scrapy kept using it anyway, because Scrapy's built-in proxy handling has no opinion about what happens after a proxy goes bad.

That's the whole problem with "rotating proxies in Scrapy." Rotation isn't a switch you flip. It's three separate jobs: assign an IP per request, decide when a response means "this IP is burned," and swap it without losing the request. This guide covers all three, plus the part most tutorials skip — what the IPs actually cost and which billing model fits which crawling pattern.

## Why Scrapy's default proxy middleware quietly fails you

`HttpProxyMiddleware` reads `request.meta["proxy"]` and applies it. If you don't set it, nothing happens. If you set one proxy globally, every request uses that proxy until it dies, and then every request fails together.

Scrapy's retry middleware won't save you either. In recent versions the default `RETRY_HTTP_CODES` is `[500, 502, 503, 504, 522, 524, 408, 429]`, with `RETRY_TIMES = 2`. A 403 is deliberately excluded. That's a reasonable default — per RFC 9110, a 403 means the server understood the request and refused it, which could be your headers, your session, or your IP. But when the cause *is* the IP, retrying the same request through the same proxy just burns time.

So you need a layer that (a) picks a different exit IP per request, (b) recognises ban signatures, and (c) retries through a fresh IP rather than the same one.

## Two architectures, and they are not interchangeable

Before touching settings, decide which shape your rotation takes. Almost every mistake downstream traces back to confusing these two.

**A pool of individual proxies.** You have N endpoints, each with its own IP, and you rotate among them. You own the bookkeeping: which are alive, which are rate-limited, when to bring one back. This is what `scrapy-rotating-proxies` was built for.

**One rotating gateway.** A single host:port that hands you a different exit IP per connection, or per session depending on the credentials you send. There's no list to manage — the provider does the rotation. Your middleware just needs to attach the same URL to every request.

The gateway model is usually less code and less to go wrong. The pool model gives you finer control over which specific IP handles which request, which matters when you need a stable IP per account or per city.

## Setting up a proxy pool with scrapy-rotating-proxies

Install it:


pip install scrapy-rotating-proxies


Then wire it into `settings.py`. The ordering matters — if you leave Scrapy's own middleware enabled, the two fight over `request.meta["proxy"]`:

python
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': None,
    'rotating_proxies.middlewares.RotatingProxyMiddleware': 610,
    'rotating_proxies.middlewares.BanDetectionMiddleware': 620,
}

ROTATING_PROXY_LIST = [
    'http://user:pass@host-a:8000',
    'http://user:pass@host-b:8000',
    'socks5://user:pass@host-c:1080',
]


Or keep the list in a file, one per line, and point at it:

python
ROTATING_PROXY_LIST_PATH = '/etc/spider/proxies.txt'


`ROTATING_PROXY_LIST_PATH` wins if both are set.

Two behaviours worth knowing before you're surprised by them:

- **Requests with `meta["proxy"]` set are left alone.** Set `request.meta["proxy"] = None` to bypass proxying for a specific request.
- **Concurrency settings become per-proxy.** `CONCURRENT_REQUESTS_PER_DOMAIN` stops meaning "per domain" and starts meaning "per proxy." If you set it to 2, each proxy handles at most 2 concurrent connections regardless of how many domains you're hitting. People set it low, then wonder why throughput collapsed.

### Ban detection is the part that decides whether this works

The stock heuristic is blunt: a non-200 status, an empty body, or an exception means the proxy is dead. That's wrong often enough to matter. A 404 from the target isn't a dead proxy. A 403 from a Cloudflare challenge might be. A 407 is definitely about the proxy — it means the proxy rejected your authentication. Mixing those signals into one bucket is how you retire healthy IPs for no reason.

Subclass the policy and be explicit:

python
# myproject/policy.py
from rotating_proxies.policy import BanDetectionPolicy

class StrictBanPolicy(BanDetectionPolicy):
    def response_is_ban(self, request, response):
        ban = super().response_is_ban(request, response)
        # treat a 200 with a challenge page as a ban too
        ban = ban or response.status in (403, 407, 429, 503)
        return ban

    def exception_is_ban(self, request, exception):
        return True  # timeouts and connection resets -> rotate


python
ROTATING_PROXY_BAN_POLICY = "myproject.policy.StrictBanPolicy"


Dead proxies aren't written off permanently. The middleware rechecks them on a randomised exponential backoff — the first retry comes quickly, later ones stretch out — so a temporarily throttled IP can come back into rotation. Tune the starting delay with `ROTATING_PROXY_BACKOFF_BASE` if the default (random, 0–5 minutes) is too slow or too aggressive for your targets.

### Confirm it's actually rotating

Don't trust the logs. Hit an echo endpoint and log what comes back:

python
class IpCheckSpider(scrapy.Spider):
    name = "ipcheck"
    start_urls = ["https://httpbin.org/ip"]

    def parse(self, response):
        self.logger.info("exit ip: %s", response.json().get("origin"))


Run it ten times with `-s LOG_LEVEL=INFO` and count distinct addresses. If you see one IP repeated, your middleware isn't in the chain — check the numbering in `DOWNLOADER_MIDDLEWARES` and the `None` you set on Scrapy's default.

## One gateway instead of a list

If your provider gives you a rotating endpoint, the pool middleware is more machinery than you need. You can still use it — a list with one entry works — but understand the side effect: with a single URL, the first ban marks your only "proxy" as dead, and the middleware's health tracking sits in backoff instead of rotating, because there's nothing to rotate to.

The cleaner shape is a small custom middleware that stamps the gateway onto every request:

python
class GatewayProxyMiddleware:
    def __init__(self, gateway):
        self.gateway = gateway

    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler.settings.get("PROXY_GATEWAY"))

    def process_request(self, request, spider):
        if "proxy" not in request.meta:
            request.meta["proxy"] = self.gateway


python
PROXY_GATEWAY = "http://USERNAME:PASSWORD@gateway-host:PORT"


Leave the host, port, and credentials as values you copy out of your account dashboard — don't hardcode them in a file you commit. Pull them from an environment variable and keep them out of the repo.

Then handle bans in `process_response`, reissuing through the same gateway. The provider picks the new exit IP; you just need to not give up.

With 9Proxy's bandwidth-based residential product, the endpoint behaves differently depending on what you send in the username. Rotating mode hands you a fresh IP automatically, sticky mode holds an IP for the session length you specify, and you can append country, state, city, ZIP, or ISP parameters. That covers the case where your spider needs a German residential IP for one request and a Brazilian one for the next without restarting anything.

## Which billing model fits a Scrapy job

This is where most people overpay, and it's worth working out before you buy anything.

**Per-IP with unlimited traffic** makes sense when your traffic volume is unpredictable or large, and when you need sessions that live for hours. The tradeoff: those IPs last from a few hours up to roughly a day, and on 9Proxy the IP-based product authenticates through the desktop app doing local port forwarding. If your spider runs in a Docker container on a cloud box, that's friction you'll feel immediately — and it's the single biggest reason people pick the wrong plan.

**Per-GB** is the natural fit for high-rotation crawling where each page is small. Traffic rotates through the pool with no fixed IP lifetime to babysit, authentication is username/password or an IP whitelist, and it works straight from the dashboard with no app involved. For a Scrapy spider living on a server, that's the configuration that doesn't fight you.

Do the arithmetic on your own numbers. Say your spider pulls 200,000 product pages a month at roughly 60 KB each after compression — about 12 GB. On the 50 GB + 5 GB pack at $105, that balance lasts around four and a half months, which lands near $23 a month. If the same spider needed 2,000 sticky sessions held open for hours at a time, the per-IP model with unlimited bandwidth is the one that won't blow up when you accidentally stream a few gigabytes of images through a single IP.

👉 [See the current 9Proxy residential packages and per-unit rates](https://bit.ly/9-Proxy)

## Every 9Proxy plan currently on the pricing page

9Proxy adjusted IP-based and bundle pricing on June 1, 2026; the GB-based tiers were left alone. Unused IPs on IP plans don't expire. GB packages carry 180-day validity unless you're on Enterprise, where traffic never expires. There are no subscriptions — you buy balance and draw down on it.

| Package | What you get | Price | Validity / billing | Get it |
| --- | --- | --- | --- | --- |
| 100 IPs | 100 residential IPs, unlimited bandwidth while active | $24 | One-off, IPs never expire | [Grab the 100 IP package](https://bit.ly/9-Proxy) |
| 500 IPs | 500 residential IPs, unlimited bandwidth | $72 | One-off, IPs never expire | [Compare the 500 IP package](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | 1,500 IPs total, unlimited bandwidth | $126 | One-off, IPs never expire | [Check the 1,000 IP + 500 bonus package](https://bit.ly/9-Proxy) |
| 2,500 IPs | 2,500 residential IPs, unlimited bandwidth | $210 | One-off, IPs never expire | [View the 2,500 IP package](https://bit.ly/9-Proxy) |
| 5,000 IPs | 5,000 residential IPs, unlimited bandwidth | $360 | One-off, IPs never expire | [See the 5,000 IP package](https://bit.ly/9-Proxy) |
| 15,000 IPs | 15,000 residential IPs, unlimited bandwidth | $720 | One-off, IPs never expire | [Open the 15,000 IP package](https://bit.ly/9-Proxy) |
| 25,000 IPs | 25,000 residential IPs, unlimited bandwidth | $863 | One-off, IPs never expire | [View the 25,000 IP package](https://bit.ly/9-Proxy) |
| 50,000 IPs | 50,000 residential IPs, unlimited bandwidth | $1,438 | One-off, IPs never expire | [Check the 50,000 IP package](https://bit.ly/9-Proxy) |
| 100,000 IPs (Business) | 100,000 residential IPs, unlimited bandwidth | $2,300 | One-off, IPs never expire | [Ask about the 100,000 IP package](https://bit.ly/9-Proxy) |
| 200,000 IPs (Business) | 200,000 residential IPs, unlimited bandwidth | $4,140 | One-off, IPs never expire | [Ask about the 200,000 IP package](https://bit.ly/9-Proxy) |
| 500,000 IPs (Business) | 500,000 residential IPs, unlimited bandwidth | $8,625 | One-off, IPs never expire | [Ask about the 500,000 IP package](https://bit.ly/9-Proxy) |
| 5 GB | Bandwidth-based, unlimited endpoints, sticky or rotating | $15 | 180 days | [Get the 5 GB plan](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | 55 GB total, unlimited endpoints | $105 | 180 days | [Get the 50 GB + 5 GB plan](https://bit.ly/9-Proxy) |
| 100 GB | Bandwidth-based, unlimited endpoints | $150 | 180 days | [Get the 100 GB plan](https://bit.ly/9-Proxy) |
| 200 GB | Bandwidth-based, unlimited endpoints | $200 | 180 days | [Get the 200 GB plan](https://bit.ly/9-Proxy) |
| 1,000 GB | Bandwidth-based, unlimited endpoints | $800 | 180 days | [Get the 1,000 GB plan](https://bit.ly/9-Proxy) |
| 2,000 GB | Bandwidth-based, unlimited endpoints | $1,500 | 180 days | [Get the 2,000 GB plan](https://bit.ly/9-Proxy) |
| 3,000 GB (Enterprise) | Team mode, per-member traffic controls, activity logs | $2,160 | Traffic never expires | [Ask about the 3,000 GB Enterprise plan](https://bit.ly/9-Proxy) |
| 6,000 GB (Enterprise) | Team mode, per-member traffic controls, activity logs | $4,200 | Traffic never expires | [Ask about the 6,000 GB Enterprise plan](https://bit.ly/9-Proxy) |
| 10,000 GB (Enterprise) | Team mode, per-member traffic controls, activity logs | $6,800 | Traffic never expires | [Ask about the 10,000 GB Enterprise plan](https://bit.ly/9-Proxy) |
| Bundle: 100 IPs + 5 GB | IPs plus bandwidth in one balance | $30 | IPs never expire, GB on 180-day validity | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Bundle: 1,500 IPs + 50 GB | IPs plus bandwidth in one balance | $180 | IPs never expire, GB on 180-day validity | [Get the 1,500 IP bundle](https://bit.ly/9-Proxy) |
| Bundle: 5,000 IPs + 500 GB | IPs plus bandwidth in one balance | $720 | IPs never expire, GB on 180-day validity | [Get the 5,000 IP bundle](https://bit.ly/9-Proxy) |

👉 [Sign up and pick a plan against your own bandwidth estimate](https://bit.ly/9-Proxy)

## Settings that matter more than the proxy choice

Rotation gets you past the IP layer. It won't save a spider that looks like a robot in every other respect.

- **Set `DOWNLOAD_TIMEOUT` to something sane.** Default is 180 seconds. One dead proxy in your list means one request sitting there for three minutes while the scheduler waits.
- **Enable AutoThrottle.** It adapts delay to observed latency, which beats a fixed `DOWNLOAD_DELAY` once you're routing through exit IPs with wildly different quality.
- **Rotate user agents too.** `scrapy-fake-useragent` at priority 400 is the low-effort version. A residential IP paired with a Python-requests user agent is a contradiction a WAF notices.
- **Keep `RETRY_TIMES` modest.** Three or four attempts per request with a fresh IP per attempt is plenty. Twenty retries on a target that has decided to block you is just noise.
- **Track ban rate per domain, not globally.** A single `ROTATING_PROXY_BAN_CODES` list across ten sites will tell you nothing useful. If one target returns 200-with-a-challenge-page, that needs its own policy.

The honest summary: rotation is a small amount of code and a large amount of picking the right exit IPs. Datacenter ranges get scored at the ASN level no matter how many you rotate through, which is why a scraping stack that keeps failing ends up on residential either way.

## Common questions

**Do I need the middleware at all if I use a rotating gateway?**
Not strictly. If every request goes through the same endpoint and the endpoint rotates for you, a fifteen-line middleware handles it. The pool middleware earns its keep when you're managing many distinct endpoints or need per-proxy rate limiting.

**Why does my spider hang instead of retrying?**
Usually middleware order. If `HttpProxyMiddleware` is still enabled alongside another proxy middleware, one of them is setting `meta["proxy"]` after the other has read it. Set Scrapy's to `None` and verify with a single-IP echo test.

**What's the difference between a 403 and a 407 here?**
A 407 means the proxy itself rejected your credentials — a configuration problem, not a ban, and retrying through a different IP won't fix it. A 403 comes from the target. Treat them as different events.

**Can I use SOCKS5?**
Yes. 9Proxy's network supports both HTTP/HTTPS and SOCKS5, and SOCKS5 is what you want if you're feeding traffic through proxychains, an anti-detect browser, or a Python process that shouldn't be doing protocol conversion per request.

**Does unused balance go bad?**
IPs on the per-IP plans don't expire, and Enterprise GB traffic never expires. Standard GB packages run on a 180-day validity window, which is worth factoring in if your crawling is seasonal rather than constant.

The setup isn't complicated once you separate the three jobs. Assign an IP, detect the ban, swap and retry — and then make sure the IPs you're swapping through are ones the target hasn't already written off.
