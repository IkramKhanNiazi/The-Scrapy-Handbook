# Chapter 10: Understanding Item Pipelines

Think about the day a simple scraping project turns into a huge data problem. You've built a spider to collect real estate listings, and for the first few hours, it works perfectly. But when you check the database the next morning, you find that you've saved three hundred copies of the exact same apartment listing.

Even worse, some of the listings have no price, and some are missing the address entirely. Your database is full of "garbage" data that is useless for anyone looking for a home. You've saved so many duplicates that your search filters are showing the same house over and over again, like a broken mirror.

You might feel like an assembly line worker who was so focused on putting boxes in the truck that he didn't notice half the boxes were empty and the other half were duplicates of the same item. You're "shipping" poor quality data to your users, and they might lose trust in your application.

If you try to fix it inside your spider, you might add complex checks to see if you've already seen a URL, and write hundreds of lines of `if` statements to validate every field. Your spider becomes a "spaghetti" mess of code that is impossible to read or debug. You're trying to do quality control right in the middle of the production line, and it slows everything down.

That's when you discover **Item Pipelines**.

It's like adding a professional inspection station at the end of the assembly line. You can move all your validation and duplicate checking to a separate, dedicated space. Your spider can go back to being a fast, clean "picker," and the pipeline will handle the "quality control." If a piece of data isn't good enough, the pipeline will just toss it in the bin before it ever reaches the database.

In this chapter, you're going to learn how to build your own quality control station.

---

## Introduction

In Chapter 8, we learned how to structure our data with Items. In Chapter 9, we learned how to clean individual strings. Now, we're going to talk about the final stage: **Item Pipelines**.

A pipeline is a simple Python class that processes your item after it has been extracted by the spider. Think of it as the "Assembly Line" that your data travels down before it is finally packed away in a file or a database. In this chapter, we'll learn how to build pipelines that validate data, filter out duplicates, and save our results to different destinations.

## What are Pipelines?

Whenever your spider "yields" an item, Scrapy sends that item to the **Item Pipeline**. 

Each pipeline is made of one or more "components." The item goes into the first component, which can modify it, validate it, or even "drop" it (delete it). If the item survives the first component, it moves to the second, then the third, and so on.

## Creating Your First Pipeline

Pipelines are kept in the `pipelines.py` file of your project. Here is the basic structure:

```python
# pipelines.py
from itemadapter import ItemAdapter
from scrapy.exceptions import DropItem

class MyValidationPipeline:
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        
        # Check if a specific field exists
        if not adapter.get('title'):
            # If no title, we don't want this item!
            raise DropItem(f"Missing title in {item}")
        
        return item
```

### The `process_item` method
This is the heart of the pipeline. It takes two arguments:
1.  **`item`**: The data you just scraped.
2.  **`spider`**: The spider that found the item (useful if you want different rules for different spiders).

Every `process_item` method **MUST** return the item or raise a `DropItem` exception.

> [!IMPORTANT]
> **Scrapy Doc Gap: The DropItem Finality**
> One thing the official docs don't always highlight is that `DropItem` is **final**. If you have five pipelines and the first one (Priority 100) raises `DropItem`, the item is destroyed immediately. It will *not* be seen by the other four pipelines. 
> 
> This is why your **Priority** numbers are so important. If you want to save data to a database *only if it is valid*, your validation pipeline must have a lower number (higher priority) than your database pipeline. If the validation fails and drops the item, the database pipeline will never receive it, protecting your storage from junk data.

## Common Pipeline Use Cases

### 1. Data Validation
As we saw in the example above, you can check if mandatory fields are present. You can also check if a number is within a certain range (e.g., "Price must be greater than zero").

### 2. Duplicate Filtering
You can maintain a list (or a set) of IDs you've already seen. If a new item comes in with a duplicate ID, you drop it.

```python
class DuplicatesPipeline:
    def __init__(self):
        self.ids_seen = set()

    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        if adapter['id'] in self.ids_seen:
            raise DropItem(f"Duplicate item found: {item!r}")
        else:
            self.ids_seen.add(adapter['id'])
            return item
```

## Pipeline Configuration

Creating the class isn't enough; you have to tell Scrapy to use it. This happens in-your `settings.py` file.

```python
# settings.py
ITEM_PIPELINES = {
    'my_project.pipelines.MyValidationPipeline': 300,
    'my_project.pipelines.DuplicatesPipeline': 400,
}
```

