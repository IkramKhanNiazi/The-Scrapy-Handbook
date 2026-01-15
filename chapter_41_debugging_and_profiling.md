# Chapter 41: Debugging and Profiling Spiders

Think about the first time your spider finishes a run with zero items. No errors in the logs. No exceptions. It just ran and found nothing.

You stare at the code. The selectors look correct. The URL is right. You run it again and get still nothing. You add print statements everywhere. You try different URLs. You spend four hours changing random things, hoping something will work.

Finally, desperate, you open the website in your browser and view the page source. That's when you see it: the website now uses `div.card` instead of `div.product-item`. Your selector was asking for something that no longer exists.

It's frustrating. Print statements and guessing aren't debugging. They're flailing. Real debugging is systematic. It's about using the right tools to see exactly what your spider sees, exactly what it's doing, and exactly where it's failing.

In this chapter, you're going to learn how to stop guessing and start investigating.

---

## Introduction

Professional developers spend more time debugging than writing new code. The difference between an amateur and an expert isn't that experts write bug-free code. It's that experts find and fix bugs faster.

In this chapter, we're going to explore the debugging tools built into Scrapy, techniques for isolating problems, and profiling methods for finding performance bottlenecks. By the end, you'll approach bugs with confidence instead of frustration.

## The Scrapy Shell: Your Detective Workshop

The Scrapy shell is the single most important debugging tool. It lets you interact with a webpage in real-time:

```bash
scrapy shell 'https://quotes.toscrape.com'
```

Once inside:
```python
>>> response.css('div.quote span.text::text').getall()
['"The world as we have created it is a process of our thinking..."', ...]

>>> response.css('.nonexistent::text').get()
# Returns None - now you know your selector is wrong!

>>> response.xpath('//title/text()').get()
'Quotes to Scrape'
```

> [!TIP]
> **Scrapy Doc Gap: Shell with Custom Settings**
> The shell uses default settings, which might differ from your spider. To debug with your actual settings:
> ```bash
> scrapy shell -s USER_AGENT='MyBot/1.0' -s ROBOTSTXT_OBEY=False 'https://example.com'
> ```
> This ensures you're seeing exactly what your spider would see.

### Shell Shortcuts

```python
# View raw HTML
view(response)  # Opens in browser

# Check response status
response.status

# Check actual URL (after redirects)
response.url

# Check response headers
response.headers

# Inspect request that led here
response.request.headers
```

## The `scrapy parse` Command

Test a specific callback without running the entire spider:

```bash
scrapy parse 'https://example.com/product/123' --callback parse_product
```

This runs *just* that callback and shows you what it yields:

```
>>> scrapy parse 'https://quotes.toscrape.com' --callback parse -d 2
[
  {'text': '"The world as we have created..."', 'author': 'Albert Einstein'},
  ...
]
# Scraped 10 items
```

**Useful flags:**
- `--callback`: Specify which method to call
- `-d 2`: Follow links 2 levels deep
- `--pipelines`: Run items through pipelines
- `--noitems`: Only show requests, not items

## Logging: More Than Print Statements

Scrapy has a powerful logging system. Use it instead of `print()`:

```python
class MySpider(scrapy.Spider):
    def parse(self, response):
        self.logger.debug(f'Parsing {response.url}')
        
        items = response.css('.product')
        if not items:
            self.logger.warning(f'No products found on {response.url}')
            return
        
        for item in items:
            self.logger.info(f'Found product: {item.css(".name::text").get()}')
            yield {...}
```

**Log levels:**
- `DEBUG`: Detailed info for debugging
- `INFO`: General operational messages
- `WARNING`: Something unexpected but not fatal
- `ERROR`: Something failed
- `CRITICAL`: System is unusable

**Configure in settings:**
```python
LOG_LEVEL = 'DEBUG'  # Show everything
LOG_FILE = 'spider.log'  # Save to file
LOG_FORMAT = '%(asctime)s [%(name)s] %(levelname)s: %(message)s'
```

## Debugging with Breakpoints

Add Python breakpoints to pause execution:

```python
def parse(self, response):
    items = response.css('.product')
    
    import pdb; pdb.set_trace()  # Debugger stops here
    # Or in Python 3.7+:
    breakpoint()
    
    for item in items:
        yield {...}
```

When the breakpoint hits:
```
(Pdb) len(items)
0
(Pdb) response.css('.product-card').getall()  # Try different selector
['<div class="product-card">...']
(Pdb) c  # Continue execution
```

**Common pdb commands:**
- `n`: Next line
- `s`: Step into function
- `c`: Continue until next breakpoint
- `p variable`: Print variable
- `l`: Show current code location

## Inspecting Requests and Responses

### The "Empty Response" Problem

Your spider gets a response, but it's empty or wrong:

```python
def parse(self, response):
    # Debug: What did we actually receive?
    self.logger.debug(f'Status: {response.status}')
    self.logger.debug(f'Body length: {len(response.body)}')
    self.logger.debug(f'First 500 chars: {response.text[:500]}')
    
    # Save for inspection
    with open('debug_response.html', 'wb') as f:
        f.write(response.body)
```

### The "Wrong Content" Problem

Sometimes you get HTML, but not the HTML you expected (blocked, redirected, served mobile version):

