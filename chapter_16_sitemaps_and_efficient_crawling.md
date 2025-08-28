# Chapter 16: Sitemaps and Efficient Crawling

Think about the first time you're asked to scrape a news archive with over two million articles. You sit down with your usual "Link Following" spider, where you start at the homepage, click on "Archive," then click on "Year," then "Month," then "Day."

You might do some quick math and realize that at the rate your spider is going, it would take four months just to *find* all the links, let alone scrape the content. Every time the spider visits a daily archive page, it has to download the whole HTML just to find the three links it doesn't have yet. It's incredibly inefficient. You're wasting 90% of your bandwidth and 90% of the website's server energy just on "navigation."

It can feel as if you're trying to map a massive city by walking down every single alleyway to see where it leads. You're exhausted, your logs are a mess, and you're pretty sure the website owner is going to block you for being so annoying.

"There has to be a map," you think. "The website owner *wants* Google to find these pages. They must have a list somewhere."

That's when you discover **XML Sitemaps**.

It's like being handed a master blueprint of the entire city. Instead of walking every alley, you can just look at the map and see exactly where every single article lives. You don't need to visit the "Year" or "Month" pages anymore. You can go straight to the articles. Your four-month project turns into a four-day project. Your spider isn't "crawling" anymore; it's "harvesting."

In this chapter, you're going to learn how to find these master maps and how to use them to make your scraping ten times faster.

---

## Introduction

In the world of professional scraping, speed isn't just about how many requests per second you can send. It's about how few requests you can send to get the data you need. This is **Efficient Crawling**.

The secret weapon for efficiency is the **Sitemap**. Sitemaps are files that website owners create specifically to help search engines (and us!) find their content. In this chapter, we're going to master the **SitemapSpider**, a specialized Scrapy class that can read these files and jump straight to the data without wasting time on navigation.

## What are Sitemaps?

An XML Sitemap is a simple file (usually named `sitemap.xml`) that lists the URLs for a website. It often includes extra information like when the page was last updated and how important it is.

Website owners love sitemaps because they ensure that search engines like Google don't miss any of their pages. Scrapers love them because they are a "shortcut" to 100% coverage of a site's content.

## Finding Sitemaps

There are three main places to look for a sitemap:
1.  **The Root Directory:** Try visiting `https://example.com/sitemap.xml`.
2.  **The robots.txt File:** As we learned in Chapter 2, many site owners list their sitemap URL at the bottom of their `robots.txt` file. Look for a line that says `Sitemap: https://...`.
3.  **The Sitemap Index:** For very large sites, the `sitemap.xml` might just be a "table of contents" that points to ten other sitemaps (like `sitemap-products-1.xml`, `sitemap-products-2.xml`). Scrapy handles these indexes automatically.

## SitemapSpider: The Built-in Roadmap

Scrapy provides a specialized spider class just for this. It's called the `SitemapSpider`.

```python
from scrapy.spiders import SitemapSpider

class MySitemapSpider(SitemapSpider):
    name = 'sitemap_spider'
    sitemap_urls = ['https://example.com/sitemap.xml']
    
    # Optional: Only follow links that match these patterns
    sitemap_rules = [
        ('/products/', 'parse_product'),
        ('/blog/', 'parse_blog'),
    ]

    def parse_product(self, response):
        yield {
            'title': response.css('h1::text').get(),
            'url': response.url,
        }

    def parse_blog(self, response):
        # Different logic for blog posts
        yield {'headline': response.css('h2::text').get()}

> [!TIP]
> **Scrapy Doc Gap: The Sitemap Index Filter**
> On huge sites (like news sites or e-commerce giants), a sitemap isn't just one file. It’s a "Sitemap Index" that points to hundreds of smaller files. 
> 
> By default, Scrapy will try to read *all* of them, which can take a long time. You can use the `sitemap_follow` setting to only read specific sub-sitemaps: `sitemap_follow = ['/products_2026/']`. This lets you target only the newest data without waiting for the spider to read the entire archive.
```

### Why this is better:
1.  **Zero Navigation:** You don't have to write code to find "Next" buttons or "About" links.
2.  **No Duplicates:** The sitemap is a unique list of articles.
3.  **Update Tracking:** Sitemaps often include a `<lastmod>` tag. You can even write code to only scrape pages that have been updated since your last crawl!

### Advanced Sitemap Filtering

You don't just have to filter by URL. You can inspect the sitemap entry itself (e.g., check `<lastmod>` or `<priority>`).

```python
def sitemap_filter(self, entries):
    for entry in entries:
        # entry is a dict: {'loc': 'http...', 'lastmod': '2026-01-01'}
        date_str = entry.get('lastmod')
        if date_str and "2026" in date_str:
            yield entry
```

