# Chapter 37: Proxies and IP Rotation

Think about the first time you try to scrape a major retail site from your laptop. Your spider runs perfectly for the first five hundred pages. You're feeling confident, maybe even a little smug. Then suddenly, every request starts returning a blank page or a "Please verify you're human" message.

You check your code. Nothing has changed. You try again. Same result. You try a different product page and get blocked. A category page, also blocked. You open the site in your browser, and it works perfectly. But your spider is completely useless.

This is frustrating. The website hasn't broken your code. It has banned your **IP address**. Every computer connected to the internet has a unique address, and large websites keep lists of addresses that act like bots. Your laptop's IP is now on that list, possibly forever.

That's when you discover the solution that every professional scraper uses: **Proxies**.

In this chapter, you're going to learn how to become invisible.

---

## Introduction

Your IP address is your digital fingerprint. When you scrape from a single IP, you're essentially walking into a store wearing the same bright orange jacket every day. Eventually, security will notice you.

A **Proxy** is a middleman server. Instead of your computer directly requesting a webpage, you ask the proxy to fetch it for you. The website only sees the proxy's IP, not yours. And if one proxy gets blocked? You simply switch to another.

In this chapter, we're going to explore the different types of proxies, how to set them up in Scrapy, and how to build professional-grade rotation systems that can handle millions of requests without getting blocked.

## Types of Proxies: The Price vs. Quality Tradeoff

Not all proxies are created equal. Here's the hierarchy:

### 1. Datacenter Proxies (Cheapest)
- **What they are:** IP addresses from cloud providers (AWS, Google Cloud, DigitalOcean).
- **Cost:** $1-$5 per 100 IPs.
- **Problem:** Websites know these IP ranges! They are the *first* to be blocked.
- **Use for:** Low-security sites, testing, bulk scraping of easy targets.

### 2. Residential Proxies (Best Value)
- **What they are:** Real IP addresses from ISPs (Comcast, AT&T, Vodafone).
- **Cost:** $5-$15 per GB of traffic.
- **Why they work:** These look like normal home internet users.
- **Use for:** E-commerce, social media, any site with serious anti-bot protection.

### 3. Mobile Proxies (Premium)
- **What they are:** IP addresses from mobile carriers (4G/5G).
- **Cost:** $20-$50 per GB.
- **Why they work:** Mobile IPs are shared by thousands of real users. Blocking them means blocking paying customers.
- **Use for:** The most protected sites (sneaker drops, ticket sales).

### 4. ISP Proxies (Hybrid)
- **What they are:** Datacenter speeds with residential IP registrations.
- **Cost:** $10-$30 per IP per month.
- **Use for:** When you need speed AND legitimacy.

> [!TIP]
> **Scrapy Doc Gap: The Hidden Cost of "Unlimited" Proxies**
> Many proxy providers advertise "unlimited bandwidth." Read the fine print! They often throttle speeds after a certain amount or provide lower-quality IPs. 
> 
> For production scraping, **pay-per-GB residential proxies** are almost always more cost-effective than "unlimited" datacenter pools because you won't waste time on blocked requests.

## Setting Up Proxies in Scrapy

The simplest way to use a proxy is with the `meta` parameter:

```python
yield scrapy.Request(
    url='https://example.com',
    meta={'proxy': 'http://user:pass@proxy.example.com:8080'}
)
```

But hardcoding a single proxy is useless. You need **rotation**.

## Building a Proxy Rotation Middleware

Here's a professional-grade middleware that rotates through a list of proxies:

```python
# middlewares.py
import random

class ProxyRotationMiddleware:
    def __init__(self, proxy_list):
        self.proxies = proxy_list
        self.failed_proxies = set()
    
    @classmethod
    def from_crawler(cls, crawler):
        proxy_list = crawler.settings.getlist('PROXY_LIST')
        return cls(proxy_list)
    
    def process_request(self, request, spider):
        # Filter out failed proxies
        available = [p for p in self.proxies if p not in self.failed_proxies]
        if not available:
            spider.logger.error("All proxies have failed!")
            self.failed_proxies.clear()  # Reset and try again
            available = self.proxies
        
        proxy = random.choice(available)
        request.meta['proxy'] = proxy
        spider.logger.debug(f"Using proxy: {proxy}")
    
    def process_exception(self, request, exception, spider):
        # Mark this proxy as failed
        proxy = request.meta.get('proxy')
        if proxy:
            self.failed_proxies.add(proxy)
            spider.logger.warning(f"Proxy failed: {proxy}")
```

Enable it in `settings.py`:
```python
PROXY_LIST = [
    'http://user:pass@proxy1.example.com:8080',
    'http://user:pass@proxy2.example.com:8080',
    'http://user:pass@proxy3.example.com:8080',
]

DOWNLOADER_MIDDLEWARES = {
    'myproject.middlewares.ProxyRotationMiddleware': 350,
}
```

