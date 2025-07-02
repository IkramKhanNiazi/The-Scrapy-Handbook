# Chapter 12: Advanced Spider Patterns

Imagine the first time you have to scrape a massive electronics catalog. The website has over twenty thousand products organized into hundreds of categories. Your usual spider pattern, where you manually find the "Next" button and follow it, could quickly become a disaster. You might find yourself writing complex nested loops and callback functions that look like a plate of spaghetti. Every time the website changes a single menu link, your whole logic could fall apart.

It can feel as if you're trying to map a dense jungle by hand, drawing every single tree one by one. You're exhausted, and mistakes start to creep in. You might accidentally crawl the same category twice, or miss an entire section because you forgot to check for a specific "Sub-category" button.

You're working against the sheer scale of the web. Trying to be too precise can make your code fragile.

Then, a senior developer looks at your screen and laughs. "You're building a manual excavator," they say. "Why don't you just build a self-driving car?"

They introduce you to the **CrawlSpider**.

It's a total shift in perspective. Instead of telling the spider exactly where to go step-by-step, you just give it a set of **Rules**. You tell it, "Follow any link that has the word `category` in it, and if you find a link that looks like a `product`, use this extraction logic." Suddenly, your messy code can turn into elegant, powerful rules. Your spider isn't just following a path anymore; it's exploring the entire domain on its own.

In this chapter, you're going to learn how to build your own "self-driving" spiders.

---

## Introduction

So far, we've used the basic `scrapy.Spider`. It's great for simple tasks, but for large-scale crawling, there's a better way. In this chapter, we're going to explore the **CrawlSpider** a specialized version of the spider designed for exploring entire websites with minimal code.

We'll also learn how to use **LinkExtractors** to define exactly which links our spider should follow and how to pass **Arguments** to our spiders so they can be dynamic and flexible. This is the chapter where you stop building individual scrapers and start building professional crawling systems.

## CrawlSpider: The Professional Crawler

The `CrawlSpider` is a class that comes with Scrapy. Its main feature is a list of **Rules**. These rules define the behavior of the spider as it moves through a site.

Instead of writing a `parse` method and manually yielding `response.follow()` calls, the `CrawlSpider` does the following for you automatically.

## Rules and LinkExtractors

A `Rule` tells Scrapy what to do when it finds a link. To find the links, we use a **LinkExtractor**.

```python
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class MyCrawlSpider(CrawlSpider):
    name = 'electronics_crawler'
    allowed_domains = ['example-electronics.com']
    start_urls = ['https://example-electronics.com']

    rules = (
        # Rule 1: Follow any link that contains '/category/'
        Rule(LinkExtractor(allow=r'/category/')),
        
        # Rule 2: If a link contains '/product/', follow it and run 'parse_item'
        Rule(LinkExtractor(allow=r'/product/'), callback='parse_item'),
    )

    def parse_item(self, response):
        # Your extraction logic here
        yield {'url': response.url}

> [!CAUTION]
> **Scrapy Doc Gap: The "parse" Trap**
> In a normal spider, you *must* use the name `parse`. In a `CrawlSpider`, you **must NOT** use the name `parse`. 
> 
> The `CrawlSpider` uses the `parse` method internally to handle all the complex logic of following links and checking rules. If you define your own `parse` method in a `CrawlSpider`, you will "override" the framework's brain, and your rules will stop working. Always use a different name like `parse_item` or `parse_details` for your callbacks.
```

```

### Advanced LinkExtractor Parameters

LinkExtractors are smarter than just regex. You can tell them exactly *where* to look on the page. This is critical for performance and accuracy.

**restrict_css / restrict_xpaths**
Instead of searching the whole page, only search inside a specific area (like the pagination bar).

```python
# Only follow links inside the <div class="pagination">
Rule(LinkExtractor(restrict_css='.pagination'), follow=True)
```

**process_value**
Sometimes links are messy (e.g., `javascript:goTo('123')`). You can pass a function to clean them before Scrapy sees them.

```python
def clean_link(value):
    # Turn "javascript:goTo('123')" into "http://site.com/item/123"
    return value.replace("javascript:goTo('", "http://site.com/item/").replace("')", "")