To use this, set `sitemap_filter` in your spider class. Scrapy will run every entry through this function before generating a request.

## Sitemap vs. Regular Crawling

| Feature | Regular Crawling | Sitemap Crawling |
| :--- | :--- | :--- |
| **Speed** | 🐢 Slower (needs nav) | 🚀 Much faster |
| **Coverage** | ⚠️ Can miss hidden pages | ✅ Usually 100% |
| **Complexity**| ⚠️ High (handling nav) | ✅ Low (read-only) |
| **Availability**| ✅ Always available | ❌ Not every site has one |

**The Professional's Rule:** Always check for a sitemap first. If one exists, use a `SitemapSpider`. If not, fall back to a `CrawlSpider` or a regular `Spider`.

## Efficient Crawling Strategies

Beyond sitemaps, here are three ways to make your crawls more efficient:

### 1. The "Only Changed" Strategy
If you are monitoring a site daily, don't re-scrape everything. Check the date on the page or in the sitemap and only request the URL if it's new.

### 2. Header-Only Requests (HEAD)
If you're unsure if a page has changed, send a `HEAD` request instead of a `GET` request. This asks the server for the "meta-data" only (which is very small). You can check the `Last-Modified` header and only perform the full `GET` request if the page is new.

### 3. DNS Caching
Scrapy can "remember" a website's IP address so it doesn't have to look it up every single time. This saves a few milliseconds on every request, which adds up to minutes over a large crawl.

## Broad Crawls: Scraping the Entire Internet

If you need to scrape 1,000,000 pages from different domains, the default Scrapy settings are too slow and polite. You need "Broad Crawl" tuning.

**Key Settings for High Throughput:**
```python
# settings.py

# 1. Concurrency: Crank it up!
CONCURRENT_REQUESTS = 100
CONCURRENT_REQUESTS_PER_DOMAIN = 100

# 2. Priority: Crawl strictly (BFO Order)
DEPTH_PRIORITY = 1
SCHEDULER_DISK_QUEUE = "scrapy.squeues.PickleFifoDiskQueue"
SCHEDULER_MEMORY_QUEUE = "scrapy.squeues.FifoMemoryQueue"

# 3. Memory: Use less RAM
COOKIES_ENABLED = False # Don't need them for public data
LOG_LEVEL = 'INFO'
```

> [!WARNING]
> **Scrapy Doc Gap: The RAM Killer**
> Broad crawls kill RAM. When `COOKIES_ENABLED = True`, Scrapy keeps a separate cookie jar for *every domain*. If you visit 10,000 domains, you have 10,000 cookie jars in RAM. Always disable cookies for broad crawls unless absolutely necessary.

## State Persistence: Stopping and Resuming

Imagine scraping 900,000 pages, and your power goes out. Do you start over? No.

You use the `JOBDIR` setting.

```bash
scrapy crawl my_spider -s JOBDIR=crawls/spider-1
```

This tells Scrapy: "Save the state of the Scheduler and DupeFilter to this folder."
- If the spider crashes, run the exact same command.
- Scrapy will see the folder, replay the state, and resume exactly where it left off.

## Avoiding Re-Scraping (Bloom Filters)

Scrapy's default duplicate filter uses a Python `set`. It's fast, but it lives in RAM. For 100 million URLs, your RAM will explode.

**Solution: Bloom Filters or DeltaFetch**
- **Bloom Filters**: Identify if a URL has been seen with 99.9% accuracy using tiny memory. (Requires `scrapy-redis` or custom Middleware).
- **DeltaFetch**: A middleware that uses a local database (BerkleyDB) to remember URLs *across* multiple crawl runs. "I scraped this last week, don't initiate a request today."

```python
# Example concept for settings.py if using scrapy-deltafetch
SPIDER_MIDDLEWARES = {
    'scrapy_deltafetch.DeltaFetch': 100,
}
DELTAFETCH_ENABLED = True
```

## Chapter Summary

**What we covered:**
- Sitemaps are XML files that list every important URL on a website.
- The `SitemapSpider` is a specialized Scrapy class that reads these maps automatically.
- Sitemap crawling is the fastest and most reliable way to achieve total coverage of a site.
- You find sitemaps in the root directory or by checking the `robots.txt` file.
- Professional efficiency means sending the fewest requests possible to get the data you need.

**Key code:**
```python
class MySpider(SitemapSpider):
    sitemap_urls = ['http://www.example.com/sitemap.xml']
```

**Previous chapter:**
[Chapter 15: Handling Media Files](./chapter_15_handling_media_files.md)

**Next chapter:**
Even with a perfect roadmap, things can go wrong. Servers crash, internet connections fail, and websites return errors. In the next chapter, we're going to learn about **Error Handling and Retries** how to build robust spiders that can survive any digital storm.

---

**End of Chapter 16**
