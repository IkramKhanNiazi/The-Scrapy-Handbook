# Chapter 36: Signals and Events

Think about the first time you feel like a "fly on the wall" inside your own software. You're trying to debug a very strange issue: every few hours, your spider stops for exactly three minutes, then continues as if nothing had happened. You check the logs nothing. You check the database nothing. You check the server's CPU nothing.

You might feel like you're trying to solve a crime where the suspect is invisible and left no fingerprints.

Then, you learn about **Signals**.

You'll realize that even when Scrapy "isn't saying anything," it's actually talking constantly. It's shouting messages like "Request scheduled!" and "Spider idle!" every millisecond. You just haven't been listening. You write a tiny script to "eavesdrop" on every signal the engine is sending out. 

Ten minutes into the next crawl, the secret will be revealed. You'll see a signal named `spider_idle`. It's being sent over and over again because a specific middleware you wrote is "swallowing" requests but not telling the engine it's done. The engine is essentially sitting in a room, waiting for a person who never arrived.

That experience will teach you that the real power of Scrapy isn't in what *you* do; it's in what the engine *tells* you it's doing. Signals are the secret language of the framework. Once you learn that language, you stop being a passenger and start being the conductor of the orchestra.

In this chapter, you're going to learn how to listen.

---

## Introduction

Scrapy's internal architecture is built on a "Signal" system. Whenever any part of the engine the Downloader, the Scheduler, or the Spider does something important, it sends out a message to the rest of the framework.

In this chapter, we're going to explore the **Nervous System** of Scrapy. We'll learn how to identify the core signals, how to "connect" your own code to these signals so you can react to events as they happen, and how to build a unified system of alerts and events that make your project feel alive. By the end of this chapter, you'll have the "X-ray vision" to see exactly what Scrapy is doing at any given second.

## What are Signals?

Think of a **Signal** as an "Event." When you click a button in your browser, an event is sent. When Scrapy finds an item, a signal is sent.

Signals allow different parts of Scrapy to remain "decoupled." The Spider doesn't need to know that the MongoDB pipeline exists. It just shouts, "I found an item!" and the MangoDB pipeline (which is listening for that specific signal) catches it and saves it.

## The Core Signals List

> [!NOTE]
> **Scrapy Doc Gap: The Missing dependency**
> Scrapy's signal system relies on a library called `blinker`. For small projects, it works without it, but for robust signaling, Scrapy will complain if it's missing.
> Always run `pip install blinker` in production environments to ensure signals fire reliably.

Here are the signals that professional Scrapy developers listen to the most:

*   **`spider_opened`**: Perfect for initializing database connections or starting log timers.
*   **`item_scraped`**: Ideal for real-time validation or live dashboards.
*   **`spider_error`**: Trigger this to send an urgent Slack message if a crash occurs.
*   **`request_scheduled`**: Learn which URLs are "in the queue" before they are visited.
*   **`engine_stopped`**: The final signal. Use this for your last "Goodnight" logs.

### Cleanup: `spider_closed` vs `engine_stopped`
There is a subtle difference.
*   `spider_closed`: The spider is done, but the process is still running (pipelines might be finishing up). Use this to close DB connections.
*   `engine_stopped`: The process is about to exit. Use this to delete temporary files or PID locks (`/tmp/myspider.lock`).

> [!TIP]
> **Scrapy Doc Gap: The spider_idle Sleeping Giant**
> The `spider_idle` signal is triggered whenever the engine is out of requests. By default, Scrapy shuts down when it hears this. 
> 
> However, a pro move is to "catch" this signal and yield a new set of requests from a database or an API. This allows you to create an **Infinite Crawler** that never stops it just waits for the next task to appear. If you yield a request during `spider_idle`, the engine will stay alive and keep working!

## Signal Handlers: How to "Eavesdrop"

To listen to a signal, you need to "connect" a function to it. You usually do this in an **Extension** (as we saw in Chapter 35), but you can also do it inside your Spider.

```python
from scrapy import signals

class MySpider(scrapy.Spider):
    name = 'secret_listener'

    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        spider = super(MySpider, cls).from_crawler(crawler, *args, **kwargs)
        # Connect to the 'item_scraped' signal
        crawler.signals.connect(spider.item_saved_alert, signal=signals.item_scraped)
        return spider

    def item_saved_alert(self, item):
        self.logger.info(f"YAY! I just caught a {item['name']}!")
```

### Pro Pattern: The TQDM Progress Bar
Scrapy's default log output (`Item scraped: 1`) is boring.
Let's add a real-time progress bar using `tqdm` and signals.

```python
from tqdm import tqdm

class ProgressBarExtension:
    def __init__(self):
        self.pbar = tqdm(desc="Items Scraped", unit="it")

    @classmethod
    def from_crawler(cls, crawler):
        ext = cls()
        crawler.signals.connect(ext.item_scraped, signal=signals.item_scraped)
        crawler.signals.connect(ext.spider_closed, signal=signals.spider_closed)
        return ext

    def item_scraped(self, item, spider):
        self.pbar.update(1)

    def spider_closed(self, spider):
        self.pbar.close()
```

## Real-World Use Case: Global Exception Logging

Imagine you want a record of *every* error that happens across your entire cluster of ten servers. Instead of looking at ten different log files, you can listen to the `spider_error` signal and send the details to a central database.

```python
def spider_error(self, failure, response, spider):
    # This function catches errors from ANY callback
    send_to_central_db({
        'spider': spider.name,
        'url': response.url,
        'error': str(failure.value)
    })
```

## Creating Custom Signals

One of the most advanced techniques in Scrapy is creating your own signals. If you are building a very complex system, you might want your pipeline to send a signal to your extension.

```python
from scrapy.signalmanager import SignalManager

# Create a unique object for your signal
my_signal = object()

# To send it:
crawler.signals.send_catch_log(signal=my_signal, arg1="data")
```

## Chapter Summary

**What we covered:**
- Signals are the "internal messages" that Scrapy sends out during its lifecycle.
- The system is "decoupled," meaning parts of Scrapy can talk without knowing who is listening.
- Professional developers use signal handlers to build robust alerts and monitoring.
- The `spider_error` and `spider_idle` signals are critical for high-level debugging.
- Learning to "listen" to Scrapy is the final step in moving from a beginner to an expert.

**Previous chapter:**
[Chapter 35: Scrapy Extensions](./chapter_35_scrapy_extensions.md)

**Next chapter:**
You've completed **Part VII: Advanced**. You now know more about Scrapy's internal architecture than 95% of users. This marks the end of your technical training. 

In the next part, **Part VIII: Industry Patterns**, we're going to put all this power to use by exploring how the "Big Players" scrape the web. We'll start with **E-commerce Strategies** learning the specific techniques used to crawl sites like Amazon and eBay.

---

**End of Chapter 36**
