# Chapter 28: Speed vs. Ethics

Think about the first time you are tempted to "break the rules." You're working on a project for a startup that needs data from a small, specialized medical research site. The site is old, and it is slow. You know that if you use your standard Scrapy settings, it will take three days to finish. 

You might be in a hurry. You might think, "If I just increase the concurrency and remove the download delay, I can get this done in two hours. Who is going to notice? It's just a program running in the background."

You hit the "Run" button with zero delays. Within ten minutes, you've scraped five thousand pages. You feel like a genius.

Then, you try to open the website in your own browser to check the data. The page doesn't load. You wait thirty seconds. Still nothing. You try again from your phone using cellular data. The site works perfectly there. You'll realize with a sudden, sinking feeling in your stomach what you've done: you haven't just "scraped" the site; you've essentially "attacked" it. Your spider is taking so much of that old server's resources that real researchers real doctors looking for life-saving information can't access the site while you're running your script.

You might feel like a thief. You'll realize that your "speed" has a human cost. It's a wake-up call. Being a "professional" isn't about being fast. It's about being responsible. 

In this chapter, you're going to learn where to draw the line.

---

## Introduction

As a master of Scrapy, you now hold an incredible amount of power. You can build systems that can visit millions of pages in a day. But with that power comes a serious responsibility. In the world of web scraping, we are not authors or developers; we are **Guests**.

In this chapter, we're going to explore the ethics of high-speed scraping. We'll talk about the impact your spiders have on the health of the servers you visit, how to build "Respectful Crawlers" that don't get you noticed, and how to handle it when a website politely (or not-so-politely) asks you to slow down. By the end of this chapter, you'll know how to build scrapers that are not only powerful but also ethical and professional.

## The Ethics of Speed: Why Faster Isn't Always Better

Why should you care if a website's server slows down?
1.  **Server Costs:** Every time your spider visits a page, the website owner pays for the bandwidth and the electricity. If you scrape too fast, you are literally costing them money.
2.  **User Experience:** As I learned with the medical site, your speed can prevent real people from using the website.
3.  **The "Scraper Arms Race":** Every time a "bad" scraper crashes a site, that site owner adds more aggressive anti-bot protections. Eventually, this makes the web harder to scrape for everyone.

## Impact on Server Health: The "DDoS" Effect

A Distributed Denial of Service (DDoS) attack is designed to crash a server by sending too much traffic. If you run a distributed swarm of Scrapy spiders with no delays, you are essentially launching a DDoS attack.

A professional scraper should never aim to use more than 10-20% of a website's capacity.

## Responsible Scraping Patterns

Ethics shouldn't just be a feeling; it should be built into your code. Here are the professional standards:

### 1. Identify Yourself (The User-Agent)
Don't hide who you are. Include your project name or a contact email in your `USER_AGENT` string. This allows a site owner to reach out and ask you to slow down before they block you permanently.
```python
USER_AGENT = 'MedicalScraperProject (Contact: research@example.com)'
```

### 2. Use Random Delays
Humans don't click a link exactly every 1.5 seconds. Spiders do. Using a random delay (between 0.5x and 1.5x your setting) makes your traffic look more natural and less aggressive.

### 3. Respect the "Crawl-Delay"
As we learned in Chapter 2, always check the `robots.txt` for a `Crawl-delay` directive. If a site owner asks for a 5-second wait, give them 5 seconds.

## When to Slow Down

Most scrapers "start fast and slow down." Professionals do the opposite. They **start slow and speed up** only if the server shows it can handle it.
*   **Watch for 503 Errors:** These mean the server is overloaded. If you see them, stop your spider immediately.
*   **Watch for 429 Errors:** This means "Too Many Requests." The server is asking you to slow down. Listen to it.

### Implementing Adaptive Throttling
Scrapy's AutoThrottle is good, but building your own logic gives you more control.

