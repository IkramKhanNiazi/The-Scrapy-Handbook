# Chapter 19: Introduction to Databases

Think about the first time you exceed the limits of a CSV file. You're scraping property listings for a real estate client, and your dataset has grown to over eight hundred thousand rows. You try to open the file in Excel to check your work, and your entire computer freezes. You wait ten minutes, and then the screen turns blue. Your machine simply can't handle the sheer weight of the text.

You might feel like an architect who has built a massive skyscraper but forgotten to build a foundation. All your data is sitting there, trapped in a file that is too big to read and too fragile to use.

You might try to be clever, splitting the CSV into a hundred smaller files. But then you have a new problem: if you want to find every house that has three bedrooms and a pool, you have to write a Python script that opens every single one of those hundred files, reads every row, and closes them again. It could take twenty minutes to answer a single question. You find yourself spending more time managing files than actually doing research.

You'll realize that "files" are for humans to look at, but "databases" are for computers to work with.

When you finally move your data into a database, everything changes. You can find that three-bedroom house with a pool in less than 0.1 seconds. You can update a single price without rewriting the whole file. You can connect your data to other tools effortlessly. It's like you've finally built that foundation, and now your skyscraper is solid, fast, and ready to grow even higher.

In this chapter, you're going to learn how to build your own foundation.

---

## Introduction

As your scraping projects grow, you'll eventually outgrow CSV and JSON files. Files are great for small tasks, but they are slow, fragile, and hard to search as your data reaches thousands or millions of records.

In this chapter, we're going to bridge the gap between "files" and "databases." We'll talk about why you need a database, the different types available (SQL vs. NoSQL), and we'll get our hands dirty with **SQLite** the simplest database in the world that is already installed on your computer.

## Files vs. Databases: The Core Difference

Think of a **CSV file** like a long, paper scroll. To find a specific item at the bottom, you have to unroll the whole thing. To change a word in the middle, you have to rewrite the entire scroll.

A **Database** is like a library with a highly organized catalog. You go to the librarian (the database engine), ask for exactly what you want, and they bring it to you instantly. It's built for speed, searching, and "transactions" (making sure your data is saved safely even if the power goes out).

## When to Use Databases

You should switch from files to a database when:
1.  **Your data is large:** Anything over 50,000 rows starts to become slow in a file.
2.  **You need to search:** Finding specific items based on filters (like "Price < 500") is much faster in a database.
3.  **You need to update:** If you are monitoring prices, it's easier to "Update" an existing row in a database than to rewrite a whole JSON file.
4.  **You have relationships:** If you have "Authors" who have many "Books," a database handles that connection beautifully.

> [!TIP]
> **Scrapy Doc Gap: DB vs. File for Small Scale**
> Beginners often feel like they "must" use a database to be professional. This isn't true. For projects with fewer than 5,000 items, Scrapy's built-in **FEEDS** (JSON/CSV) is often better. 
> 
> Files are stateless, which means if your script crashes, the file is still there and easy to read. Databases require a running server, connection settings, and maintenance. If your goal is a quick analysis or a one-time project, stick to files. Only reach for a DB when you need to query and update data frequently over time.

## Database Types Overview

### 1. Relational (SQL)
Think of these as highly organized spreadsheets that can talk to each other. They use a language called **SQL** (Structured Query Language).
*   **Examples:** SQLite, PostgreSQL, MySQL.
*   **Best for:** Most scraping tasks where your data has a clear, consistent structure.

### 2. NoSQL (Document-based)
These are more flexible and store data as "Documents" (similar to JSON).
*   **Examples:** MongoDB, CouchDB.
*   **Best for:** When your data structure is constantly changing or very messy.
*   **Also:** Key-Value stores (Redis) for high-speed caching.