Rule(LinkExtractor(process_value=clean_link))
```

### Filtering Links (`process_links`)

You can modify the list of links *after* they are extracted but *before* they are followed.

```python
class MySpider(CrawlSpider):
    rules = (
        Rule(LinkExtractor(), process_links='filter_links', callback='parse_item'),
    )

    def filter_links(self, links):
        # Only follow links that have an odd length (just an example)
        return [l for l in links if len(l.url) % 2 != 0]
```

## Crawling Strategy: BFS vs. DFS

By default, Scrapy uses **Depth-First Search (DFS)**. It tries to dive deep into the site as fast as possible.
- **DFS**: Homepage -> Cat 1 -> Product A -> Related Product B...
- **BFS (Breadth-First)**: Homepage -> Cat 1 -> Cat 2 -> Cat 3...

**When to use BFS:**
If you want to scrape data from multiple categories simultaneously or if you fear getting stuck in a deep "rabbit hole" of related products.

**Enable BFS:**
```python
# settings.py
DEPTH_PRIORITY = 1
SCHEDULER_DISK_QUEUE = 'scrapy.squeues.PickleFifoDiskQueue'
SCHEDULER_MEMORY_QUEUE = 'scrapy.squeues.FifoMemoryQueue'
```

> [!TIP]
> **Pro Tip: Depth Limit**
> You can stop the spider from going too deep:
> ```python
> # settings.py
> DEPTH_LIMIT = 3  # Only go 3 clicks away from start_url
> ```

## Handling Different Page Types

One of the biggest advantages of the `CrawlSpider` is that you can have different extraction logic for different types of pages.
*   Category Page: No extraction, just follow more links.
*   Product Page: Extract name, price, and specs.
*   Review Page: Extract user names and ratings.

By pointing different `Rules` to different `callback` methods, your code stays perfectly organized.

## Passing Arguments to Spiders

Sometimes you want your spider to be dynamic. Maybe you want to tell it which category to scrape from the command line.

We do this with the `-a` flag:
```bash
scrapy crawl my_spider -a category=electronics
```

In your Python code, you can access this argument like this:
```python
class MySpider(scrapy.Spider):
    name = 'my_spider'

    def __init__(self, category=None, *args, **kwargs):
        super(MySpider, self).__init__(*args, **kwargs)
        self.category = category
        self.start_urls = [f'https://example.com/{category}']

### Dynamic Rules 

If you want to use arguments to change your Rules (e.g., only follow links for the specific category):

```python
class DynamicRuleSpider(CrawlSpider):
    def __init__(self, category=None, *args, **kwargs):
        # 1. Define rules based on argument
        self.rules = (
            Rule(LinkExtractor(allow=f'/{category}/'), callback='parse_item'),
        )
        # 2. Call super (which compiles the rules)
        super().__init__(*args, **kwargs)
```
```

## Request Meta Data: Passing Info Between Pages

Imagine you find the "Product ID" on the list page, but the "Price" is only on the detail page. You need to pass that ID from the first method to the second one.

We do this using the `meta` dictionary:
```python
# In the first method:
yield response.follow(url, callback=self.parse_details, meta={'product_id': '123'})

# In the second method (parse_details):
product_id = response.meta['product_id']
```

## Chapter Summary

**What we covered:**
- **CrawlSpider** is a specialized class for exploring websites based on rules.
- **LinkExtractors** use patterns to decide which links to follow and which to ignore.
- **Rules** connect LinkExtractors to specific callback methods.
- **Spider Arguments** let you pass data into your spider from the command line.
- **Meta Data** is the "envelope" used to carry information from one page to another.

**Key code:**
```python
Rule(LinkExtractor(allow='/items/'), callback='parse_item', follow=True)
```

**Previous chapter:**
[Chapter 11: Exporting Your Data](./chapter_11_exporting_your_data.md)

**Next chapter:**
You've finished **Part II: Extracting and Cleaning Data**. You are now a competent, professional-grade scraper. But the internet is full of obstacles. In the next part, **Handling Complex Scenarios**, we're going to learn how to deal with forms, logins, and the biggest challenge of all: JavaScript-heavy websites.

---

**End of Chapter 12**
