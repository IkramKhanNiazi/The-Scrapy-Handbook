# Chapter 32: Scheduling Spiders (Cron)

Think about the first time you achieve the "Scraper's Dream." You've built a price monitoring tool for your own personal use. You want to know when a specific pair of hiking boots goes on sale. Every morning for a week, you wake up at 7:00 AM, open your laptop, run the spider manually, and check the results.

You might find it annoying to be "babysitting" your code.

One Friday night, you decide to learn about **Cron**. You might spend two hours reading confusing tutorials about asterisks and slashes. You write a tiny shell script, add a line to your system's "crontab," and go to bed.

The next morning, you don't open your laptop. You go for a run. You have breakfast. You forget about the boots entirely. Around 10:00 AM, you check your email. There, sitting at the top of your inbox, you find an automated alert: "Price Dropped! $149 -> $99." 

You haven't touched a single button. Your computer "woke up," ran the spider, found the deal, and emailed you while you were sleeping. You'll feel as if you've finally built a robot that actually works for you. You are no longer a "task runner"; you are a "system owner." Your time is your own again, but your data is still being collected as if you were working 24/7.

In this chapter, you're going to learn how to achieve that same dream.

---

## Introduction

In the previous chapters, we built a server and a monitoring system. But you still have to manually type the command to start the crawl. In this chapter, we're going to remove the human from the equation.

We'll learn about **Cron**, the universal scheduling tool for Linux. We'll master its "mysterious" syntax, learn how to write **Shell Scripts** to wrap our Scrapy commands, and discover how to handle "overlaps" (to make sure you don't run the same spider twice at the same time). By the end of this chapter, your spiders will run entirely on their own.

## Why Automate? The "Set and Forget" Dream

A professional scraper doesn't just collect data once. It monitors the world.
*   **Daily:** Tracking stock levels or prices.
*   **Weekly:** Gathering newly published research papers.
*   **Hourly:** Monitoring breaking news or social media trends.

Automation ensures that your data is always fresh, without you having to remember to "hit the button."

## Introduction to Cron

**Cron** is a background service that runs on almost every Linux server (like the VPS we built in Chapter 30). It checks a "to-do list" (the **Crontab**) every minute and runs any command scheduled for that time.

### Opening the Crontab
To edit your to-do list, type:
```bash
crontab -e
```

## Cron Syntax Masterclass

A cron job looks like five numbers (or asterisks) followed by a command:

`* * * * * command_to_run`

Here is what they mean:
1.  **Minute** (0 - 59)
2.  **Hour** (0 - 23)
3.  **Day of Month** (1 - 31)
4.  **Month** (1 - 12)
5.  **Day of Week** (0 - 6, where 0 is Sunday)

### Examples:
*   `0 3 * * *`: Every night at 3:00 AM.
*   `0 0 * * 1`: Every Monday at midnight.
*   `*/15 * * * *`: Every 15 minutes.

For Cron, run `sudo apt install cron -y` if it's not installed.

### Professional Scheduling: Scrapyd
Using `cron` for 50 spiders gets messy. The professional tool is **Scrapyd**.
It's a daemon (service) that listens for API commands to run spiders.

1. Install it: `pip install scrapyd scrapyd-client`
2. Run it: `scrapyd` (runs on port 6800)
3. Deploy your project: `scrapyd-deploy`

Now, you can schedule a job via a simple HTTP request:
```bash
curl http://localhost:6800/schedule.json -d project=myproject -d spider=myspider
```
This manages the queue, logging, and concurrency for you. No more overlapping cron jobs!

## Writing a Wrapper Script (The .sh File)

You shouldn't put a raw `scrapy crawl` command directly into your crontab. It's too fragile. Instead, write a simple **Shell Script** that handles the environment setup:

```bash
#!/bin/bash
# run_spider.sh

# 1. Navigate to the project folder
cd /root/projects/my-scraper

# 2. Activate the virtual environment
source venv/bin/activate

# 3. Run the spider and log the output
scrapy crawl quotes -O results.json >> cron_logs.txt 2>&1
```

**Pro Tip:** Make the script executable with `chmod +x run_spider.sh`.

## Handling Overlaps with `flock`

What happens if your spider takes 20 minutes to run, but you have it scheduled every 15 minutes? You'll end up with two spiders running at once, which can lead to memory crashes or IP bans.

The solution is `flock` (File Lock). It ensures that only one instance of the script runs at a time.
```bash
*/15 * * * * /usr/bin/flock -n /tmp/spider.lock /root/run_spider.sh
```

## Containerized Scheduling (Docker + Cron)
If you are using Docker (Chapter 31), you don't use the host's cron. You use a "Sidecar" container.

```dockerfile
# Dockerfile
FROM python:3.9
RUN apt-get update && apt-get install -y cron
COPY . .
RUN crontab /app/crontab.txt
CMD ["cron", "-f"] # Run cron in foreground
```

## Orchestration with Airflow
For complex workflows ("Run Spider A, *then* if successful, run Spider B"), Cron is not enough. You need **Apache Airflow**.

```python
# daily_dag.py
from airflow import DAG
from airflow.operators.bash import BashOperator

with DAG('daily_scrape') as dag:
    t1 = BashOperator(task_id='scrape_news', bash_command='scrapy crawl news')
    t2 = BashOperator(task_id='scrape_prices', bash_command='scrapy crawl prices')
    
    t1 >> t2 # Run t1 first, then t2
```
This is the Enterprise Standard for data engineering.

In 2026, you don't always need Cron.
*   **GitHub Actions:** Can run Scrapy spiders on a schedule for free.
*   **Heroku Scheduler:** A simple web interface for running daily tasks.
*   **AWS EventBridge:** Highly scalable but more complex.

## Chapter Summary

**What we covered:**
- Cron is the industry-standard tool for automating tasks on Linux.
- The CRON syntax uses five time positions (* * * * *) to define a schedule.
- Shell scripts (.sh) are the professional way to "wrap" Scrapy commands for execution.
- `flock` prevents dangerous overlaps by ensuring only one spider runs at a time.
- Automation turns your scrapers from "tools" into "systems that run themselves."

**Key code:**
```bash
0 0 * * * /path/to/my_script.sh
```

**Previous chapter:**
[Chapter 31: Monitoring and Logging](./chapter_31_monitoring_and_logging.md)

**Next chapter:**
You've mastered **Part VI: Deployment**. You can build, harden, host, monitor, and automate any scraper in the world. You are now a fully-fledged Scraper Developer. 

In the next part, **Part VII: Advanced**, we're going to "break open" the Scrapy engine itself. We'll start with **Spider Middlewares** learning how to change how Scrapy thinks before it even sends a request.

---

**End of Chapter 32**
