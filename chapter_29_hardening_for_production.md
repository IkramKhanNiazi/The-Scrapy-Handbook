# Chapter 29: Hardening for Production

Think about the first time you leave a spider running on a remote server and go to dinner. You're feeling confident. You've tested it on your laptop, and it worked perfectly. You've even added a basic `try...except` block. You figure nothing can go wrong while you're eating pasta.

When you check your phone an hour later, you might have forty-three missed "Error Alert" emails.

You rush back to your computer and find a disaster. The website you were scraping has detected your server's IP address and started sending back a "Bait" HTML page. Instead of product data, it's sending back ten thousand lines of gibberish. Your spider, not knowing it is being tricked, is dutifully extracting that gibberish and saving it over your clean database records. Even worse, because you hadn't set a memory limit, the spider has tried to process a 500MB "Zip bomb" the website sent as a response, and your entire server has frozen.

You might feel like an amateur who has brought a toy knife to a sword fight. You'll realize that the "Live" internet is a hostile environment. It's full of traps, anti-bot shields, and unexpected server behavior. If you want your code to survive out there while you're sleeping or eating dinner, you can't just write "working" code. You have to write "Hardened" code.

In this chapter, you're going to learn how to armor your spiders.

---

## Introduction

So far, we've focused on making things work. But "it works on my machine" is the most dangerous phrase in programming. When you move your code to a production server, everything changes: the IP address is different, the resources are Limited, and there is no human present to hit "Ctrl+C" when things go sideways.

In this chapter, we're going to learn how to **Harden** your project for production. We'll cover security best practices, how to build "Validation Guards" to catch bad data before it hits your database, and how to configure Scrapy to protect itself (and your server) from the many traps of the modern web.

## Security Best Practices: No More Hardcoding

The most common "Production Fail" is security. On your laptop, you might have put your database password directly in `pipelines.py`. On a server, that's a massive risk.

**Rule #1: Use Environment Variables.**
Instead of: `password = "secret123"`, use:
```python
import os
password = os.environ.get('DB_PASSWORD')
```
This ensures your "Secrets" never end up in your Git repository or on a shared server screen.

### Validating Environment Variables
It's risky to just "hope" the environment variables exist. If `DB_PASSWORD` is missing, your spider might run for hours before crashing at the very end.

**Pro Pattern:** Use `pydantic-settings`.
```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DB_PASSWORD: str
    AWS_ACCESS_KEY: str
    
    class Config:
        env_file = ".env"

# If a variable is missing, the spider crashes IMMEDIATELY 
# (which is good!) before doing any work.
config = Settings()
```

## Handling "Dirty" Input at Scale

In production, a website might change its structure mid-crawl. You need guards to catch this.

```python
# The "Contract" Pattern in a Spider
price = response.css('.price::text').get()
if not price:
    self.logger.error(f"PRICE DISAPPEARED: {response.url}")
    # Don't yield a broken item!
    return
```

## Defensive Settings: Protecting the Server

Your `settings.py` is your primary shield. In production, you MUST set these:

1.  **`DOWNLOAD_TIMEOUT = 10`**: Don't let a slow request hang forever.
2.  **`MEMUSAGE_LIMIT_MB = 1024`**: If your spider uses more than 1GB of RAM, Scrapy will kill it automatically before it crashes your whole server.
3.  **`CLOSESPIDER_ERRORCOUNT = 10`**: If your spider hits 10 errors in a row, something is wrong. Shut it down and send an alert rather than continuing to fail.

### The "Kill Switch" Middleware
Sometimes standard limits aren't enough. You can write a middleware to kill the spider if it hits too many `403 Forbidden` responses (meaning you are banned).

```python
class BanDetectionMiddleware:
    def __init__(self):
        self.ban_count = 0

    def process_response(self, request, response, spider):
        if response.status == 403:
            self.ban_count += 1
            if self.ban_count > 20:
                spider.logger.critical("TOO MANY 403s! STOPPING SPIDER.")
                spider.crawler.engine.close_spider(spider, 'banned')
        return response
```

## Dealing with IP Bans: Pro Strategies

If you are running from a VPS like AWS, your IP address is already on many "Scraper Blacklists."
*   **The "Proxy Rotation" Middleware:** Use a service that gives you a new IP for every single request.
*   **User-Agent Rotation:** Don't just use one User-Agent. Use a library like `scrapy-user-agents` to look like a different person every time.

## Automated Quality Checks

Before you save an item to your database, ask: "Does this look right?"
*   Is the price a number?
*   Is the URL a real link?
*   Is the title longer than three characters?
A "Sanity Check" pipeline can save you from a morning of cleaning up junk data.

### The "Drop Item" Pattern
Don't be afraid to drop data. It is better to have *no data* than *corrupt data*.

```python
from scrapy.exceptions import DropItem

def process_item(self, item, spider):
    if not item.get('price'):
        raise DropItem(f"Missing price in {item['url']}")
        
    if "out of stock" in item['title'].lower():
        raise DropItem("Item is out of stock")
        
    return item
```
*Note: We will dive deep into automated validation with **Spidermon** in Chapter 34.*

## Developing for "Unattended" Runs

A hardened spider needs to be self-sufficient. This means:
1.  **Detailed Logging:** If it fails at 3:00 AM, the logs should tell you *why* without you having to rerun it.
2.  **Graceful Restarts:** Use `JOBDIR` settings so that if the server reboots, the spider picks up where it left off.

> [!WARNING]
> **Scrapy Doc Gap: The JOBDIR Ghost**
> While `JOBDIR` is a lifesaver for long crawls, it can be a nightmare for deployments. If you update your spider's code (e.g., you fix a broken selector) and restart the spider using the same `JOBDIR`, Scrapy might still use the "pickled" state from the old run. 
> 
> This means your new code might not actually run until the old jobs are finished. Always clear your `JOBDIR` folder when deploying a major code change to ensure your spiders are running the latest version!

## Chapter Summary

**What we covered:**
- Production is a hostile environment that requires "Armored" code.
- Always use environment variables for secrets like passwords and API keys.
- Use `MEMUSAGE` and `TIMEOUT` settings to protect your server's health.
- Validation guards in your pipeline prevent "Gibberish" from ruining your database.
- Designing for "unattended" runs means prioritizing logs and persistence.
- The goal is for your spider to fail safely rather than fail in a way that breaks everything.

**Key code:**
```python
# settings.py
CLOSESPIDER_TIMEOUT = 3600 # Stop after 1 hour
CLOSESPIDER_PAGECOUNT = 1000 # Stop after 1000 pages
```

**Previous chapter:**
[Chapter 28: Speed vs. Ethics](./chapter_28_speed_vs_ethics.md)

**Next chapter:**
Your code is hardened. It's secure. It's ready. In the next chapter, we're going to learn about **VPS Setup for Scrapy** leasing your first remote server and setting up a professional Linux environment from scratch.

---

**End of Chapter 29**
