# Chapter 31: Monitoring and Logging

Think about the first time you realize you are "scraping blind." You've been running a spider for two weeks. Every morning, you check the database and see that the "total rows" number is increasing by about ten thousand. You're happy. Everything seems to be working.

Then, one morning, you decide to actually *look* at the rows you've collected.

To your horror, you'll realize that for the last seven days, your spider has been collecting empty results. The website changed its class names from `.price` to `.product-price`. Because Scrapy doesn't crash if a selector finds nothing, your spider has been faithfully visiting ten thousand pages a day, finding nothing, and saving "None" into your database. For an entire week, you've been spending money on servers and proxies to collect a pile of digital trash.

You might feel like a captain of a ship who was so proud of his compass that he hadn't noticed the ship was sinking.

You'll realize that "running" isn't the same as "working." If you don't have eyes on your data, you are just gambling. You need to know the moment your success rate drops. You need to know the moment the website changes. You need a system that "shouts" at you when something goes wrong, rather than just silently failing in the dark.

In this chapter, you're going to learn how to build those eyes.

---

## Introduction

In production, silence is rarely golden. A silent spider is often a broken one. As a professional, you need visibility into your crawl's health. You need to know your "Items-per-minute," your "HTTP Error Rate," and your "Success Percentage" in real-time.

In this chapter, we're going to dive deep into **Monitoring**. We'll learn how to master Scrapy's logging system, how to send automated alerts directly to your phone or Slack channel, and how to use professional tools like **ScrapeOps** and **Spidermon** to build a dashboard that gives you peace of mind.

## Why Monitor? The "Invisible Wall"

There are three main killers of production scrapers:
1.  **Website Structure Changes:** Selectors stop working.
2.  **IP Bans:** The website starts returning `403` or `429` errors.
3.  **Content Drift:** The data is there, but it's now in a different format (like prices changed from USD to EUR).

Monitoring catches these killers before they ruin your entire dataset.

## Scrapy Logs Deep Dive

You already know about `self.logger.info()`. But in production, you shouldn't be reading raw text files. You should be saving your logs with a strategy.

**The "Level" Strategy:**
*   `CRITICAL`: The crawler has stopped.
*   `ERROR`: Data is being lost (e.g., a missing price).
*   `WARNING`: Something is unusual (e.g., a 10-second delay).
*   `INFO`: General heartbeat (e.g., "1,000 pages finished").

### Saving logs to a file
```bash
scrapy crawl myspider -L ERROR --logfile=production_errors.log
```

### The "ELK Stack" Strategy (Elastic, Logstash, Kibana)
For massive logging (millions of lines), files are too slow. 
Professionals stream logs to **Logstash**, which indexes them in **Elasticsearch**.

You can use `python-logstash-async` to send logs silently in the background:
```python
import logging
from logstash_async.handler import AsynchronousLogstashHandler

logger = logging.getLogger('python-logstash-logger')
logger.addHandler(AsynchronousLogstashHandler('localhost', 5959, database_path='...'))
```
This lets you search `message:"blocked"` across 100 servers instantly.

> [!TIP]
> **Scrapy Doc Gap: The LOG_STDOUT Setting**
> If you have a habit of using `print()` for debugging, those messages will end up in your production logs as `DEBUG` messages. On a high-speed crawl, this can create gigabytes of junk text. 
> 
> By default, `LOG_STDOUT` is `False`. If you find your logs are full of "random" text, check if someone enabled this setting. In production, always prefer `self.logger` and keep `LOG_STDOUT = False` to ensure your log files are readable and efficient.

## Sending Alerts: The Digital Heartbeat

You shouldn't have to log into your server to see if a spider is okay. It should tell *you*.

### Slack and Discord Alerts
You can write a simple Pipeline that sends a message if the error count gets too high:
```python
import requests

def send_slack_msg(text):
    url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    requests.post(url, json={"text": text})

# In your pipeline:
if item_quality_is_low:
    send_slack_msg(f"Alert: Low quality data on {spider.name}!")
```

## Professional Tools

### 1. ScrapeOps
A "SaaS" (Software as a Service) platform built specifically for Scrapy. You add one line of code to your settings, and it gives you a beautiful, live dashboard of your crawl speed, success rate, and proxy usage. It's the industry standard for 2026.

### 2. Spidermon
A library that allows you to write "Scrapy Monitors." It can check your data after the crawl and say: "If less than 90% of items have a price, send an email."

### 3. Prometheus & Grafana (The DevOps Standard)
If you want real-time graphs ("Items per Second" or "Memory Usage"), use **Prometheus**.

Install `scrapy-prometheus-exporter` and valid metrics will be exposed at `localhost:9466/metrics`.
- **Grafana** connects to this endpoint to draw beautiful line charts.
- You can set alerts like: "If `item_scraped_count` is 0 for 5 minutes, wake me up."

### 4. Sentry (Error Tracking)
Logs are messy. Sentry groups errors together.
- Instead of 1000 lines of "TimeoutError", Sentry says: **"TimeoutError: 1k events"**.
- Integration is via `SentrySdk` in `settings.py`.

## Building a Simple Dashboard

If you don't want to use an external tool, you can save your "Stats" (which Scrapy collects automatically) to a small database and build a simple HTML page to display them. Scrapy's `Stats Collector` is your best friend here.

```python
# Accessing stats in your spider
stats = self.crawler.stats.get_stats()
item_count = stats.get('item_scraped_count', 0)
```

## Chapter Summary

**What we covered:**
- Professionals never "scrape blind"; they build systems of visibility.
- Logging levels ensure you only see the information you need to act on.
- Automated alerts (Slack/Discord) bring the news to you so you can react quickly.
- Tools like **ScrapeOps** and **Spidermon** provide professional-grade monitoring with minimal setup.
- The most important stat is the "Item Success Rate" it's the ultimate measure of your spider's health.

**Key code:**
```python
# settings.py
LOG_LEVEL = 'INFO'
LOG_FILE = 'my_logs.txt'
```

**Previous chapter:**
[Chapter 30: VPS Setup for Scrapy](./chapter_30_vps_setup_for_scrapy.md)

**Next chapter:**
We have the server. We have the monitoring. Now, we need the "Schedule." In the next chapter, we're going to learn about **Scheduling Spiders** using **Cron** and simple scripts to make your spiders run every day, every hour, or every week automatically.

---

**End of Chapter 31**
