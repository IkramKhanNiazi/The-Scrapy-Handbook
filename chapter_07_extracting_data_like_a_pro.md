# Chapter 7: Extracting Data Like a Pro

Think about the first time you try to scrape a large e-commerce site. You've got your selectors ready, and your spider is crawling away, but when you look at the data you've collected, it might be a total mess. Some products have prices, others didn't. Some titles could be cut off, and some links might just be fragments like `/p/12345` instead of full URLs.

It can feel as if you're trying to catch water with a sieve. You're getting *some* data, but most of it is leaking through holes you didn't even know existed.

You might spend an entire weekend trying to "fix" the data after it's already scraped. You'll write complex Python scripts to try and glue the URLs back together and clean up the extra whitespace. It's exhausting. It feels like doing the work twice: once to scrape it, and once to make it usable. You might even start to think that web scraping is just naturally "dirty" and that you'll always have to spend hours cleaning up the mess.

Then, you discover the built-in magic of Scrapy's extraction methods.

You'll realize that doing things the hard way often comes from not yet knowing the "pro" shortcuts. You'll learn how to use `.get()` to avoid errors, how to use `response.follow()` to handle links effortlessly, and how to identify pagination patterns that previously seemed impossible to automate. It's like you've been trying to build a house with a screwdriver, and someone has finally handed you a power drill.

In this chapter, you're going to get that power drill.

---

## Introduction

Finding an element with a selector is only half the battle. The other half is actually pulling the data out of that element and making it usable.

In this chapter, we're going to level up your extraction skills. We'll move beyond the basics and learn how to handle missing data gracefully, how to extract hidden attributes, and how to follow links across the web without manual intervention. We'll also tackle the most common challenge in scraping: **Pagination**.

By the end of this chapter, you won't just be "getting data"; you'll be performing professional-grade data extraction.

## The Extraction Process

In Scrapy, extraction usually follows a three-step cycle:
1.  **Selection:** Pointing to the element (using CSS or XPath).
2.  **Extraction:** Pulling the raw data out of those elements.
3.  **Refinement:** Choosing exactly what part of that data you want (text vs. attributes).

## Using .get() and .getall()

In Chapter 5, we saw these methods briefly. Now let's understand why they are so important.

### `.get()`: Safety First
When you use `.get()`, Scrapy returns a single string. If the selector doesn't find anything, it returns `None`.

```python
# Safe extraction
title = response.css('h1::text').get()
```
Why is this better than older methods? Because if the title is missing, your code won't crash with an error; it will just give you `None`, which you can handle easily.

### `.getall()`: Working with Lists
If you want every item that matches a selector (like every author on a page), use `.getall()`. It will always return a **list**, even if it only finds one item or zero items.

```python
authors = response.css('small.author::text').getall()
```

## Extracting Text Content

Sometimes `::text` isn't enough. Imagine this HTML:
`<p>Price: <b>$19.99</b></p>`

If you use `p::text`, you'll get `"Price: "`. You'll miss the actual price inside the `<b>` tag! 

To get all the text inside an element and its children, you can use a "recursive" selector in XPath:
```python
# Grab all text inside the paragraph, including the bold part
text = response.xpath('//p//text()').getall()
cleaned_text = "".join(text)
```

## Extracting Attributes

Attributes are often more valuable than the text on the screen.
*   **Links:** `<a href="...">`
*   **Images:** `<img src="...">`
*   **Metadata:** `<div data-id="...">`

In CSS, we use `::attr(name)`:
```python
link = response.css('a.next::attr(href)').get()
image_url = response.css('img.thumbnail::attr(src)').get()
```

## Working with Lists and Loops

Professional scrapers almost never extract individual items one by one. They find a "container" and then loop through it. This ensures that the data stays synchronized.

```python
for product in response.css('div.product-card'):
    yield {
        'name': product.css('h2::text').get(),
        'price': product.css('.price::text').get(),
    }
```

## Handling Nested Data

Sometimes a website hides data inside a specific element that requires multiple steps to reach.
```python
# Find the sidebar first
sidebar = response.css('div.sidebar')

# Look inside the sidebar for the menu items
menu_items = sidebar.css('li.nav-item::text').getall()
```

## Discovery: Finding Hidden APIs

Sometimes the best data isn't in the HTML at all. Modern React/Vue/Angular sites often load data via JSON APIs.

**How to spot them:**
1. Right-click > Inspect > Network Tab
2. Filter by "Fetch/XHR"
3. Refresh the page
4. Look for responses that look like data (JSON), not HTML

If you find a JSON endpoint, you've hit the jackpot. You don't need CSS selectors; you just need to parse the dictionary.

```python
import json

def parse(self, response):
    # If the response is JSON, convert it to a Python dict
    data = json.loads(response.text)
    
    # Access data directly
    for item in data['items']:
        yield {
            'name': item['title'],
            'price': item['price']['amount']
        }
```

## Following Links: The Pro Way

One of the most common issues for beginners is dealing with **Relative URLs**. A website might give you a link like `/p/12345`. If you try to visit that address directly, it will fail because it's missing the `https://domain.com` part.

