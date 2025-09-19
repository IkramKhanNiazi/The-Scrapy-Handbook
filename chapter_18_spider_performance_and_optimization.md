# Chapter 18: Spider Performance and Optimization

Think about the first time you feel the pressure of a "Big Data" project. You've been tasked with scraping prices for five million products every Sunday night so a retail company can update their prices on Monday morning.

You run your spider on Sunday at 8:00 PM. By Monday morning at 6:00 AM, you check the progress and realize the spider has finished exactly... forty thousand products. At that rate, it would take over a hundred days to finish a "weekly" crawl. You might feel a sting of embarrassment; you promised the client that Scrapy was the fastest framework in the world, and now you're delivering less than 1% of the data on time.

You might spend the next forty-eight hours in a panic, trying to run ten copies of your spider at once on different laptops or increasing the speed by removing all delays. The latter only results in your home IP address being banned within five minutes. You feel as if you're trying to run an Olympic sprint, but your shoes are made of lead and there's a wall in front of every hurdle.

You're working against the limits of your computer and the patience of the websites you're visiting.

Then, you look deep into the `settings.py` file and start to understand how Scrapy actually works. You learn about "Concurrency," "AutoThrottle," and "Request Prioritization." You'll realize that you don't need ten laptops; you just need to tell Scrapy how to use the power it already has. Once you tune your settings properly, that hundred-day crawl can turn into an eight-hour crawl. You'll deliver the data, save the contract, and finally understand that in scraping, "fast" isn't just about speed it's about balance.

In this chapter, you're going to learn how to find that balance.

---

## Introduction

Scrapy is built for speed. Under the hood, it uses an asynchronous engine that can handle thousands of requests simultaneously. But by default, Scrapy is "polite." It runs at a careful, slow speed so it doesn't accidentally crash a website or get you banned.

In this chapter, we're going to learn how to take off the training wheels. We'll explore the settings that control how many pages your spider visits at once, how to automatically adjust your speed based on the website's health, and how to optimize your code to handle millions of items without breaking a sweat.

## Understanding Scrapy Performance: The Pipeline

