# Chapter 35: Scrapy Extensions

Think about the first time you want to add a "self-destruct" feature to your scrapers. You're running a project on a server with very limited disk space. If the scraper collects more than 5GB of images, you want it to stop immediately and send you a text message before it crashes the entire server.

You might try to put the logic inside your spider. You add a check to the `parse` method: "Is the folder bigger than 5GB?" 

But there's a problem. The spider only checks that logic when it's processing a page. If the downloader is busy fetching five hundred images in the background, the spider might not "run" for another three minutes. By then, the folder could be 6GB, and the server would be dead. 

You'll realize that your "Disk Space Guard" shouldn't be part of the spider. It needs to be part of the *Scrapy Engine itself*. It needs to be something that sits outside the crawl and watches the overall health of the system every second, regardless of what the spider is doing.

That's when you discover **Scrapy Extensions**.

It's like hiring a dedicated "Safety Officer" for your factory. Instead of having the workers check for fire (the spider), you can have a separate person whose only job is to watch the sensors and hit the emergency stop button if things get dangerous. Scrapy extensions are the most powerful way to add features to your projects that don't belong in a spider or a middleware. They are the ultimate "Plugin" system.

In this chapter, you're going to learn how to build your own safety officers.

---

## Introduction

So far, we've modified how data is handled (Pipelines) and how pages are fetched (Middlewares). But what if you want to modify how the Scrapy engine itself behaves? What if you want to track global stats, send an email when a crawl ends, or shut down the spider after a certain amount of time?

This is where **Scrapy Extensions** come in.

Extensions are simple Python classes that listen to **Signals** messages that Scrapy sends out internally when important things happen. In this chapter, we're going to learn how to create custom extensions, how to use the `from_crawler` method to access every part of Scrapy, and how to build a professional "Crawl Protector" that monitors your project's health from the outside.

## What are Extensions?

Think of an extension as a "Background Process." While your spider is busy following links and your pipeline is busy saving items, the extension is just sitting there, listening.

When something happens like "Spider Opened" or "Item Scraped" Scrapy shouts out a signal. The extension hears it and performs its specific task.

> [!TIP]
> **Scrapy Doc Gap: Extensions vs. Middlewares**
> Beginners often struggle to decide which one to use. Here is the rule of thumb: 
> 
> *   Use a **Middleware** if you need to touch the **Data** (Requests, Responses, or Items). 
> *   Use an **Extension** if you need to touch the **State** of the project (Time, Server Resources, or External Alerts). 
> 
> If you are adding a header to a request, that's a middleware. If you are sending a Slack message when the crawl finishes, that's an extension.

## Built-in Extensions: The "Invisible Protectors"

You've actually been using extensions this whole time without knowing it.
*   **LogStats:** The extension that prints that "Scraped X items at Y items/min" message every minute.
*   **AutoThrottle:** The performance booster we learned about in Chapter 18 is actually an extension!
*   **CloseSpider:** An extension that shuts down your spider based on time or page count.

## Creating a Custom Extension

Extensions are usually kept in a file named `extensions.py`. Here is the skeletal structure:

```python
from scrapy import signals

class MyCustomExtension:
    @classmethod
    def from_crawler(cls, crawler):
        # 1. Instantiate the extension
        ext = cls()
        
        # 2. Connect to signals
        crawler.signals.connect(ext.spider_opened, signal=signals.spider_opened)
        crawler.signals.connect(ext.item_scraped, signal=signals.item_scraped)
        crawler.signals.connect(ext.spider_idle, signal=signals.spider_idle)
        
        return ext

    def spider_idle(self, spider):
        # This triggers when the queue is empty.
        # It's your LAST chance to schedule more requests!
        if self.should_crawl_more():
            self.crawler.engine.schedule(Request("..."), spider)
            raise DontCloseSpider("We found more work!")

    def spider_opened(self, spider):
        spider.logger.info("The Safety Officer is on duty!")

    def item_scraped(self, item, spider):
        # Do something every time an item is found
        pass
```

## The `from_crawler` Method: The Master Key

This is the most important method for advanced Scrapy developers. It gives you access to the **Crawler Object**, which contains:
*   `crawler.settings`: Every configuration value.
*   `crawler.stats`: The global stats collector.
*   `crawler.engine`: The heart of Scrapy.

With `from_crawler`, your extension can basically reach out and touch any part of the system.

## Listening to Signals: The Heartbeat

Common signals you can listen to:
*   `signals.spider_closed`: Run clean-up code or send a final report.
*   `signals.request_scheduled`: Track every single URL before it's even visited.
*   `signals.spider_error`: Trigger an alert if things go wrong.

## Real-World Use Case: The "Completion Email"

Imagine you want a summary of your crawl emailed to you as soon as it's done.

```python
def spider_closed(self, spider, reason):
    # Access the stats
    stats = self.crawler.stats.get_stats()
    items = stats.get('item_scraped_count', 0)
    
    # Send the summary
    send_email(
        subject=f"Crawl Finished: {spider.name}",
        body=f"Reason: {reason}\nItems: {items}"
    )
```

## Advanced: Building a Custom Stats Collector

Scrapy's default `MemoryStatsCollector` forgets everything when the spider closes.
If you want to send stats to **Grafana/InfluxDB**, you should build a custom collector.

```python
# settings.py
STATS_CLASS = "myproject.stats.InfluxStatsCollector"
```

Your class just needs to implement `set_value`, `inc_value`, and other service methods. 
Every time Scrapy increments `item_scraped_count`, your code runs and can push that number to an external database instantly.

## Enabling the Extension

In your `settings.py`:
```python
EXTENSIONS = {
    'my_project.extensions.MyCustomExtension': 100,
}
```

## Chapter Summary

**What we covered:**
- Extensions are background processes that handle engine-level tasks.
- They work by listening to **Signals** sent out by Scrapy.
- The `from_crawler` method provides access to settings, stats, and the engine.
- Extensions allow for global monitoring and control that isn't possible inside a spider.
- Using extensions keeps your spider logic focused purely on data extraction.

**Previous chapter:**
[Chapter 34: Downloader Middlewares](./chapter_34_downloader_middlewares.md)

**Next chapter:**
We've mentioned "Signals" many times in the last three chapters. They are the nervous system of Scrapy. In the next chapter, we're going to explore **Signals and Events** in detail learning how to "eavesdrop" on every single conversation inside Scrapy to build the ultimate customized scraper.

---

**End of Chapter 35**
