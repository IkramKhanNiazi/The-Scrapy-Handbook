# Chapter 3: How Websites Work

Think about the first time you try to scrape a modern news website. You've spent hours studying HTML tags, and you feel confident. You find the `<h1>` tag with the headline, the `<p>` tags with the article text, and everything seems perfect.

But when you run your script, it returns... nothing. Or rather, it returns a page that looks like a skeleton. The headlines are missing. The images are gone. Even the menus are just empty bullet points.

You might be baffled. You think, "I can see the text in my browser! Why can't my code see it?"

You might spend days trying to "fix" your CSS selectors, thinking you're just typing them wrong. You try every combination of dots, hashes, and brackets you can find. It's frustrating enough to make you want to throw your laptop across the room. It feels like the website is playing a game of hide-and-seek with you, and you're losing.

Then, you learn the secret. You press `F12` on your keyboard, click on the "Network" tab, and hit refresh.

"Look at that," the data reveals. "The website you're looking at is empty when it first loads. It uses JavaScript to go out and 'fetch' the news from a different server after the page is already open. Your script is grabbing the empty box before the news arrives."

It's like someone has suddenly turned on the lights in a dark room. You realize that a website isn't just a static document; it's a living, breathing conversation between your computer and a server. If you don't know how that conversation works, you'll always be chasing ghosts.

In this chapter, you're going to learn the language of that conversation.

---

## Introduction

To be a master scraper, you need to understand the "invisible" world behind the colors and fonts. You need to understand how data travels from a server thousands of miles away to your computer screen.

In this chapter, we're going to break down the technical "DNA" of a website. We'll explore the language of the web (HTTP), the skeleton of the page (HTML), the makeup that makes it look good (CSS), and the brain that makes it interactive (JavaScript). We'll also master the most important tool in your arsenal: the Browser Developer Tools.

## Understanding HTTP: The Language of the Web

When you type a website address into your browser and hit Enter, your computer sends a "request" to a server somewhere in the world. That server sends back a "response" (the HTML, images, and other files that make up the page).

This conversation happens using a protocol called **HTTP** (HyperText Transfer Protocol). It's the language that browsers and servers use to talk to each other.

### HTTP Versions: 1.1, 2, and 3

Most tutorials only teach HTTP/1.1, but modern websites use newer versions:

**HTTP/1.1** (1997-present)
- One request per connection (or sequential requests)
- Text-based protocol
- Still widely used

**HTTP/2** (2015-present)
- Multiplexing: multiple requests over one connection simultaneously
- Binary protocol (faster)
- Server push: server can send resources before you ask
- Used by 50%+ of websites in 2026

**HTTP/3** (2022-present)
- Uses QUIC instead of TCP
- Faster connection establishment
- Better performance on unreliable networks
- Growing adoption

> [!NOTE]
> **Scrapy and HTTP/2**
> As of 2026, Scrapy primarily uses HTTP/1.1 via Twisted. While this works fine for scraping, it means you won't benefit from HTTP/2's multiplexing. For most scraping tasks, this doesn't matter. If you need HTTP/2 support, consider using `scrapy-playwright` or `httpx`.

### HTTP Request Anatomy

A request is like sending a letter. It has:

**1. Method**: What you want to do
- **GET**: Retrieve data (most common for scraping)
- **POST**: Send data (forms, login)
- **HEAD**: Get headers only (check if resource exists without downloading)
- **PUT/DELETE**: Modify/remove resources (rare in scraping)

**2. URL**: The address of the resource
```
https://example.com/products/123?sort=price&filter=new
```

**3. Headers**: Metadata about the request
```
GET /products/123 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (compatible; MyBot/1.0)
Accept: text/html,application/json
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Referer: https://example.com/
Cookie: session_id=abc123
```

### Critical Request Headers for Scraping

**User-Agent**: Identifies your client
- Default Scrapy: `Scrapy/2.11 (+https://scrapy.org)`
- Better: `Mozilla/5.0 (compatible; YourBot/1.0; +https://yoursite.com)`
- Anti-bot systems check this heavily