## Smart Retry with Different Proxies

When a request fails, you want to retry it with a *different* proxy:

```python
def process_response(self, request, response, spider):
    if response.status in [403, 429, 503]:
        # This proxy is detected! Retry with a new one.
        proxy = request.meta.get('proxy')
        if proxy:
            self.failed_proxies.add(proxy)
        
        # Create a new request (will get a fresh proxy)
        return request.replace(dont_filter=True)
    
    return response
```

## Using Proxy Services (The Easy Way)

Managing your own proxy pool is complex. Most professionals use managed services:

| Service | Type | Cost | Best For |
|---------|------|------|----------|
| **Bright Data** | All types | $$$$ | Enterprise, highest quality |
| **Oxylabs** | Residential | $$$ | E-commerce, large scale |
| **ScraperAPI** | Managed | $$ | Beginners, turnkey solution |
| **Smartproxy** | Residential | $$ | Good balance of cost/quality |

These services handle rotation, health checking, and geographic targeting for you.

```python
# ScraperAPI example - they handle everything
PROXY = 'http://api.scraperapi.com?api_key=YOUR_KEY&url='

def start_requests(self):
    target = 'https://example.com/products'
    yield scrapy.Request(PROXY + target)
```

## Geographic Targeting

Some sites show different content based on location. With proxies, you can appear to be anywhere:

```python
# Request a German IP
request.meta['proxy'] = 'http://user:pass@de.proxy.example.com:8080'
```

Most proxy services support country targeting:
```python
# Bright Data format
proxy = 'http://user-country-de:pass@proxy.brightdata.com:22225'
```

## Proxy Pool Health Monitoring

In production, you need to know which proxies are working:

```python
class ProxyHealthMonitor:
    def __init__(self):
        self.stats = {}  # proxy -> {success: 0, fail: 0}
    
    def record_success(self, proxy):
        if proxy not in self.stats:
            self.stats[proxy] = {'success': 0, 'fail': 0}
        self.stats[proxy]['success'] += 1
    
    def record_failure(self, proxy):
        if proxy not in self.stats:
            self.stats[proxy] = {'success': 0, 'fail': 0}
        self.stats[proxy]['fail'] += 1
    
    def get_success_rate(self, proxy):
        s = self.stats.get(proxy, {'success': 0, 'fail': 0})
        total = s['success'] + s['fail']
        return s['success'] / total if total > 0 else 0
    
    def get_best_proxies(self, min_success_rate=0.8):
        return [p for p, s in self.stats.items() 
                if self.get_success_rate(p) >= min_success_rate]
```

## Cost Analysis: When Are Proxies Worth It?

| Scenario | Recommendation |
|----------|---------------|
| Scraping < 1,000 pages/day from easy sites | No proxies needed |
| Scraping 1,000-10,000 pages/day | Free rotating proxies or cheap datacenter |
| Scraping 10,000+ pages/day from protected sites | Paid residential proxies |
| Scraping sneaker sites, tickets, etc. | Mobile proxies (expensive but necessary) |

**The Math:**
- Your time: $50/hour
- Debugging blocked requests: 5 hours = $250
- Monthly residential proxy cost: $100
- **Conclusion:** Proxies pay for themselves in saved debugging time.

> [!WARNING]
> **Scrapy Doc Gap: The "Free Proxy" Trap**
> Free proxy lists from the internet are tempting but dangerous. They are:
> 1. **Slow:** Overloaded by thousands of users.
> 2. **Unreliable:** 90% are dead at any given time.
> 3. **Insecure:** Some log your traffic or inject ads.
> 
> For anything beyond learning, invest in paid proxies. Your data integrity depends on it.

## Chapter Summary

**What we covered:**
- Your IP address is the first thing websites use to identify scrapers.
- Proxies act as middlemen, hiding your real IP from target sites.
- Residential and mobile proxies are harder to detect than datacenter IPs.
- Professional rotation requires middleware that tracks proxy health.
- Managed proxy services handle complexity but cost more.
- Geographic targeting lets you appear to browse from any country.

**Key code:**
```python
request.meta['proxy'] = 'http://user:pass@proxy.example.com:8080'
```

**Previous chapter:**
[Chapter 36: Signals and Events](./chapter_36_signals_and_events.md)

**Next chapter:**
Proxies hide your IP, but modern anti-bot systems look at much more than that. In the next chapter, we're going to dive into **Anti-Bot Bypass Techniques**, learning how to defeat fingerprinting, behavior analysis, and the sophisticated shields that guard today's most valuable websites.

---

**End of Chapter 37**
