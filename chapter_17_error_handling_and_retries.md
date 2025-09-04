# Chapter 17: Error Handling and Retries

Think about the first time you run a production spider overnight. You've spent all week building a scraper that's supposed to collect five hundred thousand product reviews. You hit "Run" at 10:00 PM, go to bed, and dream of the clean, massive dataset you'll have in the morning.

When you wake up at 7:00 AM, your computer is silent. The terminal is empty. No data has been saved.

You look at the logs and might feel a wave of nausea. At 10:15 PM just fifteen minutes after you'd gone to bed the website's server had a temporary "blip." It returned a `502 Bad Gateway` error for a single request. Your spider, not knowing what to do with that error, might have simply panicked and crashed.

Because you hadn't built in any "resilience," a single five-second server stutter ruined nine hours of potential scraping time. You've wasted an entire night and have nothing to show for it but a long list of error logs.

You might feel like an amateur. You'll realize that the internet is a chaotic, unpredictable place. Servers go down, Wi-Fi flickers, and websites change their structure without warning. If your spider is built like a delicate glass tower, it will shatter at the first sign of trouble. If you want to be a professional, you have to build spiders that are like tanks able to keep moving even when they take a hit.

In this chapter, you're going to learn how to build those tanks.

---

## Introduction

As a scraper, you have to expect things to go wrong. Errors aren't just a possibility; they are a guarantee. A professional spider isn't one that never encounters errors; it's one that knows exactly what to do when they happen.

In this chapter, we're going to explore the most common types of scraping errors. We'll learn how to use **Defensive Programming** to prevent crashes, how to use **Try-Except** blocks for critical logic, and how to master **Retry Middleware** so that a temporary server glitch doesn't end your crawl.

## Common Scraping Errors

1.  **Network Errors:** Your internet goes out, or the website's server is too slow to respond (**Timeout**).
2.  **HTTP Errors:** The server is up, but it sends back a "Bad" code (like `404 Not Found`, `403 Forbidden`, or `500 Server Error`).
3.  **Parsing Errors:** The website changed its HTML structure, and your CSS selector is now trying to find something that doesn't exist.
4.  **None Value Errors:** You expect a string, but the selector found nothing and returned `None`.

## Defensive Programming: The "None" Problem

The most common cause of a spider crash is a `AttributeError`. It happens when you do this:
```python
# Dangerous!
text = response.css('h1::text').get().strip()
```
If the `h1` is missing, `.get()` returns `None`. You cannot call `.strip()` on `None`.

**The Solution:** Always check before you act.
```python
# Professional
title = response.css('h1::text').get()
if title:
    clean_title = title.strip()
else:
    clean_title = ""
```

## Try-Except Blocks

Sometimes, you have a complex calculation or a data transformation that might fail. Wrap those in a `try...except` block to ensure the spider keeps moving.

```python
try:
    # Converting a messy price string to a float
    price = float(item['price'].replace('$', ''))
except (ValueError, TypeError, KeyError):
    # If it fails, log the error and provide a default
    self.logger.error(f"Failed to parse price for {item['url']}")
    price = 0.0
```

## Retry Middleware: The Second Chance

Scrapy has a built-in "Retry Middleware" that handles temporary server glitches for you. By default, if a request fails with a `500`, `502`, `503`, or `504` error, Scrapy will automatically put that URL back in the "To-Do List" and try again.

### Customizing Retries
You can control the behavior in your `settings.py`:
```python
RETRY_ENABLED = True
RETRY_TIMES = 3  # Try 3 times before giving up
RETRY_HTTP_CODES = [500, 502, 503, 504, 408]
```

> [!WARNING]
> **Scrapy Doc Gap: Troubleshooting "Silent" Failures**
> Sometimes a spider will stop bringing back data but won't show any errors in the logs. This often happens because a website is using "Soft Rate Limiting" returning codes like `403` (Forbidden) or `429` (Too Many Requests). 
> 
> By default, Scrapy does *not* retry these codes because it assumes you aren't allowed there. If you know you are just being rate-limited, you can add `403` or `429` to your `RETRY_HTTP_CODES` list. This gives your spider a chance to "wait and try again" rather than just ignoring those pages forever.

## HTTP Error Handling

By default, Scrapy ignores "Error" codes like `404` or `403`. It won't even call your `parse` method for those pages!

If you *want* to handle these errors (for example, to log which pages are missing), you can use the `errback` argument in your request:

