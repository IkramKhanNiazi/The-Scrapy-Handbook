# Chapter 22: ORM Integration with Scrapy

Think about the first time you have to manage a complex dataset involving "Products" and "Reviews." In your raw SQL spider, you have to save the product first, then get the "Product ID" that the database created, and then manually pass that ID into another query to save the reviews.

It can be a huge mess of callbacks and nested logic. You're constantly checking if the product already exists, trying to handle database errors, and worrying that a single mistake will leave you with "orphan" reviews that aren't connected to any product. You might feel like you're trying to build a complex clock using only a hammer and a few nails. Your code becomes hard to read, impossible to test, and it might break every time the website changes its layout even slightly.

You'll feel like the "logic" of your project is being swallowed by the "details" of the database.

Then, you integrate **SQLAlchemy** into your Scrapy pipeline.

It's like finally having a blueprint for the clock. You won't have to worry about IDs or raw SQL anymore. You can just create a `Product` object and a `Review` object, tell SQLAlchemy they are connected, and call `session.commit()`. SQLAlchemy handles the IDs, the relationships, and the transactions perfectly. Your hundreds of lines of messy "database code" can turn into one elegant pipeline that you can reuse for every project. It's the moment you stop being just a "scraper" and start being a "data architect."

In this chapter, you're going to learn how to build that architecture.

---

## Introduction

In the previous chapter, we learned the basics of **ORM (Object-Relational Mapping)**. Now, we're going to put that knowledge to work in a real Scrapy project.

We're going to build a professional-grade database integration. We'll learn how to organize our Models, how to build a robust Item Pipeline that uses SQLAlchemy, and how to handle advanced scenarios like **UPSERTs** and multi-table relationships. This is the setup that most professional data teams use to ensure their scraped data is clean, consistent, and easy to query.

## Step 1: Organizing Your Project

In a professional project, we keep our database logic separate from our spider logic. I recommend adding a new file called `models.py` to your Scrapy project folder (next to `items.py` and `pipelines.py`).

```python
# models.py
from sqlalchemy import Column, Integer, String, Text, DateTime, create_engine
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class Quote(Base):
    __tablename__ = 'quotes'
    
    id = Column(Integer, primary_key=True)
    text = Column(Text, nullable=False)
    author = Column(String(255))
    scraped_at = Column(DateTime, default=datetime.utcnow)
```

## Step 2: The SQLAlchemy Pipeline

Now, let's build the "bridge" in `pipelines.py`. This pipeline will handle the connection, the session, and the saving of each item.

```python
from sqlalchemy.orm import sessionmaker
from my_project.models import Quote, create_engine, Base

class SQLAlchmeyPipeline:
    def __init__(self, db_url):
        self.engine = create_engine(db_url)
        # Create the tables if they don't exist
        Base.metadata.create_all(self.engine)
        self.Session = sessionmaker(bind=self.engine)

    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler.settings.get('DATABASE_URL'))

    def open_spider(self, spider):
        # Create tables (if they don't exist)
        Base.metadata.create_all(self.engine)

    def process_item(self, item, spider):
        # We use a "Scoped Session" for thread safety or just a new session per item
        session = self.Session()
        try:
            # ... business logic ...
            session.commit()
        except:
            session.rollback()
            raise
        finally:
            session.close()
        return item
```

> [!WARNING]
> **Scrapy Doc Gap: The Session Scope**
> Common Question: "Should I keep one `session` open for the whole spider?"
> **Answer: NO.**
> SQLAlchemy sessions are not thread-safe. Scrapy is multi-threaded. If two threads try to use the same session at the same time, your data will be corrupted.
> **Rule:** Create a fresh session for every `process_item` call, or use `scoped_session` from `sqlalchemy.orm`.

## Async ORM Support

If you are using the `AsyncioSelectorReactor` (Chapter 18), you can use SQLAlchemy's **Async Extension** to write non-blocking pipelines!

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

