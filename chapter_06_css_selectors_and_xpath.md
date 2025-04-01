# Chapter 6: CSS Selectors and XPath

In the early days of your scraping career, you might treat a website's code like a giant, terrifying haystack. You know there's a needle (a piece of data) in there somewhere, but you don't have a magnet. You're just reaching in blindly and hoping for the best.

You might open the "Source" of a page, see ten thousand lines of code, and your brain could just shut down. You'd try to use the "Find" feature on your keyboard to look for a specific price, only to find that the price is mentioned fifteen different times on the page in fifteen different places.

You might feel like an archaeologist trying to read an ancient language without a dictionary. You're just guessing. You write a scraper that works for one product, but as soon as you try it on the next product, it breaks because the "structure" is slightly different. You spend more time fixing broken scrapers than actually collecting data. You might feel frustrated, as if you're failing at something that should be simple.

Then, you learn about the "Inspect" tool and the concept of "Selectors." You'll realize: "Stop looking for the text. Look for the path. A website is a map, and every piece of data has an address. If you know how to write the address, you can find the data every single time."

In this chapter, we're going to give you that dictionary. We're going to master the two most powerful languages in web scraping: **CSS** and **XPath**. By the time you finish this chapter, you'll never "guess" again. You'll be able to point your magnet at any haystack and pull out exactly what you need.

---

## Introduction

To be a great web scraper, you don't need to know how to *build* a website. You just need to know how to *locate* things inside one.

As we learned in Chapter 3, a website is a collection of interest-linked "boxes" called HTML elements. Every element has a unique identity and a position in the document. **Selectors** are the patterns we use to point to those specific elements.

In this chapter, we're going to dive deep into two types of selectors:
1.  **CSS Selectors:** The modern, fast, and easy-to-read standard used by web designers.
2.  **XPath:** The "heavy-duty" query language that can do things CSS simply can't.

By the end of this chapter, you'll be able to find any piece of data on any website, no matter how deeply it's hidden.

## Why Selectors Matter

Imagine you are looking at a product page. You want the price.
You could tell Scrapy: "Find the text that has a dollar sign."
But wait! There's a "List Price," a "Member Price," and a "Shipping Cost," all with dollar signs.

A **Selector** lets you be specific. You can say: "Find the `<span>` tag that has the class `current-price` which is inside the `<div>` with the ID `product-details`." This specificity is what makes your scrapers reliable.

## CSS Selectors Fundamentals

CSS (Cascading Style Sheets) selectors are the same tool web designers use to decide which text should be blue or which box should be large.

### 1. Tag Selectors
Targets every element of a specific type.
*   Selector: `h1`
*   Finds: All `<h1>` tags.

### 2. Class Selectors (The Workhorse)
Targets elements with a specific "class" attribute. Always start with a dot (`.`).
*   Selector: `.price`
*   Finds: Any tag like `<p class="price">` or `<span class="price">`.

### 3. ID Selectors
Targets a single, unique element. Always start with a hash (`#`).
*   Selector: `#main-title`
*   Finds: The tag with `id="main-title"`.

### 4. Attribute Selectors
Targets elements based on any attribute they have.
*   Selector: `a[href]` (Finds all links)
*   Selector: `img[src*="logo"]` (Finds images with "logo" in the source URL)

### 5. Combining Selectors
You can chain these together for laser precision.
*   Selector: `div.content p.description`
*   Finds: All paragraphs with the class `description` that are *inside* a div with the class `content`.

## XPath Fundamentals

XPath (XML Path Language) is older and a bit more complex than CSS, but it is incredibly powerful. While CSS can only look "down" the tree (from parent to child), XPath can look "up" (from child to parent) and even "sideways" (from sibling to sibling).

### 1. The Basics
*   `//` : Look anywhere in the document.
*   `/` : Look at the immediate child.
*   `@` : Target an attribute.

### 2. XPath Examples
*   `//h1` : Find every `<h1>` tag.
*   `//div[@class="price"]` : Find every `<div>` with the class "price".
*   `//a/@href` : Direct access to the link URL attribute.

