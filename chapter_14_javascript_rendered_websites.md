# Chapter 14: JavaScript-Rendered Websites

Think about the first time you try to scrape a modern flight-booking website. You're looking for ticket prices, and you feel confident. You've found the exact `<span>` tag in your browser's "Inspect" panel. You write your CSS selector, build your spider, and hit "Run."

The spider returns an empty list.

You might be confused. You check the selector again; it's perfect. You check the URL; it's correct. You even try running the same CSS selector in the Scrapy Shell, and it still comes back with nothing. It can feel as if you're looking at a mirage. You can see the data in your browser, but your code is telling you the page is empty.

You might spend the next six hours in a cycle of frustration. You might think you've broken your Scrapy installation, or that the website is using some kind of high-tech anti-scraping shield. You might even try to copy the entire HTML manually, only to find thousands of lines of code that don't seem to contain any prices.

Then, you learn about the "View Page Source" test.

You right-click on the page and select "View Page Source." Your heart might sink. The HTML is almost empty just a few lines of code and a giant block of JavaScript. You'll realize that the website you're looking at doesn't really "exist" until you open it in a browser. A browser like Chrome has a "JavaScript Engine" that runs the code and builds the page on the fly. Scrapy, by itself, does not. Scrapy was just downloading the empty "skeleton," while the "meat" (the prices) was being created by JavaScript after the page loaded.

In this chapter, you're going to learn how to give your spiders a "browser-brain" so they can see even the most elusive dynamic content.

---

## Introduction

The web has changed. In the old days, a server sent a complete HTML page to your computer. Today, many modern websites (built with frameworks like React, Vue, or Angular) send a tiny empty page and a large JavaScript file. The JavaScript then "renders" the page in your browser.

If you use a standard Scrapy spider on these sites, you'll get an empty result. In this chapter, we're going to learn how to identify these "JavaScript roadblocks" and how to overcome them using **Playwright** a tool that lets Scrapy control a real web browser to see exactly what a human sees.

## The JavaScript Problem

Why doesn't Scrapy just include a JavaScript engine? Because JavaScript is slow and heavy. Running a full browser engine to scrape a page takes ten times more memory and five times more time than a simple HTML request.

As professional scrapers, we always try to avoid using a browser if we can. But sometimes, we have no choice.

## Identifying JavaScript-Rendered Content

How do you know if you need Playwright? Use the **"View Source Test"**.
1. Open the website in your browser.
2. Right-click and choose **"View Page Source"** (NOT "Inspect").
3. Press `Ctrl+F` and search for the data you want (like a price).
4. **If the data is NOT in the source code**, it is being rendered by JavaScript. You'll need a browser-based solution.

## Your Options for JavaScript