```python
from scrapy.spidermiddlewares.httperror import HttpError
from twisted.internet.error import DNSLookupError
from twisted.internet.error import TimeoutError

def start_requests(self):
    yield scrapy.Request(url, callback=self.parse, errback=self.errback_httpbin)

def errback_httpbin(self, failure):
    # check if the error is a HttpError (404, 500, etc)
    if failure.check(HttpError):
        response = failure.value.response
        self.logger.error(f'HttpError on {response.url}')

    # check if the error is a DNSLookupError
    elif failure.check(DNSLookupError):
        self.logger.error('DNSLookupError - Address not found')

    # check if the error is a TimeoutError
    elif failure.check(TimeoutError):
        self.logger.error('TimeoutError - Server too slow')
```

**Why this matters:**
This gives you granular control. If it's a `Timeout`, maybe you retry. If it's a `404`, maybe you delete the item from your database. If it's a `DNS` error, maybe your proxy is broken.

## Logging Errors Properly

Professional scrapers don't just "print" errors. They use the **Logging System**.
*   `self.logger.info()`: General progress.
*   `self.logger.warning()`: Something unusual happened, but we are still moving.
*   `self.logger.error()`: Something important failed (like a missing price).
*   `self.logger.critical()`: The entire spider is about to crash.

You can save these logs to a file so you can review them after the crawl:
`scrapy crawl my_spider --logfile=logs.txt`

### External Monitoring (Sentry)

Logs are great, but you don't read them until it's too late. Professional teams screen external monitoring tools like **Sentry** to get real-time alerts when a spider crashes.

**Integration:**
```bash
pip install sentry-sdk
```

```python
# settings.py
import sentry_sdk
sentry_sdk.init("https://your-sentry-dsn")
EXTENSIONS = {
    "scrapy.extensions.logstats.LogStats": 100,
}
```

## Handling Signals

Want to know *exactly* when an error happens? Scrapy emits "Signals."

```python
from scrapy import signals

class MySpider(scrapy.Spider):
    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        spider = super().from_crawler(crawler, *args, **kwargs)
        crawler.signals.connect(spider.spider_error, signal=signals.spider_error)
        return spider

    def spider_error(self, failure, response, spider):
        # Send an email or Slack message here!
        self.logger.critical(f"Spider crashed on {response.url}: {failure}")
```

## Stopping the Spider (`CloseSpider`)

Sometimes, if you get too many errors, you *should* stop. Continuing might ban your IP forever.

```python
from scrapy.exceptions import CloseSpider

def parse(self, response):
    if "Account Suspended" in response.text:
       raise CloseSpider('account_banned')
```

You can also configure automatic stops in settings:
```python
CLOSESPIDER_ERRORCOUNT = 10  # Stop after 10 errors
CLOSESPIDER_PAGECOUNT = 1000 # Stop after 1000 pages
CLOSESPIDER_TIMEOUT = 3600   # Stop after 1 hour (3600s)
```

## Graceful Degradation

If a specific piece of data is missing (like the "Author's Twitter Handle"), don't let it stop you from saving the "Author's Name." A professional scraper should collect as much data as possible, even if the result isn't perfect. It's better to have a dataset that is 95% complete than one that is 0% complete because it crashed.

## Chapter Summary

**What we covered:**
- Professionals expect and plan for errors.
- **Defensive Programming** prevents `None` errors from crashing your spider.
- **Try-Except** blocks isolate risky logic so the rest of the crawl can continue.
- **Retry Middleware** automatically handles temporary server glitches.
- The `errback` method allows you to handle network failures gracefully.
- Proper **Logging** ensures you can troubleshoot your spider without being present.

**Key code:**
```python
# The "Tank" Pattern
def parse(self, response):
    item = {}
    try:
        item['name'] = response.css('h1').get() or "Unknown"
        # ... more extraction ...
    except Exception as e:
        self.logger.error(f"Fatal parsing error: {e}")
    yield item
```

**Previous chapter:**
[Chapter 16: Sitemaps and Efficient Crawling](./chapter_16_sitemaps_and_efficient_crawling.md)

**Next chapter:**
Your spiders are now resilient. But are they fast? In the next chapter, we're going to explore **Spider Performance and Optimization** learning how to fine-tune your settings to scrape thousands of items while staying under the radar.

---

**End of Chapter 17**
