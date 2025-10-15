# Chapter 20: SQL Databases with Scrapy

Think about the first time you deploy a scraper to a "live" server. You're using SQLite, just like you learned in your initial tutorials. It works perfectly on your laptop, and you think, "This is great. Why does anyone bother with more complex databases?"

Two days later, your phone starts buzzing with alerts. Your spider is failing.

You log into your server and see a terrifying error message: `database is locked`. You'll realize that because you're running sixteen concurrent threads to make your scraping faster, they're all trying to write to the same SQLite file at the exact same millisecond. SQLite, being a single file, can't handle the traffic. It's like a revolving door that sixteen people are trying to run through at once eventually, the door just gets stuck, and nobody can get inside.

You might feel like an amateur. You'll realize that while SQLite is a great "toy" or a tool for personal research, a professional production environment needs a "real" database engine. You need a system that's built to handle multiple connections, high-speed writes, and complex data relationships.

That's when you move your project to **PostgreSQL**.

It's like moving from a bicycle to a high-speed train. Suddenly, your sixteen threads won't be fighting anymore. PostgreSQL manages the traffic like a professional air traffic controller, ensuring that every piece of data is saved safely and quickly. Your `database is locked` errors vanish forever. Your data is now sitting in a professional environment that can handle millions of records and dozens of users at the same time.

In this chapter, you're going to learn how to move your data into that professional environment.

---

## Introduction

SQLite is wonderful for learning, but for production-grade scraping, we need a robust database server. In the world of open-source databases, **PostgreSQL** is the gold standard. It is fast, incredibly reliable, and handles heavy traffic with ease.

In this chapter, we're going to bridge the gap between Scrapy and PostgreSQL. We'll learn how to set up your database, how to write a Scrapy Pipeline that connects to it, and how to handle common issues like connection pooling and duplicate management. By the end of this chapter, you'll be able to scale your data storage to handle any project size.

## Why PostgreSQL?

Why do professionals reach for Postgres?
1.  **Concurrency:** It can handle hundreds of spiders writing data at the same time without locking.
2.  **Reliability:** It is designed to never lose data, even if the power goes out mid-crawl.
3.  **Data Integrity:** It allows you to set strict rules (like "the price must be a positive number") before the data is saved.
4.  **Scaling:** It can grow from a few megabytes to dozens of terabytes of data.

## Step 1: Installing PostgreSQL

To get started, you'll need the PostgreSQL server on your machine.
*   **Mac:** Use `brew install postgresql` or download "Postgres.app".
*   **Windows:** Download the installer from `postgresql.org`.
*   **Linux:** Use `sudo apt install postgresql`.

You'll also need a Python library called `psycopg2` to act as the "bridge" between Python and the database.
```bash
pip install psycopg2-binary
```

## Step 2: Creating Your Database and Tables

Before we run our spider, we need to build the "boxes" where our data will live. You can use a tool like **pgAdmin** or the command line (`psql`) to create your table:

```sql
CREATE TABLE scraped_quotes (
    id SERIAL PRIMARY KEY,
    text TEXT NOT NULL,
    author VARCHAR(255),
    scraped_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Step 3: The PostgreSQL Pipeline

Now, let's build the assembly line that saves our items. We'll do this in `pipelines.py`.

```python
import psycopg2
from itemadapter import ItemAdapter

class PostgresPipeline:
    def __init__(self, db_settings):
        self.db_settings = db_settings

    @classmethod
    def from_crawler(cls, crawler):
        # Grab the database connection info from settings.py
        return cls(crawler.settings.get('DB_SETTINGS'))

    def open_spider(self, spider):
        # This runs when the spider starts
        self.connection = psycopg2.connect(**self.db_settings)
        self.cursor = self.connection.cursor()

    def close_spider(self, spider):
        # This runs when the spider finishes
        self.connection.close()

    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # Build our SQL query
        sql = "INSERT INTO scraped_quotes (text, author) VALUES (%s, %s)"
        params = (adapter['text'], adapter['author'])
        
        try:
            self.cursor.execute(sql, params)
            self.connection.commit()
        except Exception as e:
            # If it fails, undo the changes
            self.connection.rollback()
            spider.logger.error(f"Error saving to Postgres: {e}")
            
        return item