> [!CAUTION]
> **Scrapy Doc Gap: The // vs ./ Trap**
> This is a bug that even experienced scrapers make. When you are inside a loop (like `for quote in response.xpath('//div[@class="quote"]')`), if you want to find something *inside* that quote, you **must** start your XPath with a dot: `.//`. 
> 
> If you use `//span` inside the loop, XPath will ignore the current quote and search the *entire document* from the beginning. This results in your scraper returning the same first item over and over again. Always use the dot (`.`) when searching inside a nested element!

### 3. The Superpowers of XPath
XPath can find elements based on the **text they contain**, which CSS cannot do.
*   `//p[contains(text(), "Price:")]` : Find the paragraph that mentions the word "Price:".
*   `//h2/following-sibling::p` : Find the paragraph that comes immediately after an `<h2>`.

## CSS vs. XPath: Which to Use?

| Feature | CSS Selectors | XPath |
| :--- | :--- | :--- |
| **Ease of Reading** | ✅ Very easy | ❌ Can be complex |
| **Speed** | ✅ Very fast | ⚠️ Slightly slower |
| **Text Matching** | ❌ Impossible | ✅ Easy (`contains`) |
| **Parent/Sibling Navigation**| ❌ Limited | ✅ Full power |

**My Rule of Thumb:** Start with CSS. It's cleaner and faster. Only switch to XPath if you need to find an element by its text or if you need to climb "up" the tree.

## Scrapy Shell for Testing Selectors

Never guess in your spider file. Always test in the shell.

```bash
scrapy shell "https://quotes.toscrape.com"
```

Try these commands:
```python
# CSS
response.css('small.author::text').get()

# XPath
response.xpath('//small[@class="author"]/text()').get()
```

## Common Selector Patterns

*   **Extracting a link:** `response.css('a::attr(href)').get()`
*   **Extracting an image source:** `response.css('img::attr(src)').get()`
*   **Extracting the first item in a list:** `response.css('li:first-child::text').get()`
*   **Handling missing elements:** Always check if your `.get()` returned `None` before you try to process the data!

## Debugging Selector Issues

If your selector works in the browser but fails in Scrapy:
1.  **Check for JavaScript:** Use `view(response)` in the shell. If the data is missing, the site is dynamic.
2.  **Check for nested tags:** Sometimes a browser "fixes" broken HTML, but Scrapy sees the raw (broken) version. Use the shell to explore the structure.
3.  **Check your dots and hashes:** It's the most common mistake! Dot for class, hash for ID.

## Advanced CSS Selectors

### Pseudo-Selectors: The Power Tools

CSS has powerful pseudo-selectors that most tutorials skip:

**:nth-child() - Select by position**
```python
# Get the first product
response.css('div.product:nth-child(1)').get()

# Get every other product (odd)
response.css('div.product:nth-child(odd)').getall()

# Get every 3rd product
response.css('div.product:nth-child(3n)').getall()

# Get the last product
response.css('div.product:last-child').get()
```

**:not() - Exclude elements**
```python
# Get all links except external ones
response.css('a:not([href^="http"])').getall()

# Get all divs except those with class 'ad'
response.css('div:not(.ad)').getall()
```

**:contains() - Text matching** (XPath only in standard CSS, but Scrapy supports it)
```python
# Find elements containing specific text
response.css('a:contains("Next")').get()
```

**Attribute selectors - Advanced matching**
```python
# Starts with
response.css('a[href^="/products/"]').getall()  # Links starting with /products/

# Ends with
response.css('img[src$=".jpg"]').getall()  # All JPG images

# Contains
response.css('div[class*="product"]').getall()  # Class contains 'product'

# Exact match
response.css('input[type="submit"]').get()  # Exact type match
```

### Combining Selectors

```python
# Multiple classes (AND)
response.css('div.product.featured').getall()  # Has BOTH classes

# Multiple selectors (OR)
response.css('h1, h2, h3').getall()  # Any heading

# Child combinator
response.css('div.container > p').getall()  # Direct children only

# Descendant combinator
response.css('div.container p').getall()  # Any descendant

# Adjacent sibling
response.css('h2 + p').getall()  # <p> immediately after <h2>

# General sibling
response.css('h2 ~ p').getall()  # All <p> siblings after <h2>
```