class AsyncORMPipeline:
    def __init__(self, db_url):
        self.engine = create_async_engine(db_url, echo=False)
        self.async_session = sessionmaker(
            self.engine, expire_on_commit=False, class_=AsyncSession
        )

    async def process_item(self, item, spider):
        async with self.async_session() as session:
            async with session.begin():
                # logic here
                session.add(Quote(**item))
                # Auto-commit happens at end of 'async with' block
        return item
```
        
        # Create a new Quote object from our item data
        quote = Quote(
            text=item.get('text'),
            author=item.get('author')
        )

        try:
            session.add(quote)
            session.commit()
        except:
            session.rollback()
            raise
        finally:
            session.close()

> [!IMPORTANT]
> **Scrapy Doc Gap: The session.close() Essential**
> This is a small line of code with a huge impact. Every time you open a session with `self.Session()`, you are taking a connection from the "pool." 
> 
> If you forget to call `session.close()` in the `finally` block, that connection remains "checked out" forever. Eventually, your spider will have used up every available connection in the pool, and any new request will wait forever or crash. Always ensure your sessions are closed, even if an error occurs earlier in the method!

        return item
```

## Update vs. Insert: The Professional way

In Chapter 20, we talked about "UPSERTs." In SQLAlchemy, we use a pattern called `merge()` or `on_conflict_do_update`. This ensures that if you scrape the same quote tomorrow, you don't save a duplicate; you just update the existing record.

```python
# Professional Update Logic
# Professional Update Logic
stmt = select(Quote).where(Quote.text == item['text'])
existing_quote = session.execute(stmt).scalar_one_or_none()

if existing_quote:
    # Update existing
    existing_quote.author = item['author']
    existing_quote.scraped_at = datetime.utcnow()
else:
    # Insert new
    session.add(Quote(**item))

session.commit()
```

> [!TIP]
> **Performance Tip: The Check-First Strategy**
> Doing a `SELECT` before every `INSERT` doubles your database traffic.
> For massive speed, use the Postgres-specific `INSERT ... ON CONFLICT` feature built into SQLAlchemy's `dialects.postgresql` module. It does the check-and-update in a single query!

## Handling Relationships

What if an "Author" has many "Quotes"? 
Instead of saving them separately, we can define a relationship in our `models.py`:

```python
class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String(255), unique=True)
    quotes = relationship("Quote", back_populates="author_rel")

class Quote(Base):
    # ... other fields ...
    author_id = Column(Integer, ForeignKey('authors.id'))
    author_rel = relationship("Author", back_populates="quotes")
```

Now, in your pipeline, you can just do:
```python
quote.author_rel = Author(name="Albert Einstein")
session.add(quote)
```
SQLAlchemy will automatically save the author, get the new ID, and link it to the quote for you.

## Connection Pooling and Performance

SQLAlchemy includes **Connection Pooling** by default. This means it keeps a few connections open even when it's not currently saving an item. This makes your scraper much faster because it doesn't have to perform the expensive "handshake" with the database every single time.

You can tune this in your `create_engine` call:
```python
engine = create_engine(db_url, pool_size=10, max_overflow=20)
```

## Chapter Summary

**What we covered:**
- Professionals keep their database models in a dedicated `models.py` file.
- The **SQLAlchemy Pipeline** handles the lifecycle of the session (open, commit, rollback, close).
- **CRUD** logic is much cleaner and more readable when using an ORM.
- **Relationships** (like Author → Quotes) are handled automatically by SQLAlchemy using Foreign Keys.
- **Connection Pooling** is built-in, providing a massive performance boost for high-speed scrapers.

**Key code:**
```python
session = self.Session()
session.add(my_object)
session.commit()
```

**Previous chapter:**
[Chapter 21: Understanding ORM](./chapter_21_understanding_orm.md)

**Next chapter:**
SQL databases are excellent for structured data. But what if your data is messy, nested, or constantly changing? In the next chapter, we're going to explore **NoSQL Databases** learning how to use **MongoDB** to handle flexible data structures with ease.

---

**End of Chapter 22**