### What is the number (300/400)?
This is the **Priority**. Scrapy runs pipelines in order from the lowest number to the highest.
*   The **Validation** pipeline has a priority of 300.
*   The **Duplicates** pipeline has a priority of 400.
*   This means we validate the item *before* we check if it's a duplicate.

## Multiple Pipelines

You can have as many pipelines as you want. A professional project might have:
1.  `PriceConverterPipeline` (Priority 100): Changes currency.
2.  `ValidationPipeline` (Priority 200): Checks for missing data.
3.  `PostgreSQLPipeline` (Priority 300): Saves the clean, valid data to a database.

This modularity makes your code incredibly easy to maintain. If you want to stop saving to the database and start saving to a file, you just change one line in your `settings.py`.

## Advanced Pipeline Features

### 1. The Pipeline Lifecycle (`open_spider` and `close_spider`)

Pipelines aren't just for processing individual items. They can also take actions when the spider starts and stops. This is perfect for opening and closing database connections.

```python
import sqlite3

class SQLitePipeline:
    def open_spider(self, spider):
        # Runs once when the spider starts
        self.connection = sqlite3.connect('my_data.db')
        self.cursor = self.connection.cursor()
        self.cursor.execute('CREATE TABLE IF NOT EXISTS items (id TEXT, price REAL)')

    def close_spider(self, spider):
        # Runs once when the spider finishes
        self.connection.commit()
        self.connection.close()

    def process_item(self, item, spider):
        # Runs for every single item
        self.cursor.execute('INSERT INTO items VALUES (?, ?)', 
                          (item['id'], item['price']))
        return item
```

### 2. Accessing Settings (`from_crawler`)

Hardcoding database passwords in your pipeline is a bad idea. Instead, read them from your `settings.py` using the `from_crawler` class method.

```python
class MongoPipeline:
    def __init__(self, items_collection):
        self.items_collection = items_collection

    @classmethod
    def from_crawler(cls, crawler):
        # Get values from settings.py
        collection_name = crawler.settings.get('MONGO_COLLECTION')
        return cls(collection_name)
    
    def process_item(self, item, spider):
        # Use self.items_collection here...
        return item
```

### 3. Async Pipelines (The Modern Way)

Since Scrapy 2.0, you can make `process_item` async! This is **huge** for databases. If your database takes 50ms to insert a row, a normal pipeline blocks the entire spider for that time. An async pipeline lets the spider continue crawling while the database works.

```python
import aiohttp

class AsyncApiPipeline:
    async def process_item(self, item, spider):
        # This await doesn't block the spider!
        async with aiohttp.ClientSession() as session:
            await session.post('https://api.mybackend.com/items', json=item)
        return item
```

### 4. Batch Processing (Optimization)

Inserting 1,000 items one by one is slow. Inserting 1,000 items in one "Batch" is fast.

```python
class BatchInsertPipeline:
    def __init__(self):
        self.batch = []
    
    def process_item(self, item, spider):
        self.batch.append(item)
        
        # If we have 1000 items, insert them all
        if len(self.batch) >= 1000:
            self._insert_batch()
            self.batch = []  # Clear the list
            
        return item

    def close_spider(self, spider):
        # Don't forget the leftovers!
        if self.batch:
            self._insert_batch()
            
    def _insert_batch(self):
        # Pseudo-code for bulk insert
        db.insert_many(self.batch)
```

## Chapter Summary

**What we covered:**
- Item Pipelines are the "quality control" stage of your scraper.
- The `process_item` method is where you validate or modify the data.
- The `DropItem` exception is used to discard low-quality or duplicate data.
- The `ITEM_PIPELINES` setting controls which pipelines run and in what order.
- Using pipelines keeps your spider code clean and focused on extraction.

**Key code:**
```python
# The basic pattern
def process_item(self, item, spider):
    if some_condition:
        return item
    else:
        raise DropItem("Error message")
```

**Previous chapter:**
[Chapter 9: Data Cleaning Best Practices](./chapter_09_data_cleaning_best_practices.md)

**Next chapter:**
Our data is validated and clean. Now, we need to get it out of Scrapy and into the real world. In the next chapter, we're going to explore **Exporting Your Data** mastering JSON, CSV, XML, and even uploading your results to the cloud.

---

**End of Chapter 10**
