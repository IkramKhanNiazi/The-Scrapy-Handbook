# Chapter 40: Asynchronous Programming for Scrapers

Think about the first time you try to add a "quick database check" to your spider. You want to skip URLs you've already scraped. Simple enough. You add a database query inside your `parse` method.

Your spider starts. It's slow. Really slow. What used to scrape 100 pages per minute now crawls at 5 pages per minute. You check CPU usage and it's nearly idle. Memory is fine. The network is fast. So why is everything so slow?

You might spend hours profiling before you discover the problem: your database query is **blocking**. Every time your spider waits for the database, the entire Scrapy engine pauses. Those 2 seconds per query, multiplied by thousands of requests, have destroyed your performance.

This is confusing at first. Scrapy isn't just "fast Python code." It's an asynchronous, non-blocking framework built on something called Twisted. And if you don't understand async, you can accidentally cripple it with a single line of code.

In this chapter, you're going to learn to think asynchronously.

---

## Introduction

Scrapy is one of the fastest web scraping frameworks because it's **asynchronous**. While waiting for one website to respond, it can send requests to ten others. But this speed comes with a catch: you need to understand async programming to avoid accidentally blocking the engine.

In this chapter, we're going to explore how Scrapy's async engine works, when blocking is dangerous, and how to integrate modern Python `asyncio` code with Scrapy's Twisted foundation.

## Understanding Async: The Restaurant Analogy

Imagine a restaurant with one waiter (synchronous) vs. one waiter who can handle multiple tables (asynchronous):

**Synchronous waiter:**
1. Takes order from Table 1
2. Walks to kitchen, waits for food
3. Brings food to Table 1
4. *Then* goes to Table 2

**Asynchronous waiter:**
1. Takes order from Table 1, sends to kitchen
2. Takes order from Table 2, sends to kitchen
3. Takes order from Table 3, sends to kitchen
4. Food for Table 1 ready → delivers it
5. Food for Table 3 ready → delivers it
6. ...

The async waiter isn't faster at walking. They're just **not waiting around doing nothing**.

Scrapy is the async waiter. But if you make it wait for something (like a database query), it stops serving all tables.

## The Twisted Reactor

Scrapy is built on **Twisted**, an event-driven networking engine. At its core is the **reactor**, a loop that:
1. Sends HTTP requests
2. Checks if any responses have arrived
3. Runs your callbacks for completed responses
4. Repeats

```
┌─────────────────────────────────────────────┐
│                  REACTOR LOOP               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ Send    │→ │ Check   │→ │ Run     │     │
│  │ Request │  │ Network │  │ Callback│──┐  │
│  └─────────┘  └─────────┘  └─────────┘  │  │
│       ↑                                  │  │
│       └──────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**The Golden Rule:** Never block the reactor.

## What "Blocking" Means

Blocking operations are anything that makes Python wait:

```python
# ❌ BLOCKING - Stops the entire engine
import time
time.sleep(5)  # Nothing else can run for 5 seconds

import requests
data = requests.get('http://api.example.com')  # Blocks until response

import psycopg2
cursor.execute('SELECT * FROM products')  # Blocks until query completes
```

```python
# ✅ NON-BLOCKING - Other work can happen during the wait
import asyncio
await asyncio.sleep(5)  # Reactor continues running

import aiohttp
async with aiohttp.get('http://api.example.com') as resp:
    data = await resp.json()  # Other requests can proceed
```

> [!WARNING]
> **Scrapy Doc Gap: The Hidden `requests` Trap**
> Many developers use the `requests` library for quick API calls inside their spiders. This is a performance disaster! Every `requests.get()` blocks the entire Scrapy engine.
> 
> If you need to make HTTP calls outside of Scrapy's normal request flow, use `scrapy.Request` or an async library like `aiohttp`.

## Async Callbacks in Scrapy

Modern Scrapy (2.0+) supports native `async`/`await`:

```python
class AsyncSpider(scrapy.Spider):
    name = 'async_example'
    
    async def parse(self, response):
        # You can use await inside parse methods!
        items = response.css('.product')
        
        for item in items:
            product = {
                'name': item.css('.name::text').get(),
                'url': response.urljoin(item.css('a::attr(href)').get()),
            }
            
            # Async enrichment - doesn't block other requests
            product['sentiment'] = await self.analyze_sentiment(product['name'])
            
            yield product
    
    async def analyze_sentiment(self, text):
        # Async API call
        async with aiohttp.ClientSession() as session:
            async with session.post(
                'https://api.sentiment.example.com/analyze',
                json={'text': text}
            ) as resp:
                data = await resp.json()
                return data['score']
```

## Integrating asyncio with Twisted

Scrapy uses Twisted, but modern Python uses `asyncio`. To bridge them, you need the async reactor:

```python
# settings.py
TWISTED_REACTOR = 'twisted.internet.asyncioreactor.AsyncioSelectorReactor'
```

Now you can use `asyncio` libraries directly:

```python
import asyncio