**Accept**: What content types you'll accept
- `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`
- Tells server you prefer HTML but will accept anything

**Accept-Language**: Your language preference
- `en-US,en;q=0.9`
- Some sites serve different content based on this

**Accept-Encoding**: Compression formats you support
- `gzip, deflate, br`
- Scrapy handles decompression automatically

**Referer**: Where you came from
- Some sites check this to prevent direct linking
- Scrapy can set this automatically

**Cookie**: Session data
- Scrapy's CookiesMiddleware handles this automatically
- Critical for maintaining sessions

**Origin**: For CORS requests
- Only matters for browsers, not Scrapy
- But some sites check it anyway

### HTTP Response Anatomy

A response is the server's reply. It includes:
**1. Status Code**: Did it work?
```
HTTP/1.1 200 OK
```

**2. Headers**: Metadata about the response
```
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 15234
Set-Cookie: session_id=xyz789; Path=/; HttpOnly
Cache-Control: max-age=3600
X-RateLimit-Remaining: 99
```

**3. Body**: The actual content (HTML, JSON, etc.)

### Critical Response Headers for Scraping

**Content-Type**: What kind of data this is
- `text/html; charset=UTF-8` - HTML page
- `application/json` - JSON data
- `application/xml` - XML data
- Scrapy uses this to decode the response correctly

**Content-Encoding**: How the data is compressed
- `gzip`, `deflate`, `br` (Brotli)
- Scrapy decompresses automatically

**Set-Cookie**: Server wants to store data in your browser
- Scrapy's CookiesMiddleware handles this
- Critical for sessions and authentication

**Cache-Control**: How long to cache this response
- `max-age=3600` - cache for 1 hour
- `no-cache` - don't cache
- Useful for understanding site update frequency

**X-RateLimit-*** headers: API rate limiting info
- `X-RateLimit-Limit: 100`
- `X-RateLimit-Remaining: 95`
- `X-RateLimit-Reset: 1609459200`
- Watch these to avoid getting banned

### HTTP Status Codes: The Full Story

Status codes tell you what happened. Here are the ones that matter for scraping:

**2xx Success**
- **200 OK**: Perfect, got the data
- **201 Created**: Resource was created (POST requests)
- **204 No Content**: Success but no data to return

**3xx Redirection**
- **301 Moved Permanently**: Resource moved forever (update your URLs)
- **302 Found**: Temporary redirect
- **304 Not Modified**: Cached version is still good
- Scrapy follows redirects automatically (up to `REDIRECT_MAX_TIMES`)

**4xx Client Errors**
- **400 Bad Request**: Your request was malformed
- **401 Unauthorized**: Need authentication
- **403 Forbidden**: You're not allowed (might be blocked)
- **404 Not Found**: Page doesn't exist
- **429 Too Many Requests**: You're being rate limited (SLOW DOWN!)

**5xx Server Errors**
- **500 Internal Server Error**: Server crashed
- **502 Bad Gateway**: Proxy/load balancer issue
- **503 Service Unavailable**: Server overloaded or maintenance
- **504 Gateway Timeout**: Server took too long

> [!IMPORTANT]
> **Scrapy Doc Gap: How Scrapy Handles Status Codes**
> By default, Scrapy only processes responses with status codes 200-299. If you get a 404 or 500, your parse method won't be called unless you explicitly handle it.
> 
> To handle error responses:
> ```python
> class MySpider(scrapy.Spider):
>     handle_httpstatus_list = [404, 403]  # Process these too
> ```
> 
> Or per-request:
> ```python
> yield scrapy.Request(url, meta={'handle_httpstatus_list': [404]})
> ```

## HTML Structure: The Skeleton

HTML (HyperText Markup Language) is how we define the *structure* of a page. It uses **tags** to label different types of content.

A tag looks like this: `<h1>My Headline</h1>`.
*   `<h1>` is the opening tag.
*   `</h1>` is the closing tag.
*   The text in the middle is the **content**.

