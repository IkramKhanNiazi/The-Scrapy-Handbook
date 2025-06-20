# Chapter 11: Exporting Your Data

Think back to the first time a client asks you for "the data." You've been working on a spider for three days, and you're proud of the clean, validated output you're seeing in your terminal. You tell the client, "It's ready! I just need to send it to you."

"Great," they say. "Can you send it as a CSV? Our marketing team needs to open it in Excel."

You might panic. You hadn't actually thought about *how* to save the data. You were so focused on the scraping that you'd forgotten about the "shipping." You might spend the next four hours trying to write a manual Python script using the `csv` library to take your list of dictionaries and turn them into a file. You'll struggle with column headers, fight with special characters that break the rows, and potentially overwrite your data three times before getting it right.

It feels like being a gold miner who has found a massive vein of gold but has no way to get it out of the mountain. All that value is sitting right there, but you can't move it.

If you finally send the file long after the deadline, that's when you learn about Scrapy's **Feed Exports**.

It's a huge discovery. You'll realize that Scrapy already knows how to save your data to almost any format you can imagine. You don't need to write any file-handling code. You just have to tell Scrapy, "Save this as a CSV," and it will handle the headers, the encoding, and the file creation automatically. You could save yourself hours of stress with just a single line of configuration.

In this chapter, you're going to learn how to let Scrapy handle the "shipping" so you can focus on the "mining."

---

## Introduction

Getting data out of Scrapy and into a format your users can actually use is the final step in the extraction process. Whether you need a simple JSON file for yourself, a CSV for a business team, or an XML feed for another program, Scrapy has you covered.

In this chapter, we're going to master **Feed Exports**. We'll learn how to export data using command-line flags, how to configure permanent exports in your settings, and even how to send your data directly to the cloud.

## Output Format Options

Scrapy supports several formats out of the box:
*   **JSON:** The standard for most modern applications. Easy to read for computers.
*   **JSON Lines (`jsonl`):** Like JSON, but each item is on a new line. This is much better for huge datasets because you can read it line-by-line without loading the whole file into memory.
*   **CSV:** The king of business data. Opens in Excel and Google Sheets.
*   **XML:** Often used for legacy systems or sitemaps.

## The Quick Way: Command Line Flags

You've already seen the quickest way to export data:
```bash
# Export to JSON
scrapy crawl quotes -O results.json

# Export to CSV
scrapy crawl quotes -O results.csv
```
The `-O` (capital O) overwrites the file. If you want to *append* data to an existing file, use a lowercase `-o`.

> [!TIP]
> **Scrapy Doc Gap: FEEDS vs. Pipelines**
> A common question is: "Should I write a pipeline to save my data to a file?" In 99% of cases, the answer is **No**. 
> 
> The `FEEDS` system is purpose-built for saving the results of a whole crawl. It handles all the complex file-locking and encoding for you. You should only use an **Item Pipeline** if you need to perform logic on *each item* as it arrives (like checking a database for existence) or if you need to save data to a format Scrapy doesn't support out-of-the-box (like a custom PDF generator). For JSON, CSV, and S3, always stick to `FEEDS`.

## The Pro Way: `FEEDS` Configuration

Using command-line flags is great for testing, but for production projects, you want your export settings to be part of your code. We do this in the `settings.py` file using the `FEEDS` setting.

```python
# settings.py
FEEDS = {
    'data/quotes_batch_1.json': {
        'format': 'json',
        'encoding': 'utf8',
        'store_empty': False,
        'indent': 4,
    }
}
```

This configuration tells Scrapy: "Every time this spider runs, save the results to `data/quotes_batch_1.json` in JSON format."

### Dynamic File Names
What if you want to save your data with the date of the crawl? Scrapy allows you to use "placeholders" in the filename:
```python
'data/%(name)s/%(time)s.csv': {
    'format': 'csv',
}
```
This will create a folder with the spider's name and a file with the current timestamp. It's an incredibly powerful way to keep your historical data organized.

This will create a folder with the spider's name and a file with the current timestamp. It's an incredibly powerful way to keep your historical data organized.

### Multiple Simultaneous Exports