> [!CAUTION]
> **Scrapy Doc Gap: The Postgres Connection Trap**
> One of the most common mistakes is opening and closing a database connection inside the `process_item` method. 
> 
> In PostgreSQL, a connection is a heavy operation that starts a new process on the server. If your spider scrapes 10 items per second, you’ll be asking Postgres to create 600 processes a minute. This will quickly exhaust the `max_connections` limit and crash your database. Always open the connection once in `open_spider` and close it once in `close_spider`.
```python
    def process_item(self, item, spider):
        # ... standard sync implementation ... 
        # (See "Twisted Adbapi" section below for the pro version)
        return item
```

> [!CAUTION]
> **Scrapy Doc Gap: The Postgres Connection Trap**
> One of the most common mistakes is opening and closing a database connection inside the `process_item` method. 
> 
> In PostgreSQL, a connection is a heavy operation that starts a new process on the server. If your spider scrapes 10 items per second, you’ll be asking Postgres to create 600 processes a minute. This will quickly exhaust the `max_connections` limit and crash your database. Always open the connection once in `open_spider` and close it once in `close_spider`.

## Step 4: Configuration

Finally, we need to put our database "address" and "keys" into `settings.py`:

```python
# settings.py
DB_SETTINGS = {
    'host': 'localhost',
    'database': 'my_scrape_db',
    'user': 'postgres',
    'password': 'yourpassword',
    'port': 5432,
}

ITEM_PIPELINES = {
    'my_project.pipelines.PostgresPipeline': 300,
}
```

## Handling Connection Pooling

As your spiders get more complex, creating a new connection for every single item becomes slow. In a production environment, we use **Connection Pooling**. This is like having a "waiting room" of open connections that your spiders can reuse.

While you can write this manually, most professionals use **ORM** systems (which we'll learn in the next chapter) or libraries like `psycopg2.pool` to handle this automatically.

## Handling Duplicates with "UPSERT"

One of the coolest features of modern SQL is the **UPSERT** (Update or Insert). It means: "Try to insert this item, but if it already exists (the ID is a duplicate), just update the old one instead."

```sql
INSERT INTO quotes (url, price, last_seen) 
VALUES (%s, %s, NOW())
ON CONFLICT (url) 
DO UPDATE SET 
    price = EXCLUDED.price,
    last_seen = NOW();
```
This is the ultimate way to make sure your database always has the most recent information without filling it with junk duplicates.

## Non-Blocking I/O with Twisted Adbapi

The standard `psycopg2` example above has a fatal flaw: `cursor.execute()` is **blocking**. If your database takes 500ms to insert a row, your entire spider freezes for 500ms.

Scrapy works best when we use the **Twisted Adbapi** wrapper. It creates a pool of threads specifically for database logic so the spider can keep crawling while the database is writing.

```python
from twisted.enterprise import adbapi

class AsyncPostgresPipeline:
    def __init__(self, db_pool):
        self.db_pool = db_pool

    @classmethod
    def from_crawler(cls, crawler):
        db_settings = crawler.settings.get('DB_SETTINGS')
        db_pool = adbapi.ConnectionPool('psycopg2', **db_settings)
        return cls(db_pool)

    def process_item(self, item, spider):
        # Run the insert "in the background"
        query = self.db_pool.runInteraction(self.do_insert, item)
        query.addErrback(self.handle_error, item, spider)
        return item

    def do_insert(self, cursor, item):
        # This runs in a separate thread!
        cursor.execute(
            "INSERT INTO quotes (text, author) VALUES (%s, %s)",
            (item['text'], item['author'])
        )

    def handle_error(self, failure, item, spider):
        spider.logger.error(f"Error saving item: {failure}")
```
This pattern is **critical** for high-performance spiders.

## Chapter Summary

**What we covered:**
- PostgreSQL is the production standard for SQL databases in scraping.
- `psycopg2` is the Python library we use to talk to Postgres.
- A Scrapy Pipeline can handle the entire lifecycle of a database connection: opening it, saving items, and closing it.
- **Transactions** (`commit` and `rollback`) ensure your data is saved safely.
- **UPSERT** logic is the best way to handle recurring crawls and duplicates.

**Key code:**
```python
self.connection = psycopg2.connect(host=..., database=..., ...)
```

**Previous chapter:**
[Chapter 19: Introduction to Databases](./chapter_19_introduction_to_databases.md)

**Next chapter:**
Writing raw SQL queries like `INSERT INTO...` can get tedious and error-prone. In the next chapter, we're going to learn about **ORM (Object-Relational Mapping)** the professional way to treat your database tables like Python objects, making your code cleaner and easier to maintain.

---

**End of Chapter 20**