### Tags have "Attributes"
Sometimes tags need more info.
`<a href="https://example.com" class="main-link">Click me</a>`
*   `href` is an attribute that tells the link where to go.
*   `class` is an attribute used for styling (and it's a scraper's best friend for finding data).

### The DOM (Document Object Model)
Think of the HTML as a family tree.
*   `<html>` is the grandparent.
*   `<body>` is the parent.
*   `<div>`, `<h1>`, and `<p>` are the children.
As scrapers, we use this tree structure to navigate. We might say, "Find the second `<div>` inside the `<body>` and grab the text from its `<h1>` child."

## CSS: Making Websites Pretty

If HTML is the skeleton, CSS (Cascading Style Sheets) is the skin and clothes. It defines how things look.

Scrapers care about CSS because designers use **Classes** and **IDs** to apply styles.
*   A **Class** (like `.product-price`) is used for many elements.
*   An **ID** (like `#main-title`) is meant to be used for only ONE element on the page.

In Scrapy, we use "CSS Selectors" to find our data. It's like a search query: "Find every element with the class `product-price`."

## JavaScript: The Dynamic Web

This is where things get tricky. In the old days, all the data was right there in the HTML. Today, many websites are "Dynamic."

When you load a page, the initial HTML might be almost empty. Then, a **JavaScript** script runs in your browser. It reaches out to a different server, grabs some data (often in a format called **JSON**), and "injects" it into the page while you are watching.

For a scraper, this is a challenge. If our script only looks at the initial HTML, it will miss all the dynamic data. In later chapters, we'll learn how to "see" what the JavaScript is doing and how to scrape it directly.

## Browser DevTools: Your Best Friend

You don't have to guess what's happening. Every modern browser has built-in Power Tools. Just right-click anywhere on a page and select **"Inspect"**.

### The Elements Tab
This shows you the "live" HTML of the page. You can click on an element to see its classes, IDs, and position in the family tree.

### The Network Tab
This is the most important tab for advanced scraping. It shows every single request your browser makes. If a website is using JavaScript to load data, you can see that data traveling in the Network tab! You can see exactly where it's coming from and what it looks like.

### The Console Tab
This is like a playground where you can run your own JavaScript code or see error messages from the website.

> [!WARNING]
> **Scrapy Doc Gap: "Inspect" vs. "View Source"**
> This is perhaps the most common trap for Scrapy beginners. When you use the **"Inspect"** tool in your browser, you are looking at the *rendered* version of the page   the version after JavaScript has added or changed things. 
> 
> Scrapy, by default, sees the **"View Source"** version   the raw HTML that comes straight from the server before any JavaScript runs. If you copy a CSS selector from the "Inspect" tab and it doesn't work in Scrapy, check "View Source" (Right-click -> View Page Source). If the data isn't there, you'll need the advanced techniques we cover in Chapter 14.

## Common Website Patterns

As you scrape more, you'll notice patterns:
*   **Lists:** Items are often wrapped in a `<ul>` (unordered list) or a series of identical `<div>` boxes.
*   **Pagination:** A "Next" button usually lives at the bottom in a `<li>` tag with a class like `.next`.
*   **Grids:** Product pages often use a "Grid" layout where each item has the same class names.

## Cookies: Session Management Deep Dive

Cookies are how websites "remember" you between requests. Understanding them is critical for scraping authenticated content.

### Cookie Anatomy

```
Set-Cookie: session_id=abc123; Path=/; Domain=.example.com; Expires=Wed, 21 Oct 2026 07:28:00 GMT; Secure; HttpOnly; SameSite=Strict
```

**Key Attributes:**

**Name=Value**: The actual data
- `session_id=abc123`

**Domain**: Which domains can access this cookie
- `.example.com` - accessible by example.com and all subdomains
- `www.example.com` - only accessible by www subdomain

**Path**: Which URL paths can access this cookie
- `/` - all paths (most common)
- `/admin/` - only admin section