Performance in Scrapy is like water flowing through a pipe. If you have a massive pump (your computer) but a tiny pipe (your internet speed or the website's response time), you won't get much water. If you have a huge pipe but a weak pump, you still won't get much water.

Our goal is to find the "Sweet Spot" where we are using as much of our resources as possible without "bursting the pipe" (crashing the site) or "clogging the drain" (running out of RAM).

## Concurrent Requests: The "Grocery Lane" Analogy

Imagine a grocery store with one checkout lane. If fifty people have one item each, they have to wait for everyone in front of them. It's slow.

Now imagine the store opens sixteen checkout lanes. Even if those lanes are exactly the same speed as the first one, the store is now sixteen times faster at checking people out.

This is **Concurrency**.

In `settings.py`, you can control this with:
```python
CONCURRENT_REQUESTS = 16 # Overall total
CONCURRENT_REQUESTS_PER_DOMAIN = 8 # Total for a single website
```
Increasing these numbers will make your message faster, but be careful! If you set this too high, you are essentially launching a "Denial of Service" attack on the website.

### The Reactor: Asyncio vs. Twisted

Scrapy uses an event loop called a "Reactor." For years, it used the Twisted reactor. 
But since Scrapy 2.9, the default is now the **Asyncio** reactor.

**Why does this matter?**
If you want to use modern Python libraries like `playwright`, `httpx`, or any `async/await` syntax, you **must** use the Asyncio reactor.

```python
# settings.py
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
```

## OS-Level Limits: The Invisible Wall

You set `CONCURRENT_REQUESTS = 10000`, but your spider is still slow?
You probably hit your Operating System's "Open Files Limit."

In Linux/Mac, every network connection is a "file." The default limit is often 1024.
**To fix this (on Linux):**
```bash
ulimit -n 65536
```
This single command can unlock 10x performance for massive crawls.

## Download Delays: Being a Polite Guest

If you send too many requests too fast, most modern websites will notice. They'll see a single IP address requesting a new page every 0.1 seconds and simply block you.

The `DOWNLOAD_DELAY` setting adds a "pause" between requests.
```python
DOWNLOAD_DELAY = 2 # Wait 2 seconds between every request
```
Even a small delay of 0.5 or 1 second can be the difference between a successful crawl and a permanent ban.

> [!IMPORTANT]
> **Scrapy Doc Gap: Concurrency vs. Delay**
> This is a point that confuses even intermediate scrapers. If you set `CONCURRENT_REQUESTS = 16` and `DOWNLOAD_DELAY = 1.0`, you might think you are sending one request every second. 
> 
> In reality, you are sending one request every second **per worker thread**. With 16 workers, you could still be hitting the server with 16 requests at the exact same moment. If you truly need to limit your speed to exactly 1 request per second, you must set your concurrency to `1`. Always calculate your total "Hits Per Second" by dividing concurrency by delay.

## AutoThrottle: The "Self-Driving" Speedometer

This is my favorite Scrapy feature. Instead of you guessing the perfect delay, **AutoThrottle** watches the website. If the website is responding quickly, AutoThrottle speeds up. If the website starts to slow down (showing that its server is struggling), AutoThrottle automatically adds more delay to be respectful.

Enable it in your `settings.py`:
```python
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 5.0 # Wait 5 seconds for the very first request
AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0 # Try to keep only 1 request in the air at a time per domain
```

## Caching Responses: The "Never Do Twice" Rule

During development, you'll often run your spider ten times to fix a single bug. This is a waste of time and bandwidth. Scrapy's **HTTP Cache** saves every page you visit to your hard drive. If you visit that same page again, Scrapy just reads the file instead of going to the internet.

```python
HTTPCACHE_ENABLED = True
HTTPCACHE_EXPIRATION_SECS = 0 # Never expire
HTTPCACHE_DIR = 'httpcache'
```
**Warning:** Disable this before you run your final production crawl, or you'll never see the most recent data!

## Debugging Running Spiders (Telnet)

What if your spider is running on a remote server and it suddenly "hangs"? It's not crashing, but it's not moving.

You can "remote into" the spider's brain using **Telnet**.

1. Scrapy listens on port 6023 by default.
2. Run `telnet localhost 6023`.
3. You are now inside a Python shell running *inside* your spider!

```python
>>> est() # Print scheduler statistics
>>> engine.scraper.spiders # See active spiders
>>> import objgraph
>>> objgraph.show_most_common_types() # Find memory leaks!
```

## Profiling Code

If your specific `parse` function is slow (maybe you have a complex regex), you can find the bottleneck.

```bash
# Writes a profile stats file
scrapy crawl my_spider -s PROFILE_FILE=profile.cprof
```

View it with `snakeviz` (visualizer):
```bash
pip install snakeviz
snakeviz profile.cprof
```

## Memory Usage and Leaks

If your spider runs for three days and uses 8GB of RAM, you probably have a "Memory Leak." This usually happens when you are saving too much data inside your spider's memory rather than yielding it to the pipeline.
*   **Don't store huge lists in your spider.**
*   **Yield items as soon as you find them.**

## Chapter Summary

**What we covered:**
- Performance is a balance between speed, resources, and ethics.
- **Concurrency** allows Scrapy to handle many requests at the same time.
- **Download Delays** keep you from getting banned by being a polite guest.
- **AutoThrottle** is a professional tool that automatically adjusts your crawl speed based on the server's health.
- **HTTP Cache** saves time and bandwidth during development.
- Always yield data early to keep memory usage low.

**Key code:**
```python
# settings.py
CONCURRENT_REQUESTS = 32
DOWNLOAD_DELAY = 0.5
AUTOTHROTTLE_ENABLED = True
```

**Previous chapter:**
[Chapter 17: Error Handling and Retries](./chapter_17_error_handling_and_retries.md)

**Next chapter:**
You've mastered the art of the spider. This marks the end of **Part III: Handling Complex Scenarios**. You are now a senior-level scraper. 

In the next part, **Data Storage and Databases**, we're going to learn how to move beyond simple files and into the world of professional data persistence. We'll start with an **Introduction to Databases** and why you need them.

---

**End of Chapter 18**