### The Old Way: `urljoin`
```python
relative_url = response.css('a::attr(href)').get()
absolute_url = response.urljoin(relative_url)
```

### The Pro Way: `response.follow()`
As we saw in Chapter 6, `response.follow()` handles the URL joining for you and returns a Request object.
```python
yield response.follow(next_page_url, callback=self.parse)
```

> [!TIP]
> **Scrapy Doc Gap: response.follow vs. scrapy.Request**
> In older tutorials, you'll see people using `scrapy.Request(response.urljoin(url))`. While this works, `response.follow` is much smarter. 
> 
> Not only does it handle the URL joining for you, but it can also take a **Selector** object directly. You can pass the actual `<a>` tag to it: `yield response.follow(item.css('a')[0])`, and Scrapy will find the `href` automatically. It’s a cleaner, less error-prone way to navigate.

## Pagination Patterns

There are three common ways websites handle multiple pages.

### 1. The "Next" Button
The easiest pattern. You find the link for the "Next" button and follow it.
```python
next_page = response.css('li.next a::attr(href)').get()
if next_page:
    yield response.follow(next_page, callback=self.parse)
```

### 2. Page Numbers
Some sites list numbers like `1, 2, 3...`. You can loop through these and visit each one.
```python
page_links = response.css('ul.pagination li a::attr(href)').getall()
for link in page_links:
    yield response.follow(link, callback=self.parse)
```

### 3. Infinite Scroll and Load More
These sites load more data as you scroll down. Scrapy can't "scroll," but typically, the website sends a request to a backend API to get more JSON data.

**The "Hidden API" Pattern:**
1. Open DevTools > Network Tab > XHR/Fetch
2. Scroll down on the page
3. Look for a request that returns JSON data
4. Simulate that request in Scrapy

```python
# Pro Tip: Handling API Pagination
import json

def parse(self, response):
    data = json.loads(response.text)
    for product in data['products']:
        yield {
            'name': product['name'],
            'price': product['price']
        }
    
    # Calculate next page offset
    current_page = response.meta.get('page', 1)
    if data['has_more']:
        next_page = current_page + 1
        yield scrapy.Request(
            url=f'https://api.example.com/products?page={next_page}',
            callback=self.parse,
            meta={'page': next_page}
        )
```

## Advanced Request Management

### Request Priorities

Not all links are equal. You might want to prioritize scraping product pages over category pages to get data faster.

```python
# Priority: Higher number = Higher priority (Default is 0)
yield response.follow(product_url, priority=10)
yield response.follow(next_page_url, priority=1)
```

Scrapy uses a priority queue. By giving product URLs a higher priority, they "jump the line" and get processed before the next pagination link.

### Request Fingerprinting

Scrapy automatically filters out duplicate requests to save time. It generates a "fingerprint" based on the URL, method, and body.

**When to disable filtering:**
Sometimes you *want* to visit the same URL twice (e.g., it updates every minute).

```python
yield scrapy.Request(url, dont_filter=True)
```

> [!WARNING]
> **Scrapy Doc Gap: Canonical URLs**
> Websites often have multiple URLs for the same content:
> - `example.com/product/123`
> - `example.com/product/123?utm_source=google`
> 
> Scrapy sees these as **different** URLs. To fix this, extract the `canonical` link from the HTML head and use that for fingerprinting or deduplication in your pipeline.
> ```python
> canonical_url = response.css('link[rel="canonical"]::attr(href)').get()
> ```

## Common Extraction Mistakes

1.  **Forgetting to `.get()`:** If you forget the final method, you'll get a `Selector` object instead of a string.
2.  **Over-complicating selectors:** Always try the simplest selector first. If `.price::text` works, don't use `div.container > div.main > p.price::text`.
3.  **Ignoring whitespace:** Websites are messy. Always use the `.strip()` method in Python to clean up extra spaces and newlines from your extracted strings.

## Chapter Summary

**What we covered:**
- Professionals use `.get()` and `.getall()` for safe, crash-proof extraction.
- Attributes (like `href` and `src`) are extracted with `::attr()`.
- Looping through containers is the best way to keep data organized.
- `response.follow()` is the ultimate tool for handling relative URLs.
- Pagination is just finding and following the "Next" link correctly.

**Key code:**
```python
# The "Pro" Extraction Pattern
def parse(self, response):
    for item in response.css('.container'):
        yield {
            'text': item.css('.title::text').get().strip(),
            'link': item.css('a::attr(href)').get(),
        }
    
    next_page = response.css('a.next::attr(href)').get()
    if next_page:
        yield response.follow(next_page, callback=self.parse)
```

**Previous chapter:**
[Chapter 6: CSS Selectors and XPath](./chapter_06_css_selectors_and_xpath.md)

**Next chapter:**
Our data is still just a collection of dictionaries. This is fine for small tasks, but for big projects, we need more structure. In the next chapter, we're going to learn about **Scrapy Items and ItemLoaders** the professional way to define and load your data schemas.

---

**End of Chapter 7**
