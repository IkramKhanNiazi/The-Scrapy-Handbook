# Chapter 9: Data Cleaning Best Practices

Think about the first time you present a scraped dataset to a client. You're proud. You've scraped over fifty thousand products from a competitor's website. You open the CSV file to show off your work.

"Look at all this data," you say, beaming.

The client looks at the screen for exactly three seconds and then frowns. "Why does the price column say `$ 1,299.00 * (VAT incl.)` in some rows and `1299` in others? And why are the dates formatted differently? Look at this one: `Jan 12 2026`. This one says `12/01/26`. And this one just says `Yesterday`."

Your heart might sink. You'll realize that while you technically "scraped" the data, you haven't actually made it *useful*. The client can't run any analysis on a column that has a mix of currency symbols, parentheses, and text. They can't sort the products by price, and they can't create a timeline of when the prices changed. To them, your mountain of data is just a mountain of noise.

You might spend the next two days manually writing a cleaning script that feels like it would never end. Every time you fix one pattern, you'll find five more that you missed. It feels like trying to scrub a muddy floor with a toothbrush while someone keeps walking over it with dirty boots.

That experience teaches a vital lesson: **Raw data is almost always dirty.** If you want your scraping to have value, you have to be as good at cleaning data as you are at finding it.

In this chapter, you're going to learn how to automate that "scrubbing" so your data comes out of your spider sparkling clean and ready for analysis.

---

## Introduction

Websites are built by humans, and humans are messy. One developer might use commas for thousands, while another uses periods. One might put the currency symbol before the number, and another might put it after.

As a scraper, your job is to take that mess and turn it into a consistent, predictable format. In this chapter, we're going to explore the most common cleaning tasks, learn when and where to clean your data, and discover the power of **Regular Expressions (Regex)** for handling even the weirdest text patterns.

## Why Clean Data Matters

Think of your data like food. You wouldn't eat a potato straight out of the ground with the dirt still on it. You have to wash it, peel it, and cut it before it's ready for a recipe.

Scraped data is the same. If you want to:
*   Perform mathematical calculations (like averaging prices).
*   Sort items by date.
*   Filter by specific categories.
...you need your data to be in a consistent format (like a float for prices and a `datetime` object for dates).

## When to Clean Data: The Pipeline vs. The Spider

There are three places you can clean your data in Scrapy:

