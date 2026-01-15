# Chapter 39: Testing Your Spiders

Think about the first time a client calls you at 2 AM. Their crawler hasn't run in three days. You check the logs and find no errors. You check the output and see empty files. You check the spider and it "ran" perfectly, but brought back zero items.

You might spend the next four hours tracing through the code. Finally, you find the problem: the website changed the class name of their product cards from `.product-item` to `.product-card`. Your selector broke, but it didn't raise an error. It just silently returned nothing.

The frustration is real. Your spider "worked" perfectly. It just worked on a website that no longer exists. And you had no way of knowing something was wrong until three days of missed data had piled up.

That's when you discover that **testing** isn't optional. It's the difference between professionals who sleep soundly and amateurs who get 2 AM phone calls.

In this chapter, you're going to learn how to catch problems before they wake you up.

---

## Introduction

Web scraping is one of the hardest domains to test. Unlike normal code, your spider depends on *external websites* that can change at any moment. But that doesn't mean testing is impossible. It means you need different strategies.

In this chapter, we're going to explore unit testing, contract testing, integration testing, and monitoring. By the end, you'll have a comprehensive quality assurance system that catches bugs before your clients do.

## Why Scrapers Are Hard to Test

Traditional testing assumes your code's behavior is deterministic. With scrapers:

1. **The input changes:** Websites update their HTML daily.
2. **The environment changes:** IP bans, rate limits, CAPTCHAs.
3. **"Working" is subjective:** The spider ran, but is the data correct?

This means we need multiple testing strategies working together.

## Strategy 1: Unit Testing with Mock Responses

The foundation of spider testing is **isolating your parsing logic** from the network.

```python
# tests/test_product_spider.py
import unittest
from scrapy.http import TextResponse, Request
from myproject.spiders.product_spider import ProductSpider

class TestProductSpider(unittest.TestCase):
    def setUp(self):
        self.spider = ProductSpider()
    
    def _create_response(self, html, url='https://example.com/product'):
        """Helper to create a mock response."""
        request = Request(url=url)
        return TextResponse(url=url, body=html.encode('utf-8'), request=request)
    
    def test_parse_product_extracts_name(self):
        html = '''
        <html>
            <h1 class="product-title">Amazing Widget</h1>
            <span class="price">$29.99</span>
        </html>
        '''
        response = self._create_response(html)
        results = list(self.spider.parse_product(response))
        
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0]['name'], 'Amazing Widget')
        self.assertEqual(results[0]['price'], 29.99)
    
    def test_parse_handles_missing_price(self):
        """Spider should not crash if price is missing."""
        html = '<html><h1 class="product-title">Widget</h1></html>'
        response = self._create_response(html)
        results = list(self.spider.parse_product(response))
        
        # Should still return an item, with price as None
        self.assertEqual(results[0]['name'], 'Widget')
        self.assertIsNone(results[0].get('price'))
```

> [!TIP]
> **Scrapy Doc Gap: The `TextResponse` Trap**
> When creating mock responses, always use `TextResponse`, not `HtmlResponse`. The `HtmlResponse` class requires additional encoding detection that can cause confusing test failures. 
> 
> Also, always `.encode('utf-8')` your HTML string. Scrapy responses expect bytes, not strings.

## Strategy 2: Scrapy Contracts

Scrapy has a built-in contract system that lets you embed tests in your spider's docstrings:

```python
class ProductSpider(scrapy.Spider):
    name = 'products'
    
    def parse(self, response):
        """
        @url https://quotes.toscrape.com/
        @returns items 10 100
        @returns requests 0 0
        @scrapes text author
        """
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
            }
```

**Contract annotations:**
- `@url`: The URL to test against.
- `@returns items 10 100`: Expect between 10 and 100 items.
- `@returns requests 0 0`: Expect zero new requests.
- `@scrapes text author`: These fields must exist in each item.

Run contracts with:
```bash
scrapy check products
```

**Limitations:** Contracts test against live websites, so they can be flaky. Use them for smoke testing, not as your primary test suite.

## Strategy 3: Integration Testing with Fixtures

For reliable tests, save real HTML responses and test against them:

```
tests/
  fixtures/
    product_page.html
    category_page.html
    search_results.html
  test_product_spider.py
```

```python
import os
from pathlib import Path

class TestProductSpider(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        fixtures_dir = Path(__file__).parent / 'fixtures'
        cls.product_html = (fixtures_dir / 'product_page.html').read_text()
        cls.category_html = (fixtures_dir / 'category_page.html').read_text()
    
    def test_parse_real_product_page(self):
        response = self._create_response(self.product_html)
        results = list(self.spider.parse_product(response))
        
        # Assert against known values from the fixture
        self.assertEqual(results[0]['name'], 'Sony Camera A7IV')
        self.assertEqual(results[0]['price'], 2499.99)
```