**Expires/Max-Age**: When the cookie expires
- Session cookies: no expiration (deleted when browser closes)
- Persistent cookies: have expiration date

**Secure**: Only send over HTTPS
- Critical for security
- Scrapy respects this automatically

**HttpOnly**: Not accessible via JavaScript
- Prevents XSS attacks
- Doesn't affect Scrapy (we're not JavaScript)

**SameSite**: CSRF protection
- `Strict` - only send on same-site requests
- `Lax` - send on top-level navigation
- `None` - always send (requires Secure)

### How Scrapy Handles Cookies

Scrapy's `CookiesMiddleware` automatically:
1. Stores cookies from `Set-Cookie` headers
2. Sends appropriate cookies with each request
3. Respects Domain, Path, and Secure attributes
4. Handles cookie expiration

> [!TIP]
> **Scrapy Doc Gap: Cookie Debugging**
> To see what cookies Scrapy is managing:
> ```python
> def parse(self, response):
>     cookies = response.request.headers.getlist('Cookie')
>     self.logger.info(f"Cookies sent: {cookies}")
>     
>     # Or access the cookie jar directly
>     from scrapy.http.cookies import CookieJar
>     # Cookies are stored per-domain in the middleware
> ```
> 
> Enable cookie debugging:
> ```python
> # settings.py
> COOKIES_DEBUG = True  # Logs all cookie operations
> ```

### Manual Cookie Management

Sometimes you need to set cookies manually:

```python
# Set cookies for a specific request
yield scrapy.Request(
    url='https://example.com/protected',
    cookies={'session_id': 'abc123', 'user_pref': 'dark_mode'}
)

# Or set cookies for the entire spider
class MySpider(scrapy.Spider):
    def start_requests(self):
        cookies = {'auth_token': 'xyz789'}
        for url in self.start_urls:
            yield scrapy.Request(url, cookies=cookies)
```

## HTTPS and TLS: Secure Connections

When you see `https://` instead of `http://`, the connection is encrypted using TLS (Transport Layer Security).

### Why HTTPS Matters for Scraping

**Certificate Verification**
- Scrapy verifies SSL certificates by default
- If a site has an invalid certificate, requests will fail
- Common in development/staging environments

**Disabling Certificate Verification** (development only!):
```python
# settings.py - NEVER use in production!
DOWNLOADER_CLIENT_TLS_METHOD = 'TLS'
DOWNLOADER_CLIENT_TLS_VERIFY = False
```

**SNI (Server Name Indication)**
- Allows multiple HTTPS sites on one IP
- Scrapy supports this automatically
- Older Python versions had issues (pre-2.7.9)

### TLS Fingerprinting

Advanced anti-bot systems can detect scrapers by their TLS handshake:
- Cipher suites offered
- TLS extensions
- Handshake order

Scrapy's TLS fingerprint is different from browsers. If you're being blocked despite correct headers, TLS fingerprinting might be the cause.

**Solution**: Use `scrapy-playwright` or `selenium` for browser-identical TLS.

## Scrapy's HTTP Client: Twisted Under the Hood

Understanding Scrapy's HTTP implementation helps you debug issues and optimize performance.

### The Twisted Reactor

Scrapy uses Twisted, an asynchronous networking framework:
- **Non-blocking I/O**: Can handle thousands of concurrent requests
- **Event-driven**: Callbacks fire when responses arrive
- **Single-threaded**: All your spider code runs in one thread

> [!IMPORTANT]
> **Scrapy Doc Gap: The Reactor and Blocking Code**
> Never use blocking operations in your spider:
> ```python
> # BAD - blocks the entire reactor!
> import time
> time.sleep(5)  # This freezes ALL spiders
> 
> # BAD - synchronous database query
> result = db.query("SELECT * FROM products")  # Blocks reactor
> 
> # GOOD - use Twisted's async sleep
> from twisted.internet import defer, task
> yield task.deferLater(reactor, 5, lambda: None)
> 
> # GOOD - use async database drivers
> result = yield db.async_query("SELECT * FROM products")
> ```
> 
> Blocking the reactor causes:
> - All requests pause
> - Timeouts on other requests
> - Terrible performance