> [!TIP]
> **Pro Pattern: Chaining Selectors**
> Instead of one complex selector, chain simple ones for readability:
> ```python
> # Hard to read
> price = response.css('div.product:nth-child(1) > div.details > span.price::text').get()
> 
> # Easier to debug
> product = response.css('div.product:nth-child(1)')
> details = product.css('div.details')
> price = details.css('span.price::text').get()
> ```

## XPath: The Complete Guide

### XPath Axes (Navigation)

XPath has powerful navigation axes that CSS can't match:

**ancestor** - Go up the tree
```python
# Find the parent div of a specific span
response.xpath('//span[@class="price"]/ancestor::div[1]').get()

# Find any ancestor with class 'product'
response.xpath('//span[@class="price"]/ancestor::div[@class="product"]').get()
```

**following-sibling** - Next siblings
```python
# Get all <p> tags after an <h2>
response.xpath('//h2/following-sibling::p').getall()

# Get the next <div> sibling
response.xpath('//h2/following-sibling::div[1]').get()
```

**preceding-sibling** - Previous siblings
```python
# Get the <h2> before a <div>
response.xpath('//div[@class="content"]/preceding-sibling::h2[1]').get()
```

**parent** - Direct parent
```python
# Get the parent of a link
response.xpath('//a[@class="buy-now"]/parent::div').get()
```

### XPath Functions

**text() functions**
```python
# Contains text (case-sensitive)
response.xpath('//a[contains(text(), "Next")]').get()

# Starts with
response.xpath('//div[starts-with(@class, "product-")]').getall()

# Normalize space (removes extra whitespace)
response.xpath('//p[normalize-space(text())="Hello"]').get()
```

**String manipulation**
```python
# Concatenate
response.xpath('concat(//span[@class="first-name"]/text(), " ", //span[@class="last-name"]/text())').get()

# Substring
response.xpath('substring(//span[@class="price"]/text(), 2)').get()  # Remove first char ($)

# String length
response.xpath('string-length(//title/text())').get()
```

**Counting**
```python
# Count elements
response.xpath('count(//div[@class="product"])').get()  # Returns number

# Position
response.xpath('//div[@class="product"][position() < 4]').getall()  # First 3
```

**Boolean logic**
```python
# AND
response.xpath('//div[@class="product" and @data-available="true"]').getall()

# OR
response.xpath('//div[@class="product" or @class="item"]').getall()

# NOT
response.xpath('//div[not(@class="ad")]').getall()
```

### XPath vs CSS: When to Use Each

**Use CSS when:**
- Selecting by class or ID
- Simple descendant selection
- You're familiar with CSS
- Performance matters (CSS is slightly faster)

**Use XPath when:**
- You need to go UP the tree (parent, ancestor)
- You need text matching (contains, starts-with)
- You need complex boolean logic
- You need to select by position relative to siblings

**Example where XPath wins:**
```html
<div>
  <span class="label">Price:</span>
  <span class="value">$19.99</span>
</div>
```

```python
# CSS: Can't easily get the price based on the label
# You'd have to get the label, then navigate in Python

# XPath: Easy!
price = response.xpath('//span[text()="Price:"]/following-sibling::span/text()').get()
# Result: "$19.99"
```

## Parsel: Scrapy's Selector Library

Scrapy uses Parsel under the hood. Understanding it helps you debug and optimize.

### What is Parsel?

Parsel is a standalone library that Scrapy uses for selecting elements:
- Wraps lxml for HTML/XML parsing
- Provides unified interface for CSS and XPath
- Handles encoding automatically

```python
# You can use Parsel independently
from parsel import Selector

html = '<div class="product"><span class="price">$19.99</span></div>'
selector = Selector(text=html)
price = selector.css('span.price::text').get()
```

### Selector Methods