**Capturing fixtures:**
```python
# Run once to save a fixture
import scrapy

class FixtureSpider(scrapy.Spider):
    name = 'fixture_capture'
    
    def parse(self, response):
        with open('tests/fixtures/product_page.html', 'wb') as f:
            f.write(response.body)
```

## Strategy 4: Automated Monitoring with Spidermon

Spidermon is a Scrapy extension for monitoring and validation:

```bash
pip install spidermon
```

```python
# monitors.py
from spidermon import Monitor, MonitorSuite, monitors
from spidermon.contrib.scrapy.monitors import SpiderCloseMonitorSuite

class ItemCountMonitor(Monitor):
    @monitors.name('Minimum items scraped')
    def test_minimum_items(self):
        item_count = self.data.stats.get('item_scraped_count', 0)
        self.assertTrue(
            item_count >= 100,
            msg=f'Only scraped {item_count} items, expected at least 100'
        )

class ProductFieldsMonitor(Monitor):
    @monitors.name('All items have required fields')
    def test_required_fields(self):
        missing_name = self.data.stats.get('spidermon/items/missing_name', 0)
        missing_price = self.data.stats.get('spidermon/items/missing_price', 0)
        
        self.assertEqual(missing_name, 0, 'Some items missing name field')
        self.assertEqual(missing_price, 0, 'Some items missing price field')

class MySpiderMonitorSuite(MonitorSuite):
    monitors = [ItemCountMonitor, ProductFieldsMonitor]
```

Enable in `settings.py`:
```python
SPIDERMON_ENABLED = True
EXTENSIONS = {
    'spidermon.contrib.scrapy.extensions.Spidermon': 500,
}
SPIDERMON_SPIDER_CLOSE_MONITORS = (
    'myproject.monitors.MySpiderMonitorSuite',
)
```

## Strategy 5: Data Validation Pipelines

Catch bad data before it reaches your database:

```python
from pydantic import BaseModel, validator, ValidationError
from scrapy.exceptions import DropItem

class ProductSchema(BaseModel):
    name: str
    price: float
    url: str
    
    @validator('name')
    def name_not_empty(cls, v):
        if len(v.strip()) < 3:
            raise ValueError('Name too short')
        return v.strip()
    
    @validator('price')
    def price_positive(cls, v):
        if v <= 0:
            raise ValueError('Price must be positive')
        return v

class ValidationPipeline:
    def process_item(self, item, spider):
        try:
            validated = ProductSchema(**item)
            return dict(validated)
        except ValidationError as e:
            spider.crawler.stats.inc_value('validation/failed')
            raise DropItem(f'Validation failed: {e}')
```

## Strategy 6: Alerting on Anomalies

Set up alerts when something seems wrong:

```python
class AnomalyDetectionMonitor(Monitor):
    @monitors.name('Check for unusual patterns')
    def test_no_anomalies(self):
        items = self.data.stats.get('item_scraped_count', 0)
        errors = self.data.stats.get('log_count/ERROR', 0)
        
        # Alert if error rate is too high
        if items > 0:
            error_rate = errors / items
            self.assertTrue(
                error_rate < 0.05,
                f'Error rate {error_rate:.1%} exceeds 5% threshold'
            )
        
        # Alert if item count drops dramatically
        # (you'd compare against historical data in production)
        self.assertTrue(
            items > 50,
            f'Only {items} items scraped - possible site change?'
        )
```

## The Testing Pyramid for Scrapers

```
         ▲
        /┃\    Live Monitoring (Spidermon)
       / ┃ \   - Runs after every crawl
      /  ┃  \  - Catches production issues
     /───┃───\
    /    ┃    \   Integration Tests (Fixtures)
   /     ┃     \  - Real HTML, mocked network
  /      ┃      \ - Runs in CI/CD
 /───────┃───────\
/        ┃        \   Unit Tests
──────────────────    - Pure parsing logic
                      - Fast, reliable
```

## CI/CD Integration

Run tests automatically on every code change:

```yaml
# .github/workflows/test.yml
name: Spider Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run unit tests
        run: pytest tests/ -v
      
      - name: Run Scrapy contracts
        run: scrapy check --list | xargs -I {} scrapy check {}
```

## Chapter Summary

**What we covered:**
- Web scrapers need special testing strategies because they depend on external websites.
- **Unit tests** isolate parsing logic using mock `TextResponse` objects.
- **Contracts** provide quick smoke tests embedded in docstrings.
- **Fixtures** save real HTML for reliable integration testing.
- **Spidermon** monitors production crawls and alerts on anomalies.
- **Validation pipelines** catch bad data before it corrupts your database.
- The testing pyramid balances speed, reliability, and coverage.

**Previous chapter:**
[Chapter 38: Anti-Bot Bypass Techniques](./chapter_38_anti_bot_bypass_techniques.md)

**Next chapter:**
Your tests are passing, but is your spider *fast*? In the next chapter, we're going to explore **Asynchronous Programming for Scrapers**, understanding how Scrapy's async engine works and how to squeeze maximum performance from your code.

---

**End of Chapter 39**