```python
def parse(self, response):
    # Check if we got blocked
    if 'Access Denied' in response.text or response.status == 403:
        self.logger.error(f'BLOCKED: {response.url}')
        return
    
    # Check if redirected
    if response.url != response.request.url:
        self.logger.warning(f'Redirected from {response.request.url} to {response.url}')
```

## Memory Profiling

Scrapy spiders can leak memory, especially when processing millions of items.

### Built-in Memory Debugging
```python
# settings.py
MEMUSAGE_ENABLED = True
MEMUSAGE_WARNING_MB = 512
MEMUSAGE_LIMIT_MB = 1024
MEMUSAGE_NOTIFY_MAIL = ['admin@example.com']
```

### Python's tracemalloc
```python
import tracemalloc

class MemorySpider(scrapy.Spider):
    def start_requests(self):
        tracemalloc.start()
        yield scrapy.Request(...)
    
    def closed(self, reason):
        snapshot = tracemalloc.take_snapshot()
        top_stats = snapshot.statistics('lineno')
        
        self.logger.info('Memory usage by line:')
        for stat in top_stats[:10]:
            self.logger.info(stat)
```

### Finding Memory Leaks
Common causes:
1. **Storing items in spider attributes:** Don't do `self.all_items.append(item)`
2. **Large response bodies in memory:** Process and discard
3. **Uncontrolled request queues:** Check `CONCURRENT_REQUESTS`

## Performance Profiling

### Scrapy Stats
Scrapy automatically collects stats:

```python
def closed(self, reason):
    stats = self.crawler.stats.get_stats()
    for key, value in stats.items():
        self.logger.info(f'{key}: {value}')
```

Key stats to watch:
- `item_scraped_count`: Total items
- `downloader/response_count`: Requests completed
- `scheduler/dequeued`: Requests processed
- `elapsed_time_seconds`: Total runtime
- `log_count/ERROR`: Number of errors

### Python cProfile
Profile your parsing functions:

```bash
python -m cProfile -s cumulative my_script.py
```

Or inside code:
```python
import cProfile
import pstats

def parse(self, response):
    profiler = cProfile.Profile()
    profiler.enable()
    
    # Your parsing logic
    result = self.extract_products(response)
    
    profiler.disable()
    stats = pstats.Stats(profiler).sort_stats('cumulative')
    stats.print_stats(10)  # Top 10 slowest functions
    
    return result
```

## Common Bugs and Solutions

### Bug 1: "Spider yields nothing"
**Symptoms:** No items, no errors.

**Debug steps:**
1. Is the selector correct? → Test in Scrapy shell
2. Is JavaScript required? → View page source vs. Elements
3. Are you blocked? → Check response body for CAPTCHA/error

### Bug 2: "Spider is extremely slow"
**Symptoms:** 1-2 requests per minute.

**Debug steps:**
1. Check `DOWNLOAD_DELAY`: Is it too high?
2. Check logs for retries (429, 503 errors)
3. Are you blocking the reactor? (sync database calls)

### Bug 3: "Encoding garbage in output"
**Symptoms:** `â€™` instead of `'`

**Debug steps:**
```python
# Check response encoding
self.logger.debug(f'Encoding: {response.encoding}')

# Force correct encoding
response = response.replace(encoding='utf-8')
text = response.css('.content::text').get()
```

### Bug 4: "Requests not being made"
**Symptoms:** Scheduler stays at 0 requests.

**Debug steps:**
1. Is `start_urls` correct?
2. Is `parse` method yielding `Request` objects?
3. Check duplicate filter: `dont_filter=True`

> [!WARNING]
> **Scrapy Doc Gap: The Silent `None` Yield**
> If your `parse` method doesn't explicitly yield or return, Python returns `None`. Scrapy silently ignores `None` values.
> 
> A common bug is forgetting to yield inside a loop:
> ```python
> def parse(self, response):
>     for url in urls:
>         scrapy.Request(url)  # BUG: Missing "yield"!
> ```

## The Debugging Checklist

When something goes wrong, follow this order:

1. **Check the logs** - Look for ERROR or WARNING messages
2. **Inspect the response** - Save it to a file, open in browser
3. **Test selectors in shell** - Verify they match the actual HTML
4. **Add strategic logging** - Log inputs and outputs of each step
5. **Check network activity** - Are requests actually being made?
6. **Check stats** - `item_scraped_count`, `log_count/ERROR`
7. **Use breakpoints** - Step through the logic

## Chapter Summary

**What we covered:**
- The Scrapy shell is essential for testing selectors interactively.
- `scrapy parse` lets you test specific callbacks in isolation.
- Use `self.logger` instead of `print()` for professional debugging.
- Breakpoints (`pdb.set_trace()`) pause execution for interactive debugging.
- Memory profiling with `tracemalloc` finds leaks in long-running spiders.
- Performance profiling with `cProfile` identifies slow functions.
- Most bugs fall into predictable patterns with systematic solutions.

**Key code:**
```bash
# Test selectors interactively
scrapy shell 'https://example.com'

# Test a specific callback
scrapy parse 'https://example.com/page' --callback parse_product
```

**Previous chapter:**
[Chapter 40: Asynchronous Programming for Scrapers](./chapter_40_async_programming.md)

**Next chapter:**
You've scraped the data and stored it in a database. But how do you make it accessible to the world? In the next chapter, we're going to build **APIs with Your Scraped Data**, creating RESTful endpoints with FastAPI to serve your data professionally.

---

**End of Chapter 41**