**get() vs getall()**
```python
# get() - Returns first match or None
price = response.css('span.price::text').get()
if price:
    print(price)

# get(default) - Returns default if no match
price = response.css('span.price::text').get(default='N/A')

# getall() - Returns list of all matches
prices = response.css('span.price::text').getall()
for price in prices:
    print(price)
```

**extract() vs extract_first()** (deprecated, use get/getall)
```python
# Old way (still works but deprecated)
price = response.css('span.price::text').extract_first()
prices = response.css('span.price::text').extract()

# New way (preferred)
price = response.css('span.price::text').get()
prices = response.css('span.price::text').getall()
```

**re() - Regular expressions**
```python
# Extract numbers from text
price_text = "Price: $19.99"
price = response.css('span.price::text').re_first(r'\d+\.\d+')
# Result: "19.99"

# Extract all numbers
numbers = response.css('div.stats::text').re(r'\d+')
# Result: ['10', '20', '30']
```

**attrib - Direct attribute access**
```python
# Get attribute dictionary
link = response.css('a.product-link')[0]
href = link.attrib['href']
class_name = link.attrib.get('class', '')
```

> [!IMPORTANT]
> **Scrapy Doc Gap: Selector Performance**
> CSS selectors are compiled and cached, but XPath expressions are not. For repeated selections in loops:
> 
> ```python
> # SLOW - XPath compiled on every iteration
> for row in response.css('tr'):
>     price = row.xpath('.//td[@class="price"]/text()').get()
> 
> # FASTER - Use CSS when possible
> for row in response.css('tr'):
>     price = row.css('td.price::text').get()
> 
> # FASTEST - Pre-compile XPath (advanced)
> from lxml import etree
> price_xpath = etree.XPath('.//td[@class="price"]/text()')
> for row in response.css('tr'):
>     price = price_xpath(row.root)[0] if price_xpath(row.root) else None
> ```

### Selector Performance Tips

**1. Be specific**
```python
# SLOW - searches entire document
response.css('span').getall()

# FAST - limits search scope
response.css('div.products span').getall()
```

**2. Use IDs when available**
```python
# FASTEST - ID lookup is O(1)
response.css('#product-123').get()

# SLOWER - class lookup is O(n)
response.css('.product').get()
```

**3. Limit depth**
```python
# SLOW - searches all descendants
response.css('div.container span.price').getall()

# FASTER - direct child only
response.css('div.container > div.product > span.price').getall()
```

**4. Reuse selectors**
```python
# SLOW - parses product multiple times
for product in response.css('div.product'):
    title = response.css('div.product h2::text').get()
    price = response.css('div.product span.price::text').get()

# FAST - reuse product selector
for product in response.css('div.product'):
    title = product.css('h2::text').get()
    price = product.css('span.price::text').get()
```

## Chapter Summary

**What we covered:**
- Selectors are the "addresses" we use to find data in HTML.
- **CSS Selectors** are perfect for 90% of tasks because they are clean and fast.
- **XPath** is our "secret weapon" for complex navigation and text-based searching.
- Use the Scrapy Shell as your playground to perfect your selectors.
- Specificity is the key to building scrapers that don't break.

**Remember:**
- Don't copy the "Full XPath" from Chrome; it's too fragile.
- Patterns are your friend. Find a class name that repeats.
- If it's hard to find, look at the parent element.

## Selector Challenge (Exercise)

1.  Open the Scrapy Shell for `https://quotes.toscrape.com`.
2.  Write a **CSS Selector** to get the text of the second quote on the page.
3.  Write an **XPath** to find the author who has the name "Albert Einstein" and get their "About" link.
4.  Write a selector to get all the tags for the first quote and save them as a list.

**Previous chapter:**
[Chapter 5: Your First Scrapy Spider](./chapter_05_your_first_scrapy_spider.md)

**Next chapter:**
You now know how to find any piece of data. This marks the end of **Part I: Getting Started**. You are no longer a beginner! You have the foundation. In the next part, **Extracting and Cleaning Data**, we're going to learn how to take those raw strings and turn them into professional, clean datasets.

---

**End of Chapter 6**
