# Chapter 23: NoSQL Databases

Think about the first time you try to scrape a "Social Media" style website. Every post is different. Some have a title and a body. Others have an image and a list of hashtags. Some have a location tag, and others have a complex list of "reactions" (likes, hearts, and claps).

You might try to use a SQL database, which means you have to define every possible column in advance. Your table might end up having over fifty columns, most of which are empty for 90% of the rows. Every time the website adds a new feature like "Stories" or "Live Polls" you have to manually alter your database schema to add more columns. Your SQL code can quickly become a huge mess of `NULL` checks and complex `JOIN` statements.

It can feel as if you're trying to pack a variety of strangely shaped camping gear into a rigid, rectangular chest. No matter how much you push and shove, things just won't fit right. You'll find yourself spending all your time "modeling" the data and no time actually using it.

You'll realize that your data is "unstructured," and you're trying to force it into a "structured" world.

Then, you discover **MongoDB**.

It's like switching from a rigid chest to a set of flexible, high-quality duffel bags. You won't have to define columns in advance. If a post has hashtags, you just save them. If it doesn't, you don't. Each "document" can be as unique as the data it represents. You won't have to worry about schema changes or `NULL` values ever again. You can just focus on the scraping, knowing that your database is flexible enough to handle whatever you throw at it.

In this chapter, you're going to learn how to find that flexibility.

---

## Introduction

So far, we've focused on SQL databases, which are perfect for data that fits neatly into rows and columns. But the web is often messy and unpredictable. Sometimes, you need a database that is as flexible as the data you're scraping. This is the world of **NoSQL**.

In this chapter, we're going to explore **MongoDB**, the most popular document-based database in the world. We'll learn how to connect it to Scrapy and how it handles nested, complex data effortlessly. We'll also briefly touch on **Elasticsearch**, the king of search and analytics, to see how it can help you search through millions of records in a heartbeat.

## When to Use NoSQL

You should reach for a NoSQL database (like MongoDB) when:
1.  **Your data is inconsistent:** Different items have different fields.
2.  **Your data is deeply nested:** You have items inside items inside items.
3.  **You are in a "Discovery" phase:** You don't know exactly what the data looks like yet, and you want to start saving things immediately without defining a schema.
4.  **You need high-speed writes:** NoSQL databases are often faster at "dumping" massive amounts of raw data.

## MongoDB Basics

In MongoDB, we don't have "Tables." We have **Collections**. Instead of "Rows," we have **Documents**. A document looks almost exactly like a Python dictionary or a JSON file.

### Installation
You can use **MongoDB Atlas** (a free cloud-hosted version) or install it locally. You'll also need the `pymongo` library:
```bash
pip install pymongo
```

## MongoDB with Scrapy

Integration is simple because Scrapy's "Item" already looks like a dictionary, which is exactly what MongoDB wants.

### The MongoDB Pipeline
```python
import pymongo

class MongoPipeline:
    collection_name = 'scraped_items'

    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE')
        )

    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]

    def close_spider(self, spider):
        self.client.close()

    def process_item(self, item, spider):
        # insert_one is simple, but creates duplicates. 
        # Professionals use update_one (Upsert)
        
        self.db[self.collection_name].update_one(
            {'url': item['url']},           # Filter (What makes it unique?)
            {'$set': dict(item)},           # Update (Set these fields)
            upsert=True                     # Create if not exists
        )
        return item

## Performance: Indexing

You cannot have a fast database without indexes. If you are upserting by `url`, you MUST create an index on `url`, or your scrape will get slower and slower with every item.

```python
    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]
        
        # Create unique index on 'url' to speed up queries
        self.db[self.collection_name].create_index(
            [("url", pymongo.ASCENDING)],
            unique=True
        )
```

> [!TIP]
> **Scrapy Doc Gap: NoSQL Doesn't Mean "No Schema"**
> A common trap in MongoDB development is "dumping" everything and figuring it out later. While MongoDB allows this, you shouldn't. 
> 
> You should still use **Scrapy Items** to define what your data *should* look like. This creates a "view" of your data that remains consistent even if the website changes. Without this, your database will end up with "Data Rot" years of records where the same field is named `price`, then `Price`, then `cost`. Use Scrapy Items to enforce consistency at the source!
```

## Elasticsearch Basics

While MongoDB is great for storing data, **Elasticsearch** is built for *searching* it. If you have ten million items and you want to perform a "Google-like" search for a specific keyword in the description, Elasticsearch is the tool for the job.

It uses a REST API, which means you can talk to it using the standard `requests` library or specialized Scrapy pipelines. We'll see it again in the "Deployment" part of the book.

### Bulk Indexing with Scrapy

Elasticsearch is slow if you insert items one by one. You should use the `_bulk` API.

```python
# pipelines.py
from elasticsearch import Elasticsearch, helpers

class ElasticsearchPipeline:
    def __init__(self):
        self.items_buffer = []

    def process_item(self, item, spider):
        # Add to buffer
        self.items_buffer.append({
            "_index": "products",
            "_source": dict(item)
        })
        
        # Flush if buffer is full (e.g., 500 items)
        if len(self.items_buffer) >= 500:
            helpers.bulk(self.es, self.items_buffer)
            self.items_buffer = []
        return item
```

## NoSQL vs. SQL for Scraping: The Final Verdict

*   **Choose SQL (PostgreSQL)** when you have a clear structure and need complex relationships (e.g., "Show me all reviews from authors who live in New York").
*   **Choose NoSQL (MongoDB)** when your data is messy, nested, or changing quickly (e.g., "Just save everything from this social media profile").

## Chapter Summary

**What we covered:**
- NoSQL databases offer a flexible alternative to the rigid structure of SQL.
- **MongoDB** stores data as "Documents," which are a perfect match for Scrapy Items.
- We built a **MongoPipeline** that handles connecting, inserting, and closing.
- **Elasticsearch** is the industry standard for high-speed searching across large datasets.
- Flexible storage allows you to move faster during the "discovery" phase of a project.

**Key code:**
```python
self.client = pymongo.MongoClient(mongo_uri)
self.db[collection].insert_one(item_dict)
```

**Previous chapter:**
[Chapter 22: ORM Integration with Scrapy](./chapter_22_orm_integration_with_scrapy.md)

**Next chapter:**
You've mastered the lifecycle of a single spider and its data. This marks the end of **Part IV: Data Storage and Databases**. You are now a master of data persistence.

In the next part, **Scaling and Distribution**, we're going to learn how to move from a single spider to a "swarm" of spiders. We'll start with **Understanding Distributed Crawling** legal, ethical, and technical.

---

**End of Chapter 23**
