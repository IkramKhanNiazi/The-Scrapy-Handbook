# Chapter 2: Understanding robots.txt

Imagine the feeling of being a "bad actor" on the internet. You've just built a spider that is crawling a local news website. You're so excited to see the data piling up that you don't notice the server is starting to slow down. Suddenly, your spider stops working. You try to visit the website in your browser, and you can't even load the homepage.

You might panic. Did you crash their entire server? Is the legal department going to send a notice?

You might spend the next hour in a state of pure anxiety, searching for "what happens when you scrape a site too fast." That's when you find a forum post that mentions something called the `robots.txt` file. You go to the news site's address and add `/robots.txt` to the end of the URL.

There it is. A simple, plain-text file that says, in no uncertain terms: `Disallow: /search/`.

It likely feels as if you've been walking through someone's house without checking the signs on the doors. The site owner has clearly asked visitors like you not to crawl their search pages, and you missed it because you didn't even know the sign existed. It's a key moment of realization. Being a good scraper isn't just about writing code that works; it's about being a guest that knows how to follow the house rules.

In this chapter, you're going to learn how to find these house rules and how to make sure your spiders are always the most polite guests at the table.

---

## Introduction

The internet isn't just a free-for-all. Since the early days of the web, there has been a "gentleman's agreement" between website owners and the automated programs that visit them. This agreement is founded on a single file: **robots.txt**.

In this chapter, we're going to demystify this file. We'll learn how to find it, how to read its secret language, and most importantly, how to make Scrapy obey it automatically. We'll also talk about the ethics of what to do when the rules feel a little too strict.

## Production Concerns and Edge Cases

### When robots.txt is Huge

Some sites (looking at you, Amazon) have robots.txt files with 10,000+ lines. This can slow down spider startup significantly.

**Production Pattern**: Cache robots.txt externally
```python
# For distributed crawling, cache robots.txt in Redis
# and reuse across spider instances
```

### When robots.txt is Unreachable

What happens if the robots.txt file returns a 500 error or times out?

**Scrapy's behavior**:
- If robots.txt is unreachable, Scrapy assumes "disallow everything" (safe default)
- You can change this with `ROBOTSTXT_OBEY_NOFOLLOW`

### Dynamic robots.txt (The Trap)

Some sites serve different robots.txt based on your IP or User-Agent. This can cause:
- Your development spider to work fine
- Your production spider (different IP) to be blocked

**Solution**: Always test from your production environment before deploying.

### robots.txt as a Honeypot

Rare but real: some sites list "forbidden" URLs in robots.txt that are actually traps. If you visit them, you get banned.

```text
User-agent: *
Disallow: /trap/
Disallow: /honeypot/
```

If these URLs don't appear anywhere else on the site, they're likely traps. Respecting robots.txt protects you from this.

## The Business Reality: When robots.txt Blocks What You Need

Here's the uncomfortable truth: sometimes robots.txt blocks exactly the data you need. What do you do?

### Option 1: Respect It and Find Alternatives
- Check if there's an API
- Contact the site owner for permission
- Buy the data from a vendor

### Option 2: Assess the Risk
If you decide to proceed despite robots.txt:
- Document your legal assessment
- Understand you're in a gray area
- Be extra polite (slow requests, identify yourself)
- Be prepared to stop if asked

### Option 3: The Ethical Override
Some scenarios where ignoring robots.txt might be justified:
- Government data that should be public
- Academic research with no commercial intent
- Investigative journalism

Even in these cases, be transparent about what you're doing and why.
## What is robots.txt?