1.  **In the Spider:** Good for quick hacks, but makes your code messy.
2.  **In the ItemLoader:** The best place for "real-time" cleaning as the data is extracted.
3.  **In the Item Pipeline:** The best place for "batch" cleaning or cross-referencing (which we'll cover in Chapter 10).

## Common Cleaning Tasks

### 1. Removing Whitespace and Newlines
Websites often have extra spaces or `\n` characters around text. Python's `.strip()` is your best friend here.
```python
raw_text = "   $19.99 \n  "
clean_text = raw_text.strip() # result: "$19.99"
```

### 4. Unicode Normalization: The Hidden Trap
Websites often mix different types of whitespace (non-breaking spaces, zero-width spaces).

```python
import unicodedata

raw_text = "Price:\u00A0$19.99"  # Contains non-breaking space
# Simple strip() won't catch everything
cleaned = unicodedata.normalize('NFKD', raw_text).strip()
# Result: "Price: $19.99"
```

> [!TIP]
> **Pro Tip: NFKD vs. NFC**
> - **NFKD** breaks characters down (e.g., `é` -> `e` + `´`). Good for search.
> - **NFC** keeps them combined (`é`). Good for display.
> For scraping, NFKD is often useful to standardise distinct characters that look the same.

### 5. Cleaning Prices (The Professional Way)
Writing your own price regex is error-prone. Does it handle `1.200,00 €`? or `R$ 1.500`?

**Use the `price-parser` library:**

```bash
pip install price-parser
```

```python
from price_parser import Price

price = Price.fromstring("1.200,50 €")
print(price.amount)       # Decimal('1200.50')
print(price.amount_float) # 1200.5
print(price.currency)     # '€'
```

### 6. Date Parsing (The Professional Way)
Dates come in infinite formats: "2 days ago", "12/01/2026", "Jan 12th".

**Use the `dateparser` library:**

```bash
pip install dateparser
```

```python
import dateparser

# Parses almost anything into a datetime object
date1 = dateparser.parse('2 days ago')
date2 = dateparser.parse('12/01/2026', settings={'DATE_ORDER': 'DMY'})
```

> [!WARNING]
> **Performance Warning**
> `dateparser` is "magical" but slow. If you know the format is always `YYYY-MM-DD`, use Python's built-in `datetime.strptime()` it's 100x faster. Only use `dateparser` when the format varies or is human-readable.

## Locale-Aware Formatting

European websites often swapp periods and commas: `1.000,00`.

If you just do `float("1.000,00".replace(',', ''))`, you get `1.0`!

**Correct approach:**
```python
text = "1.000,00"
# Remove thousands separator first
clean = text.replace('.', '').replace(',', '.')
value = float(clean) # 1000.0
```

## Regular Expressions (Regex) for Cleaning

Sometimes, simple string methods like `.replace()` aren't enough. Imagine a string like: `"Item #452 (In Stock: 4 units left)"`. You only want the number `4`.

This is where **Regex** comes in. It's a specialized language for pattern matching.
```python
import re

text = "Item #452 (In Stock: 4 units left)"
# Find the first number that follows "In Stock: "
match = re.search(r'In Stock: (\d+)', text)
if match:
    stock_count = match.group(1) # result: "4"
```

> [!WARNING]
> **Scrapy Doc Gap: Regex in Selectors**
> Scrapy allows you to use regex directly on a selector: `response.css('p::text').re(r'\d+')`. 
> 
> However, there is a catch: **`.re()` returns a list of strings, not a selector.** This means you cannot use `.get()` or `.css()` after a `.re()` call. Many beginners try to chain them like `.re(...).get()`, which causes a crash. If you use regex inside a selector, that is the end of the chain!

## Common Regex Patterns for Scrapers

Top patterns you'll use 90% of the time:

| Pattern | Matches | Example |
| :--- | :--- | :--- |
| `\d+` | Numbers | `Price: $100` -> `100` |
| `\d+\.\d+` | Floats | `Price: 19.99` -> `19.99` |
| `[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+` | Emails | `Contact: support@site.com` |
| `sku:\s*(\w+)` | SKU Codes | `SKU: AB-123` -> `AB-123` |
| `\s+` | Whitespace | Use to replace multiple spaces with one |

**Regex in Python:**
```python
import re

text = "  Spec:   Weight 50kg  "
# Clean multiple spaces
clean = re.sub(r'\s+', ' ', text).strip()
# Result: "Spec: Weight 50kg"
```

## Handling Missing Data and None Values

What happens if a product doesn't have a description? Your spider extracts `None`. If you try to call `.strip()` on `None`, your whole spider will crash with a `TypeError`.

**Always handle your Nones!**
```python
description = response.css('.desc::text').get()
# The safe way:
clean_desc = description.strip() if description else ""
```

## A Complete Example: The Cleaning Library

I like to create a separate file named `utils.py` in my project and put all my cleaning functions there. This keeps my spiders clean and lets me reuse the logic.

```python
# utils.py
import re

def clean_title(title):
    if not title:
        return ""
    return title.strip().title()

def clean_price(price_str):
    if not price_str:
        return 0.0
    # Use regex to find only the numbers and the decimal point
    numbers = re.findall(r'[\d\.]+', price_str)
    if numbers:
        return float(numbers[0])
    return 0.0
```

## Chapter Summary

**What we covered:**
- Raw scraped data is dirty and needs to be normalized.
- Cleaning data makes it searchable, sortable, and ready for analysis.
- ItemLoaders are the preferred place to clean data in real-time.
- Regular Expressions (Regex) are essential for extracting patterns from messy text.
- Always check for `None` values to prevent your spider from crashing.

**Key code:**
```python
# The "Safe Cleaning" Pattern
raw_data = response.css('.price::text').get()
if raw_data:
    # Perform cleaning
    cleaned = raw_data.replace('$', '').strip()
else:
    # Provide a default
    cleaned = "0.0"
```

**Previous chapter:**
[Chapter 8: Scrapy Items and ItemLoaders](./chapter_08_scrapy_items_and_itemloaders.md)

**Next chapter:**
Our data is now structured and clean. But how do we decide if a piece of data is "valid"? And how do we stop duplicates from sneaking into our database? In the next chapter, we're going into the **Item Pipelines** the "assembly line" where the final quality control happens.

---

**End of Chapter 9**