Since Scrapy 2.1, you aren't limited to one output. You can save to JSON and CSV at the same time:

```python
# settings.py
FEEDS = {
    'results.json': {'format': 'json', 'encoding': 'utf8'},
    'results.csv': {'format': 'csv', 'encoding': 'utf8'},
    's3://my-bucket/backup.json': {'format': 'json'},
}
```

## How It Works Under the Hood: Atomic Writes

Have you ever wondered what happens if your spider crashes halfway through generating a file? Do you end up with a half-broken JSON file?

**No.** Scrapy uses **Atomic Writes**.

1. Scrapy writes to a temporary file first (e.g., `.tmp.results.json`).
2. Only when the spider finishes successfully does it **rename** the file to the final name.
3. If the spider crashes, the temp file remains, and your final file is untouched (or non-existent).

**Why this matters for Cloud Storage:**
Scrapy uploads the file to S3/GCS only *after* the crawl finishes. It does not stream data to the cloud in real-time. If you have a 10GB dataset, Scrapy needs 10GB of local disk space to store the temp file before uploading.

> [!WARNING]
> **Scrapy Doc Gap: URI Parameters**
> You can pass storage-specific parameters in the URI for extra control:
> `s3://my-bucket/data.json?acl=public-read` will make the uploaded file public.

## Cloud Storage Setup Guide

### 1. Amazon S3

**Install requirements:**
```bash
pip install botocore
```

**Configuration:**
```python
# settings.py
AWS_ACCESS_KEY_ID = 'YOUR_ACCESS_KEY'
AWS_SECRET_ACCESS_KEY = 'YOUR_SECRET_KEY'

FEEDS = {
    's3://my-bucket/data/%(time)s.json': {
        'format': 'json',
    }
}
```

### 2. Google Cloud Storage (GCS)

**Install requirements:**
```bash
pip install google-cloud-storage
```

**Configuration:**
```python
# settings.py
GCS_PROJECT_ID = 'my-gcp-project'

FEEDS = {
    'gs://my-bucket/data.json': {
        'format': 'json',
    }
}
```
**Auth:** Scrapy will automatically use your "Application Default Credentials" (run `gcloud auth application-default login` on your machine).

### 3. Azure Blob Storage

**Install requirements:**
```bash
pip install azure-storage-blob
```

**Configuration:**
```python
# settings.py
AZURE_ACCOUNT_NAME = 'myaccount'
AZURE_ACCOUNT_KEY = 'mykey'

FEEDS = {
    'azure://mycontainer/data.json': {
        'format': 'json',
    }
}
```

## Handling Special Characters in CSV

CSV files are famously difficult to work with when they contain special characters like emojis or foreign language scripts. Scrapy handles this by default by using **UTF-8** encoding. If you open a Scrapy-generated CSV and see weird symbols in Excel, it's usually because Excel is trying to use a different encoding.

**Pro Tip:** Always use UTF-8. It is the universal standard for a reason.

## Appending vs. Overwriting

*   **Overwriting (`-O`)**: Use this when you are running a fresh crawl and you want to replace the old results.
*   **Appending (`-o`)**: Use this when you are scraping in small batches and you want to build up one large dataset over time.

## Chapter Summary

**What we covered:**
- Feed Exports are the built-in way to save data to files.
- JSON, CSV, and XML are the most common formats.
- JSON Lines is the preferred format for "Big Data" projects.
- You can automate exports using the `FEEDS` setting in `settings.py`.
- Dynamic placeholders like `%(time)s` help you keep your files organized.
- Scrapy can export directly to cloud storage like AWS S3.

**Key code:**
```bash
# The basic export command
scrapy crawl my_spider -O results.json
```

**Previous chapter:**
[Chapter 10: Understanding Item Pipelines](./chapter_10_understanding_item_pipelines.md)

**Next chapter:**
You've mastered the lifecycle of a standard spider. But what if you need to crawl an entire domain but only follow specific types of links? Or what if you need to pass arguments to your spider from the command line? In the next chapter, we're going to explore **Advanced Spider Patterns** taking you from a static spider to a dynamic, rule-based crawler.

---

**End of Chapter 11**