| Feature | SQL (Relational) | NoSQL (Document) |
| :--- | :--- | :--- |
| **Structure** | Rigid Tables | Flexible JSON |
| **Scaling** | Vertical (Bigger Server) | Horizontal (More Servers) |
| **Scraping Use**| structured products, prices | messy tweets, raw HTML |
| **Complexity** | Higher (Schema design) | Lower (Dump and forget) |

### 3. Time Series Databases
Specialized explicitly for timestamped data (like stock prices or weather).
*   **Examples:** InfluxDB, TimescaleDB.
*   **Best for:** Monitoring price history for millions of items.

## SQLite: Your First Database

The best thing about SQLite is that it's "Serverless." It's just a single file on your computer, but it acts like a professional database.

### Creating your first database
You don't need to install anything! Python has SQLite built-in. Let's create a database for our quotes:

```python
import sqlite3

# Connect to the database (it will be created if it doesn't exist)
conn = sqlite3.connect('quotes.db')
cursor = conn.cursor()

# Create a table
cursor.execute('''
    CREATE TABLE IF NOT EXISTS quotes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        text TEXT,
        author TEXT
    )
''')

# Insert some data
cursor.execute('INSERT INTO quotes (text, author) VALUES (?, ?)', 
               ('Scraping is fun!', 'Me'))

# Save the changes and close
conn.commit()
conn.close()
```

### Visualizing Your Database

Working with SQL in the terminal can be dry.
**Tool Recommendation:** Download **"DB Browser for SQLite"**.
It's a free, open-source GUI that lets you open your `.db` file, browse the data like a spreadsheet, and run SQL queries manually. It is an essential tool for debugging your scraper's output.

## Schema Design for Scrapers

How should you organize your tables?

**Pattern 1: The Flat Table (Simple)**
Just one big table called `products`.
- `id`, `url`, `title`, `price`

**Pattern 2: The Normalized Schema (Professional)**
Separate the *Item* from the *Price* to track history.
- **Table: Products** (`id`, `url`, `title`) - *Static info*
- **Table: Prices** (`product_id`, `price`, `scraped_at`) - *Changing info*

This allows you to chart the price history of a product over time!

## Basic SQL Queries

Once your data is in the database, you use SQL to talk to it. Here are the "Big Four" commands:

1.  **SELECT:** "Show me the data."
    `SELECT * FROM quotes WHERE author = 'Albert Einstein';`
2.  **INSERT:** "Add new data."
    `INSERT INTO quotes (text, author) VALUES ('...', '...');`
3.  **UPDATE:** "Change existing data."
    `UPDATE quotes SET author = 'Einstein' WHERE id = 1;`
4.  **DELETE:** "Remove data."
    `DELETE FROM quotes WHERE author = 'Unknown';`

## Database Best Practices

*   **Indexes are Magic:** If your database gets slow when searching for "Author," you can add an **Index**. This is like the index at the back of a book it makes searches nearly instantaneous.
*   **Use Transactions:** Always use `conn.commit()` after you make changes. This ensures that either *all* your changes are saved or *none* of them are (if something crashes).
*   **One Table, One Topic:** Don't try to cram everything into one giant table. Have one table for "Quotes" and a different table for "Authors."

## Chapter Summary

**What we covered:**
- Databases are the "professional foundation" for large datasets.
- SQL Databases are structured like interconnected spreadsheets.
- NoSQL Databases are flexible like JSON collections.
- **SQLite** is a powerful, serverless database built into Python.
- You use **SQL** to query, insert, and update your data.
- Databases provide speed, reliability, and powerful search capabilities that files can't match.

**Key code:**
```python
import sqlite3
connection = sqlite3.connect('mydata.db')
```

**Previous chapter:**
[Chapter 18: Spider Performance and Optimization](./chapter_18_spider_performance_and_optimization.md)

**Next chapter:**
Now that we understand the basics of databases, it's time to connect them to Scrapy. In the next chapter, we're going to build a **PostgreSQL Integration** for your spiders moving your data from the terminal into a professional, production-ready database.

---

**End of Chapter 19**