When you hit a JS roadblock, you have three professional paths:
1.  **Find the "Hidden" API:** Sometimes the JavaScript is getting its data from a JSON endpoint. If you can find that URL in the "Network" tab, you can scrape it directly without a browser! (We'll cover this in Chapter 16).
2.  **Scrapy-Playwright:** This is the modern, powerful standard. It integrates effortlessly with Scrapy.
3.  **Scrapy-Splash:** An older solution that uses a separate server (Docker container) with Lua scripts.
   - **Pros:** Lightweight compared to full browsers.
   - **Cons:** No longer actively maintained, limited JS support, requires learning Lua.
   - **Verdict:** Use Playwright for new projects. Only use Splash if maintaining legacy code.

> [!NOTE]
> **Selenium?**
> You might hear about Selenium. It was the standard for years, but Playwright is faster, more reliable, and has native async support which matches Scrapy's architecture perfectly. We recommend sticking to Playwright.

## Scrapy-Playwright Integration

Playwright is a tool built by Microsoft that can control Chromium (Chrome), Firefox, and Safari. To use it with Scrapy, we use the `scrapy-playwright` library.

### Installation
```bash
pip install scrapy-playwright
playwright install
```

### Configuration
You need to tell Scrapy to use the Playwright "Downloader Handler" in your `settings.py`:
```python
# settings.py
DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
```

## Basic Usage in a Spider

To make a specific request use Playwright, you just add a `meta` flag:

```python
import scrapy

class DynamicSpider(scrapy.Spider):
    name = 'dynamic'

    def start_requests(self):
        yield scrapy.Request(
            'https://example-dynamic.com',
            meta={"playwright": True},
            callback=self.parse
        )

    def parse(self, response):
        # Now response contains the fully rendered HTML!
        yield {'title': response.css('h1::text').get()}
```

## Handling Infinite Scroll

One of the most common JS patterns is "Infinite Scroll." You scroll down, and more items appear. Scrapy-Playwright can handle this by executing custom JavaScript inside the browser:

```python
def start_requests(self):
    yield scrapy.Request(
        url,
        meta={
            "playwright": True,
            "playwright_page_methods": [
                # Scroll to the bottom and wait 2 seconds
                {"method": "evaluate", "args": "window.scrollTo(0, document.body.scrollHeight)"},
                {"method": "wait_for_timeout", "args": 2000},
            ],
        },
    )
```python
from scrapy_playwright.page import PageMethod

def start_requests(self):
    yield scrapy.Request(
        url,
        meta={
            "playwright": True,
            "playwright_page_methods": [
                PageMethod("wait_for_selector", "div.product-card"),
                # Execute custom scrolling funtion
                PageMethod("evaluate", "window.scrollBy(0, document.body.scrollHeight)"),
                PageMethod("wait_for_timeout", 2000), # Wait for content to load
            ],
        },
    )
```

### Waiting for Specific Responses (Network Idle)

Sometimes "scrolling" isn't enough. You need to wait for a network request to finish.

```python
def start_requests(self):
    yield scrapy.Request(
        url,
        meta={
            "playwright": True,
            "playwright_page_methods": [
                # Wait for the JSON api response
                PageMethod("wait_for_response", lambda res: "api/products" in res.url),
            ],
        }
    )
```

## Advanced Interaction: Clicking and Typing

If you need to click a button to reveal a phone number:

```python
async def parse(self, response):
    page = response.meta["playwright_page"]
    
    # Click the "Show Number" button
    await page.click("button.show-phone")
    
    # Wait for the number to appear
    await page.wait_for_selector(".phone-number")
    
    # Extract the new HTML
    content = await page.content()
    sel = scrapy.Selector(text=content)
    phone = sel.css(".phone-number::text").get()
    
    await page.close()
    yield {'phone': phone}
```

> [!TIP]
> **Pro Tip: Avoiding Closures**
> When defining `playwright_page_methods` inside `meta`, you cannot use Python lambda functions if you are using a remote Scrapy (like Scrapy Cloud) because they cannot be serialized (pickled). Stick to strings or define the functions at the module level.

## Taking Screenshots

Sometimes you want to see what the browser is seeing to debug a tricky layout. Playwright makes this a single line:
```python
meta={"playwright": True, "playwright_include_page": True}
# Inside parse:
page = response.meta['playwright_page']
await page.screenshot(path="debug.png")
```

## When to Use Playwright vs. Regular Scrapy

| Feature | Regular Scrapy | Scrapy-Playwright |
| :--- | :--- | :--- |
| **Speed** | 🚀 Blazing fast | 🐌 Slow |
| **Resources** | 🍃 Low RAM | 🐘 High RAM |
| **JS Support** | ❌ None | ✅ Full |
| **Complexity** | ✅ Simple | ⚠️ More complex setup |

**The Golden Rule:** Use regular Scrapy 90% of the time. Only reach for Playwright when the data is literally impossible to get without a browser engine.

> [!CAUTION]
> **Scrapy Doc Gap: The Playwright Overhead**
> The Scrapy documentation makes it sound easy to "just use Playwright." However, in a production environment, Playwright is a **resource hog**. 
> 
> Running a headless browser uses significantly more RAM and CPU than a standard request. If you try to run 50 concurrent Playwright spiders on a small VPS, your server will likely crash. Always prioritize finding a "Hidden API" (see Chapter 16) before resorting to Playwright. It’s the difference between a scalpel and a sledgehammer.

## Chapter Summary

**What we covered:**
- Modern websites use JavaScript to "hide" data from standard scrapers.
- The "View Page Source" test is the easiest way to identify JS-rendered content.
- Playwright is the industry-standard tool for controlling browsers with Scrapy.
- You can trigger Playwright by adding `meta={"playwright": True}` to your requests.
- Playwright allows you to handle complex interactions like scrolling and clicking.
- Browser-based scraping is powerful but expensive; use it only when necessary.

**Key code:**
```python
yield scrapy.Request(url, meta={'playwright': True})
```

**Previous chapter:**
[Chapter 13: Forms and Authentication](./chapter_13_forms_and_authentication.md)

**Next chapter:**
We've conquered text, forms, and JavaScript. But what about images, PDFs, and files? In the next chapter, we're going to dive into **Handling Media Files** learning how to download and organize millions of files with Scrapy's built-in media pipelines.

---

**End of Chapter 14**
