# Chapter 25: Scrapy-Redis Setup

Think about the first time you see two different terminal windows working on the exact same project. You've just set up **Scrapy-Redis**, and you might feel nervous. It can feel as if you're trying to perform a high-wire act where one wrong move could send everything crashing down.

In the first window, you run your spider. In the second window, you run the exact same command.

You might watch, holding your breath. Instead of both spiders scraping the same first page, they split up. Spider A grabs the first ten items. Spider B grabs the next ten. They're "talking" to each other through the Redis server in the background, making sure they never overlap. It's like watching a well-rehearsed dance.

But the real magic happens when you deliberately "kill" the first spider. You close the terminal window while it's mid-crawl. Normally, that would mean you'd lost all your progress and would have to start over. But you look at the second window, and it's still moving. Even better, when you restart the first spider, it doesn't start from the beginning. It reaches out to Redis, asks for its place in line, and picks up exactly where it had left off.

You'll feel as if you've finally made your spiders live forever. They're no longer fragile scripts running on your laptop; they're a resilient, distributed system that can survive crashes, restarts, and small network issues without losing a single byte of data.

In this chapter, you're going to learn how to build that resilience.

---

## Introduction

In the previous chapter, we learned the theory of distributed crawling. Now, it's time to build it. We're going to use **Scrapy-Redis**, the most popular and trusted library for making Scrapy distributed.

In this chapter, we'll learn what Redis is and how to install it. We'll then walk through the process of converting a standard spider into a distributed one. We'll learn how to launch multiple spiders at once and how to monitor the "Global To-Do List" to see how much work is left. By the end of this chapter, you'll have a swarm of spiders ready to tackle any dataset.

## What is Redis?

**Redis** stands for REmote DIctionary Server. It is a "database" that lives entirely in your computer's RAM. Because it uses memory instead of a hard drive, it is incredibly fast perfect for handling the millions of "read and write" requests that a fleet of spiders will make as they update their shared queue.

## Step 1: Installing and Testing Redis

If you are on a Mac, use Homebrew:
```bash
brew install redis
brew services start redis
```
On Windows, you can download the installer from GitHub or use WSL (Windows Subsystem for Linux).

To test if it's working, type:
```bash
redis-cli ping
```
If you get back `PONG`, you are ready to go!

## Step 2: Scrapy-Redis Installation

In your virtual environment, install the library:
```bash
pip install scrapy-redis
```

## Step 3: Converting Your Spider

To make a spider distributed, we change what it inherits from.

```python
# Before:
# class MySpider(scrapy.Spider):

# After:
from scrapy_redis.spiders import RedisSpider

class MyDistributedSpider(RedisSpider):
    name = 'myswarm'
    # Notice we don't use 'start_urls' anymore!
    # Instead, we define a Redis key where we will "feed" the URLs
    redis_key = 'myswarm:start_urls'

    def parse(self, response):
        yield {'url': response.url}
    def parse(self, response):
        yield {'url': response.url}
```

### Option B: Keeping Standard Spiders (Preferred)
You don't *have* to change your code. You can use a standard `scrapy.Spider` and just change `settings.py`. This is better because you can run the same spider locally (without Redis) or distributed (with Redis) just by changing config.

```python
# settings.py
# If True, simple spiders behave like RedisSpiders
REDIS_START_URLS_AS_SET = True 
```

## Step 4: Configuration (settings.py)

This is where the real magic happens. We tell Scrapy to stop using its local scheduler and use the Redis one instead.

```python
# settings.py

# 1. Use the Scrapy-Redis scheduler
SCHEDULER = "scrapy_redis.scheduler.Scheduler"

# 2. Use the Scrapy-Redis duplicate filter
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"

# 3. Queue Types (Priority is Key!)
# Default: PriorityQueue (Respects Request.priority)
SCHEDULER_QUEUE_CLASS = "scrapy_redis.queue.PriorityQueue"
# Alternative: FifoQueue (First In First Out - Breadth First)
# Alternative: LifoQueue (Last In First Out - Depth First)

# 4. Redis Connection
REDIS_URL = 'redis://localhost:6379'
```

> [!TIP]
> **Scrapy Doc Gap: The Batch Size Bottleneck**
> By default, `scrapy-redis` pulls one URL from Redis at a time. If your spider is fast and your Redis server has even a tiny bit of network latency, the spider will spend most of its life waiting for the next link. 
> 
> You can significantly increase speed by setting `REDIS_START_URLS_BATCH_SIZE = 50`. This tells the spider to grab fifty links at once, keeping its local queue full and the engine humming at top speed. It’s a simple setting that can double your crawl speed on high-latency networks!

## Step 5: Feeding the Swarm

Because we are now in a distributed world, we don't just "run" the spider. We "start" the spiders, and then we "feed" them work.

1.  **Launch the spiders:** Open three terminal windows and run `scrapy crawl myswarm` in each one. They will sit there waiting patiently because the Redis queue is empty.
2.  **Feed the queue:** Open a new terminal and use the `redis-cli` to add a URL:
    ```bash
    redis-cli lpush myswarm:start_urls "https://quotes.toscrape.com"
    ```
As soon as you hit Enter, you'll see one of your spiders "wake up," grab the URL, and start working!

## Monitoring the Queue

You can use the `redis-cli` to see how many URLs are waiting in line:
```bash
redis-cli llen myswarm:start_urls
```
If the number is going down, your spiders are winning. If the number is going up, you might need to launch more spiders!

## Advanced: Custom Bloom Filter

The default `RFPDupeFilter` uses a lot of RAM because it stores every fingerprint in a Redis Set. For 100 million URLs, you might need 10GB of RAM just for the duplicates!

**Solution:** Use a **Bloom Filter**.
It's a probabilistic data structure that uses 90% less memory with a tiny chance of error (false positive).

To use it, you need a fork like `scrapy-redis-bloomfilter` and change your settings:
```python
DUPEFILTER_CLASS = "scrapy_redis_bloomfilter.dupefilter.RFPDupeFilter"
BLOOMFILTER_HASH_NUMBER = 6
BLOOMFILTER_BIT = 30
```

## Chapter Summary

**What we covered:**
- Redis is a high-speed, in-memory database that acts as the "Master Notebook" for your spiders.
- **Scrapy-Redis** replaces Scrapy's internal scheduler with a global, shared one.
- Distributed spiders use a `redis_key` instead of a `start_urls` list.
- `SCHEDULER_PERSIST = True` allows your crawl to survive a complete system restart.
- You "feed" a distributed crawl by pushing URLs into the Redis queue using `lpush`.

**Key code:**
```python
class MySpider(RedisSpider):
    name = 'swarm'
    redis_key = 'swarm:start_urls'
```

**Previous chapter:**
[Chapter 24: Understanding Distributed Crawling](./chapter_24_understanding_distributed_crawling.md)

**Next chapter:**
We've built the swarm. But as your speed increases, so does the attention of the websites you visit. In the next few chapters, we're going to talk about the final part of Part V: **Scaling Strategy, Resource Optimization, and Ethics**. We'll learn how to balance massive power with professional responsibility.

---

**End of Chapter 25**