### Connection Pooling

Scrapy maintains persistent connections to servers:

```python
# settings.py
REACTOR_THREADPOOL_MAXSIZE = 20  # Max threads for DNS lookups
CONCURRENT_REQUESTS_PER_DOMAIN = 8  # Connections per domain
DOWNLOAD_TIMEOUT = 180  # Seconds before timeout
```

**How it works:**
1. First request to a domain opens a connection
2. Subsequent requests reuse that connection (HTTP keep-alive)
3. Connections close after idle timeout
4. Reduces latency and server load

### DNS Caching

Scrapy caches DNS lookups:

```python
# settings.py  
DNSCACHE_ENABLED = True  # Default
DNSCACHE_SIZE = 10000  # Number of domains to cache
DNS_TIMEOUT = 60  # DNS lookup timeout
```

For distributed crawling, consider external DNS caching (dnscache, unbound).

## Browser Fingerprinting and Detection

Modern websites use sophisticated techniques to detect bots. Understanding these helps you scrape responsibly without triggering blocks.

### Common Detection Methods

**1. User-Agent Analysis**
- Check if User-Agent matches known bots
- Verify User-Agent consistency with other headers
- Scrapy's default User-Agent screams "I'm a bot!"

**2. Header Fingerprinting**
- Browsers send headers in a specific order
- Scrapy's header order differs from browsers
- Missing headers (Accept-Language, Accept-Encoding) are red flags

**3. TLS Fingerprinting** (mentioned above)
- JA3 fingerprint of TLS handshake
- Scrapy's fingerprint is unique

**4. Behavioral Analysis**
- Request timing (too fast = bot)
- Navigation patterns (direct URL access without clicking)
- Mouse movements and scrolling (only browsers do this)

**5. JavaScript Challenges**
- Render JavaScript to prove you're a browser
- Scrapy can't execute JavaScript natively

### Making Scrapy Look More Like a Browser

```python
# settings.py
USER_AGENT = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'

DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.9',
    'Accept-Encoding': 'gzip, deflate, br',
    'DNT': '1',
    'Connection': 'keep-alive',
    'Upgrade-Insecure-Requests': '1',
}
```

> [!WARNING]
> **The Arms Race**
> Anti-bot technology evolves constantly. What works today might not work tomorrow. The best defense is:
> 1. Scrape politely (slow requests, respect robots.txt)
> 2. Identify yourself honestly when possible
> 3. Use official APIs when available
> 4. For heavy anti-bot sites, consider browser automation (Playwright/Selenium)

## Chapter Summary

**What we covered:**
- Websites are a conversation of Requests and Responses via HTTP.
- Status codes like `200` (success) and `404` (missing) tell us the result of our request.
- HTML provides the structure (tags and attributes) and CSS providing the styling (classes and IDs).
- JavaScript makes pages dynamic by loading data after the initial page load.
- Browser DevTools are the ultimate way to "peek behind the curtain."

**Remember:**
- Always check the "Network" tab if you can't find the data in the HTML.
- Classes (`.`) are for many items; IDs (`#`) are for single items.
- The "Inspect" tool is your X-ray vision.

## Hands-On Exercise

1.  Open your browser and go to `https://quotes.toscrape.com`.
2.  Right-click on the first quote and select **Inspect**. Look at the HTML tags. What class is used for the quote text? What class is used for the author?
3.  Open the **Network** tab in the DevTools and refresh the page. Look at the list of files that load. Can you find the one named `quotes.toscrape.com`? Click it and look at the "Response" sub-tab to see the raw HTML.

**Previous chapter:**
[Chapter 2: Understanding robots.txt](./chapter_02_understanding_robots_txt.md)

**Next chapter:**
The theory is done. It's time to build your "Scrapy Lab." In the next chapter, we're going to set up your environment, install Python, and get Scrapy ready for action on your computer.

---

**End of Chapter 3**
