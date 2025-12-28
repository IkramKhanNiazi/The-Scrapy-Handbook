# Chapter 34: Downloader Middlewares

Think about the first time you face "The Wall." You're trying to scrape a high-security fashion retailer. You've got your selectors ready, your pipeline is perfect, and you've got your virtual environment set up. But every time you run your spider, you get the exact same response: `403 Forbidden`.

You might try everything you know. You change the User-Agent. You add random delays. You even try running it at 4:00 AM. Nothing works. It can feel as if you're shouting at a closed door, and the person on the other side is just ignoring you.

You might spend three days in a state of growing obsession. You'll realize that the website isn't just "blocking" you; it's "recognizing" you. It knows that even though your User-Agent says you're Chrome, your "Request Headers" are missing ten other tiny pieces of information that a real Chrome browser always sends. You're like a spy wearing a tuxedo but forgetting his passport.

You need a way to change your "Passport" for every single request. You need to add cookies, modify headers, and rotate your IP address all without cluttering your spider code.

That's when you discover **Downloader Middlewares**.

It's like becoming a master of disguise. You'll realize that you can build a "Wardrobe" where every outgoing request can stop, put on a new hat, a new pair of glasses, and a new ID card before heading out to the internet. Suddenly, the `403 Forbidden` messages vanish. You're not just a scraper anymore; you're a digital chameleon.

In this chapter, you're going to learn how to build that wardrobe.

---

## Introduction

In the previous chapter, we explored the "Spider Middleware," which sits between the Engine and your code. Now, we're going a level deeper.

**Downloader Middlewares** sit between the Engine and the Internet. They are the "Gatekeepers" of your requests. They decide how Scrapy talks to the outside world. In this chapter, we're going to master the two most important methods in Scrapy's internal lifecycle: `process_request` and `process_response`. We'll also build a professional-grade User-Agent rotator and learn how to handle complex proxy logic globally.

## The Downloader Lifecycle: The "Gatekeeper"

Think of the Downloader Middleware as a checkpoint.
1.  **Request Phase:** A request leaves the Engine. It must pass through the middleware "checkpoint" on its way to the internet. This is where you can add headers, change the IP, or even cancel the request entirely.
2.  **Response Phase:** The website sends data back. It must pass through the middleware again on its way to the Engine. This is where you can check for bans, handle redirects, or decompress data.

## Key Methods: `process_request` and `process_response`

### 1. `process_request(request, spider)`
This runs for every request *before* it's sent to the internet.
```python
def process_request(self, request, spider):
    # Add a custom header to every single request
    request.headers['X-Custom-Scraper'] = 'MySecretID'
    return None # Continue to the next middleware or the internet
```

### 2. `process_response(request, response, spider)`
This runs *after* the page is downloaded but *before* it reaches your spider.
```python
def process_response(self, request, response, spider):
    if response.status == 403:
        spider.logger.warning("WE ARE BANNED! Changing strategy...")
        # You could return a new Request here to try again!
    return response

### The 3 Return Values (Critical Knowledge)
What you return from `process_request` determines the fate of the request:
1. `None`: Continue to the next middleware (and eventually, the Internet). **(Standard)**
2. `Response` object: STOP! Don't go to the internet. Return this data to the spider immediately.
   * *Use Case:* Loading a cached page from a database instead of making a network call.
3. `Request` object: STOP! Discard the current request and schedule this *new* one instead.
   * *Use Case:* You detect a "Login Required" cookie is missing, so you redirect the spider to the Login Page first.
```

## Real-World Use Case: User-Agent Rotation

While there are libraries that do this, writing your own rotator is the best way to understand middlewares.

```python
import random

class MyCustomUAMiddleware:
    USER_AGENTS = [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x4) ...',
        'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...',
    ]

    def process_request(self, request, spider):
        # Pick a random UA for this request
        request.headers['User-Agent'] = random.choice(self.USER_AGENTS)
```

## Professional Proxy Management

Managing proxies inside your spider's `parse` method is a recipe for disaster. Using a middleware allows you to handle proxy "Rotation" and "Authentication" in one central place.

```python
def process_request(self, request, spider):
    # Route every request through our proxy server
    request.meta['proxy'] = "http://username:password@my-proxy-server:8080"
```

## Handling Redirects and Retries (The "Loop" Trap)

Sometimes you want to handle redirects manually, especially for login flows.
If you set `meta={'dont_redirect': True}`, Scrapy gives you the `302` response.

**Pro Pattern: Manual Redirect Handling**
```python
def process_response(self, request, response, spider):
    if response.status in [301, 302] and 'login' in response.headers['Location']:
        # The site is redirecting us to login. 
        # Intercept it and send a LOGIN request instead!
        pass # Rewrite logic here
    return response
```

### The Middleware Stack Order (Priority)
Priority numbers (`543`, `400`) are confusing.
- **process_request**: Runs Low -> High (0 to 1000). 0 is closest to Engine.
- **process_response**: Runs High -> Low (1000 to 0). 1000 is closest to Internet.

*Think of it like an onion.* Request cuts *out* from the center. Response cuts *in* from the skin.
If you want to modify the Proxy (Internet-side), put your middleware at **700-800**.
If you want to modify the User-Agent (Engine-side), put it at **400-500**.

Scrapy includes its own "Built-in" middlewares for handling redirects and retries. You can see them in action in your logs. By writing your own Downloader Middleware, you can "supercharge" these systems for example, by telling Scrapy to only retry a `404` error if it comes from a specific domain.

> [!CAUTION]
> **Scrapy Doc Gap: Middleware Priority and Retries**
> When a request fails and the `RetryMiddleware` puts it back in the queue, that request travels through the entire middleware stack **again**. 
> 
> If your middleware adds a unique ID or a timestamp to a request, it might end up with *two* IDs if you aren't careful. Always check if your logic has already been applied by using a flag in `request.meta`. This ensures your requests stay clean and efficient, no matter how many times they are retried.

## Enabling the Middleware

In your `settings.py`:
```python
# settings.py
DOWNLOADER_MIDDLEWARES = {
    'my_project.middlewares.MyCustomUAMiddleware': 400,
    # (Higher number = further from the Engine, closer to the Internet)
}
```

## Chapter Summary

**What we covered:**
- Downloader Middlewares govern the conversation between Scrapy and the Internet.
- `process_request` is the place to add headers, rotating agents, and proxies.
- `process_response` is the place to identify bans and handle complex server signals.
- Middlewares keep your spider code focused on "Data," while the middleware handles the "Details" of communication.
- Proper use of Downloader Middlewares is what separates "scripts" from "production systems."

**Key code:**
```python
request.headers['User-Agent'] = '...'
request.meta['proxy'] = '...'
```

**Previous chapter:**
[Chapter 33: Spider Middlewares](./chapter_33_spider_middlewares.md)

**Next chapter:**
We've mastered the middlewares. But what if you want to add a feature to Scrapy that doesn't just look at requests, but follows the health of the entire crawl? In the next chapter, we're going into **Scrapy Extensions** learning how to build custom add-ons for the engine itself.

---

**End of Chapter 34**