```python
# middlewares.py
import time
from scrapy.exceptions import IgnoreRequest

class TooManyRequestsRetryMiddleware:
    def process_response(self, request, response, spider):
        if response.status == 429:
            # Server says "Stop!"
            retry_after = response.headers.get('Retry-After')
            if retry_after:
                seconds = int(retry_after)
                spider.logger.warning(f"Sleeping for {seconds}s due to 429")
                time.sleep(seconds) # Blocking sleep (careful!) or use proper reactor delay
                return request.replace(dont_filter=True)
        return response
```

### The Math of Rate Limiting
How fast is "too fast"?
- **1 Request / Second:** Highly safe.
- **5 Requests / Second:** Borderline. Humanly impossible, but usually tolerated.
- **10+ Requests / Second:** Danger zone. Likely to trigger WAFs (Web Application Firewalls).

**Pro Tip:** If you have 10,000 URLs, scrape them over 3 hours (1 req/sec) rather than 10 minutes. The data is rarely *that* urgent, and staying unbanned is priceless.

## The Scraper's Code of Honor

1.  **Don't scrape sensitive data:** Never scrape personal IDs, bank info, or private records.
2.  **Don't scrape during peak hours:** If you are scraping a local shop's site, run your spider at 3:00 AM when their customers aren't online.
3.  **Give back to the data source:** If you use a site's data to build a successful project, consider reaching out to the owner to offer them a copy of your findings.

> [!IMPORTANT]
> **Scrapy Doc Gap: Identifying as a Good Citizen**
> Most documentation focuses on how to *avoid* being blocked. Professional ethics is about ensuring you *don't deserve* to be blocked. 
> 
> A scraper that identifies itself, respects `robots.txt`, and uses `AutoThrottle` is significantly less likely to face legal challenges. In many jurisdictions, "unauthorized access" is often defined by whether you bypassed active security measures or ignored explicit "politeness" instructions. By being a "Good Citizen," you are protecting not just the website, but also yourself.

If you scrape too fast, you might get the data today, but you'll be banned for life tomorrow. If you scrape ethically, you can maintain a relationship with that website for years. Professional scraping is a marathon, not a sprint.

### Checklist: Bad Bot vs. Good Bot

| Feature | Bad Bot 😈 | Good Bot 😇 |
| :--- | :--- | :--- |
| **User-Agent** | Fake (`Chrome/99`) | Honest (`MyBot (+contact@me.com)`) |
| **Robots.txt** | Ignored | Respected (`ROBOTSTXT_OBEY = True`) |
| **Concurrency** | Max (`100+`) | Low (`1-8`) |
| **Retry-After** | Ignored | Slept |
| **Goal** | Crash & Grab | Sustainable Data |

**The Resume Test:**
Don't write a scraper you would be ashamed to show in a job interview. "I scraped 1M pages by bypassing their firewall and crashing their API" is NOT a flex. It's a liability. "I scraped 1M pages over 3 days with zero downtime and 429 errors" is professional engineering.

## Chapter Summary

**What we covered:**
- High speed comes with the responsibility of server health and user access.
- Professional scrapers identify themselves and respect the infrastructure they visit.
- Ethical scraping requires monitoring for "Stress Signals" like `503` or `429` errors.
- Random delays and respectful User-Agents are the marks of a pro.
- The goal is to build long-term, sustainable data harvesting systems, not "crash and grab" scripts.

**Key code:**
```python
RANDOMIZE_DOWNLOAD_DELAY = True
# Be transparent!
USER_AGENT = "MyProjectBot/1.0 (+http://mywebsite.com/bot)"
```

**Previous chapter:**
[Chapter 27: Resource Optimization](./chapter_27_resource_optimization.md)

**Next chapter:**
You've mastered Part V! You are now an expert in scaling, distribution, and ethics. This completes your training as a "Spider Architect." 

In the next part, **Part VI: Deployment**, we're going to learn how to move your code off your laptop and into the real world. We'll start with **Hardening for Production** preparing your code for the rigors of the "Live" internet.

---

**End of Chapter 28**