class MySpider(scrapy.Spider):
    async def parse(self, response):
        # Async database query
        result = await self.async_db_query(response.url)
        yield {'url': response.url, 'db_data': result}
    
    async def async_db_query(self, url):
        # Using asyncpg (async PostgreSQL)
        import asyncpg
        conn = await asyncpg.connect('postgresql://...')
        result = await conn.fetchone('SELECT * FROM urls WHERE url = $1', url)
        await conn.close()
        return result
```

## Async Database Operations

### The Wrong Way (Blocking)
```python
# ❌ This kills your performance
import psycopg2

def process_item(self, item, spider):
    conn = psycopg2.connect('...')  # Blocks!
    cursor = conn.cursor()
    cursor.execute('INSERT INTO items ...', item)  # Blocks!
    conn.commit()  # Blocks!
    return item
```

### The Right Way (Async)
```python
# ✅ Non-blocking database access
import asyncpg

class AsyncDatabasePipeline:
    async def open_spider(self, spider):
        self.pool = await asyncpg.create_pool('postgresql://...')
    
    async def process_item(self, item, spider):
        async with self.pool.acquire() as conn:
            await conn.execute(
                'INSERT INTO items (name, price) VALUES ($1, $2)',
                item['name'], item['price']
            )
        return item
    
    async def close_spider(self, spider):
        await self.pool.close()
```

## Async-Friendly Libraries

| Purpose | Blocking Library | Async Alternative |
|---------|-----------------|-------------------|
| HTTP | `requests` | `aiohttp`, `httpx` |
| PostgreSQL | `psycopg2` | `asyncpg` |
| MySQL | `mysql-connector` | `aiomysql` |
| MongoDB | `pymongo` | `motor` |
| Redis | `redis` | `aioredis` |
| Files | `open()` | `aiofiles` |

## Using `defer` for Twisted-Style Async

If you prefer Twisted's style (or need compatibility with older code):

```python
from twisted.internet import defer

class TwistedSpider(scrapy.Spider):
    @defer.inlineCallbacks
    def parse(self, response):
        # This is Twisted's version of async/await
        result = yield self.some_deferred_operation()
        yield {'data': result}
```

## Parallel Processing with `asyncio.gather`

Process multiple async operations simultaneously:

```python
async def parse(self, response):
    product_urls = response.css('.product a::attr(href)').getall()
    
    # Fetch all products in parallel
    tasks = [self.fetch_product_details(url) for url in product_urls[:10]]
    results = await asyncio.gather(*tasks)
    
    for result in results:
        yield result

async def fetch_product_details(self, url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            html = await resp.text()
            # Parse and return
            return {'url': url, 'html_length': len(html)}
```

## Common Pitfalls

### Pitfall 1: Forgetting `await`
```python
# ❌ BUG: Returns a coroutine object, not the result
result = self.async_operation()

# ✅ Correct
result = await self.async_operation()
```

### Pitfall 2: Blocking in Pipelines
```python
# ❌ Pipelines can block too!
class BadPipeline:
    def process_item(self, item, spider):
        time.sleep(1)  # Blocks entire engine!
        return item
```

### Pitfall 3: CPU-Bound Operations
```python
# ❌ This is async but still blocks (CPU-bound)
async def process(self, data):
    # Heavy computation runs on the main thread
    result = self.expensive_calculation(data)  # Blocks!
    return result

# ✅ Offload CPU work to a thread pool
import asyncio

async def process(self, data):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, self.expensive_calculation, data)
    return result
```

> [!IMPORTANT]
> **Scrapy Doc Gap: `async` Doesn't Mean "Fast"**
> Making a function `async` doesn't magically make it non-blocking. Only I/O operations that use async-aware libraries (like `aiohttp` instead of `requests`) benefit from async. 
> 
> CPU-intensive work (parsing, regex, calculations) still runs on the main thread and blocks the reactor. For heavy CPU work, use `run_in_executor` to offload to a thread pool.

## Chapter Summary

**What we covered:**
- Scrapy's speed comes from its asynchronous, non-blocking architecture.
- The Twisted reactor is an event loop. Blocking it stops everything.
- Use `async`/`await` in modern Scrapy for clean asynchronous code.
- Replace blocking libraries (`requests`, `psycopg2`) with async alternatives.
- Configure `TWISTED_REACTOR` to enable full `asyncio` integration.
- CPU-bound work still blocks. Offload it with `run_in_executor`.

**Key code:**
```python
# settings.py
TWISTED_REACTOR = 'twisted.internet.asyncioreactor.AsyncioSelectorReactor'

# spider.py
async def parse(self, response):
    result = await self.async_operation()
    yield {'data': result}
```

**Previous chapter:**
[Chapter 39: Testing Your Spiders](./chapter_39_testing_your_spiders.md)

**Next chapter:**
Your spider is async and fast, but something's wrong and you can't figure out what. In the next chapter, we're going to explore **Debugging and Profiling Spiders**, the tools and techniques for finding bugs and performance bottlenecks.

---

**End of Chapter 40**