Think of `robots.txt` as the "Do Not Disturb" sign for a website. It is a simple text file that website owners put in their root directory to tell web crawlers (like Google's search bot or your Scrapy spider) which parts of their site are okay to visit and which parts are off-limits.

It's important to know that `robots.txt` is not a security feature. It doesn't actually "lock" anything. A malicious scraper can easily ignore it. But for professional and ethical scrapers, it is the first thing we look at before we start a new project.

> [!NOTE]
> **The Robots Exclusion Protocol (RFC 9309)**
> In 2022, robots.txt became an official internet standard: RFC 9309. For 30 years, it was just a "gentleman's agreement," but now it's a formal specification. This matters because it means the rules are standardized, not just conventions. You can read the full spec at [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html), but we'll cover everything you need to know in this chapter.

## Why robots.txt Matters

Why bother reading it?

1.  **Legal Protection:** If you ignore a `robots.txt` file and send too many requests to a "Forbidden" area, a website owner could use that as evidence that you are "trespassing" or acting in bad faith.
2.  **Efficiency:** Often, site owners disallow pages that are useless for scraping (like login pages, checkout counters, or complex search filters). Following the rules actually saves your spider time.
3.  **Reputation:** If you want to be a professional in this field, you need to show that you respect the infrastructure of the web.

## Finding robots.txt Files

Finding the rules is easy. Every standard `robots.txt` file lives at the same place: the root of the domain.

To see the rules for any site, just go to: `https://example.com/robots.txt`

Try it right now with a few big sites like Google or Amazon. You'll see that some are very short, while others are thousands of lines long.

## Reading robots.txt

The file uses a simple "Key: Value" format. Let's look at the four most important directives.

### 1. User-agent Directive
This tells you *who* the rule is for.
```text
User-agent: *
```
The asterisk (`*`) is a wildcard. It means "this rule applies to every robot in the world." If it says `User-agent: Googlebot`, that rule only applies to Google.

### 2. Disallow Directive
This is the "Keep Out" sign.
```text
Disallow: /private/
```
This tells the robot not to visit any URL that starts with `/private/`.

### 3. Allow Directive
Sometimes, a site might disallow a whole folder but "allow" one specific file inside it.
```text
Disallow: /images/
Allow: /images/public/
```

### 4. Crawl-delay Directive
This is the "Slow Down" sign.
```text
Crawl-delay: 10
```
This tells the robot to wait ten seconds between every single request. Scrapy can handle this, but it's one of the rules most likely to be found on smaller websites.

### 5. Sitemap Directive (The Hidden Treasure)
This is one of the most useful parts of robots.txt for scrapers, and it's often overlooked.
```text
Sitemap: https://example.com/sitemap.xml
```

A sitemap is an XML file that lists all the important URLs on a website. If a site provides one, it's like getting a complete map of the territory before you start exploring. We'll use sitemaps extensively in Chapter 16.

> [!TIP]
> **Pro Pattern: Always Check for Sitemaps**
> Before writing complex crawling logic, check the robots.txt for a sitemap. Many sites list thousands of URLs in their sitemap, saving you from having to discover them through link following. This is especially valuable for large e-commerce sites.

### 6. Wildcard Patterns (Advanced)
The robots.txt spec supports wildcards for more precise control:

**Asterisk (`*`)**: Matches any sequence of characters
```text
Disallow: /*.pdf$
```
This blocks all PDF files (the `$` means "end of URL").

**Dollar Sign (`$`)**: Matches the end of the URL
```text
Disallow: /*?
```
This blocks all URLs with query parameters (anything with a `?`).

**Example: Blocking Dynamic URLs**
```text
User-agent: *
Disallow: /*?sort=
Disallow: /*?filter=
Allow: /*?page=
```
This pattern blocks sorting and filtering URLs but allows pagination. This is common on e-commerce sites to prevent crawlers from hitting millions of permutations of the same products.

## Common robots.txt Patterns

You'll see a few common patterns as you explore:

*   **The "Welcome to Everyone" Pattern:**
    ```text
    User-agent: *
    Disallow:
    ```
    (An empty `Disallow` means everything is allowed!)

*   **The "Stay Out of My House" Pattern:**
    ```text
    User-agent: *
    Disallow: /
    ```
    (A single slash means "don't crawl anything on this site").

*   **The "Stay Away From My Search" Pattern:**
    ```text
    User-agent: *
    Disallow: /search/
    Disallow: /query/
    ```

## What If There's No robots.txt?

If you try to go to `/robots.txt` and get a "404 Not Found" error, it means the website has no specific rules. In this case, you are free to crawl the site, but you should still be a "polite" guest and never send too many requests at once.

## Testing with Real Websites

Let's look at the `robots.txt` for `quotes.toscrape.com`.
If you visit `http://quotes.toscrape.com/robots.txt`, you'll see:
```text
User-agent: *
Disallow:
```
This means we are officially welcome to scrape anything on the site!

## How Scrapy Handles robots.txt

One of Scrapy's best features is that it respects `robots.txt` by default. But understanding *how* it does this will help you debug issues and optimize your spiders.

### The ROBOTSTXT_OBEY Setting

In your `settings.py` file, you'll find:
```python
ROBOTSTXT_OBEY = True
```

When this is `True` (which it should be for production), Scrapy:
1. Downloads the robots.txt file once when the spider starts
2. Parses the rules
3. Checks every single URL against those rules before scheduling it
4. Drops any requests that violate the rules

> [!CAUTION]
> **Scrapy Doc Gap: robots.txt is Cached for the Spider's Lifetime**
> Here's something the docs don't emphasize: Scrapy downloads robots.txt once at spider startup and caches it in memory for the entire run. If you're running a long-lived spider (hours or days) and the site updates their robots.txt, you won't see the changes until you restart.
> 
> For production systems that run continuously, consider implementing a periodic robots.txt refresh mechanism or restart spiders daily.

### The RobotsTxtMiddleware

Under the hood, Scrapy uses a downloader middleware called `RobotsTxtMiddleware`. This middleware:
- Intercepts all outgoing requests
- Checks them against the cached robots.txt rules
- Returns a 403 response (forbidden) for blocked URLs
- Logs which requests were blocked

You can see these blocked requests in your logs:
```
DEBUG: Forbidden by robots.txt: <GET https://example.com/admin/>
```

### The ROBOTSTXT_PARSER Setting

Scrapy supports multiple robots.txt parsers:

```python
# settings.py
ROBOTSTXT_PARSER = 'scrapy.robotstxt.ProtegoRobotParser'  # Default, fast
# ROBOTSTXT_PARSER = 'scrapy.robotstxt.RerpRobotParser'  # Alternative
```

**Protego** (default): Fast, compliant with RFC 9309, handles wildcards correctly.
**Rerp**: Older parser, slightly different wildcard handling.

For 99% of cases, stick with the default.

### Testing Without robots.txt

During development, you might want to test your spider without robots.txt restrictions:

```python
# settings.py (development only!)
ROBOTSTXT_OBEY = False
```

**Warning**: Never deploy to production with this setting. It's only for local testing.

I strongly recommend keeping this set to `True` for every project unless you have a very specific, ethical reason to change it.

> [!TIP]
> **Scrapy Doc Gap: The ROBOTSTXT_OBEY Setting**
> Beginners often think that setting `ROBOTSTXT_OBEY = True` is enough to make their scraper "legal." 
> 
> In reality, this is purely a *technical* instruction to the Scrapy engine. It tells Scrapy to look for a file and skip URLs listed there. However, a website might have *legal* terms on a separate page that forbid scraping even if the `robots.txt` file is empty. Think of `ROBOTSTXT_OBEY` as a "politeness filter," not a legal shield.

## Ethical Considerations

Sometimes, you might find a `robots.txt` file that is clearly broken or overly aggressive. For example, a site that has no API and forbids all scraping, yet publishes public data that is useful for the community.

In these cases, you have a choice. You *could* set `ROBOTSTXT_OBEY = False`. But before you do that, ask yourself:
1.  **Why did they block this?** (Is it because the site is fragile? Or because they want to sell the data?)
2.  **Can I ask for permission?** (Sometimes a quick email to the site owner can get you an "official" whitelist).
3.  **Am I causing harm?** (If you ignore the rules, you *must* be even more careful with your crawl speed).

## Chapter Summary

**What we covered:**
- `robots.txt` is the "gentleman's agreement" of the web.
- It lives at the root of every domain (e.g., `example.com/robots.txt`).
- `User-agent` defines who the rules apply to.
- `Disallow` defines where you shouldn't go.
- Scrapy respects these rules automatically if `ROBOTSTXT_OBEY` is `True`.
- Ethical scrapers always check the rules before they start.

**Remember:**
- `robots.txt` is a sign, not a lock.
- Respecting the rules builds trust in the scraping community.
- Ignoring a "Stay Out" sign can lead to your IP being blocked permanently.

**Practice Exercise:**
Go to five of your favorite websites and find their `robots.txt` file. Can you find any parts of the site that are forbidden? Is there a `Crawl-delay` specified? Record what you find in a simple text file.

**Previous chapter:**
[Chapter 1: Welcome to Web Scraping](./chapter_01_welcome_to_web_scraping.md)

**Next chapter:**
Now that we know the rules of the house, it's time to understand how the house is built. In the next chapter, we're going to dive deep into **How Websites Work** the marriage of HTTP, HTML, CSS, and the mysterious world of JavaScript.

---

**End of Chapter 2**
