# Chapter 21: Understanding ORM

Think about the day you have to change the structure of your database. You're using raw SQL queries, which means your code is full of strings like `INSERT INTO products (name, price, stock) VALUES (%s, %s, %s)`. You have these strings scattered across five different files, and you've even written some shared helper functions to manage them.

Then, the client decides they want to add a "category" field to every product.

You might feel as if you're trying to rebuild a car engine while the car is doing 70 mph on the freeway. You have to find every single SQL string in your project and manually add the new field. If you miss just one in a small cleanup script, it might crash when it runs at midnight because it's trying to send three pieces of data to a table that now expects four. Your database becomes inconsistent, your logs fill with errors, and you spend your entire Monday morning cleaning up the mess.

You might feel like you're fighting the database rather than using it. Your code feels "brittle," like a glass sculpture that would break if anyone walked too loudly near it.

That's when you discover **SQLAlchemy** and the world of **ORM**.

It's like switching from building with individual grains of sand to building with high-quality, interlocking bricks. Suddenly, you don't have to write strings of SQL at all. Your database tables aren't just "tables" anymore; they're Python classes. If you want to add a field, you just add it to the class in one place, and your whole project "knows" about it instantly. You can search for products, update prices, and handle complex relationships using nothing but standard Python code. It makes your project robust, flexible, and honestly much more fun to write.

In this chapter, you're going to learn how to treat your database like a collection of Python objects.

---

## Introduction

In the previous chapter, we learned how to talk to PostgreSQL using raw SQL. While that's powerful, it can become messy and error-prone as your project grows. This is where **ORM (Object-Relational Mapping)** comes in.

An ORM is a tool that lets you interact with your database using Python classes and objects instead of raw SQL strings. The most popular and powerful ORM for Python is **SQLAlchemy**.

In this chapter, we're going to learn the fundamentals of SQLAlchemy. We'll define "Models" for our data, learn how to perform "CRUD" operations (Create, Read, Update, Delete), and see how ORMs handle complex relationships between different tables effortlessly.

## What is ORM?

Think of an ORM as a **Translator**. 
*   In **SQL**, a "Table" is where your data lives.
*   In **Python**, a "Class" is where your logic lives.
An ORM "maps" the table to the class. Each row in the table becomes an "Object" (an instance of the class) in your Python code.

## Why Use ORM?

3.  **Flexibility:** If you want to switch from SQLite to PostgreSQL, an ORM handles most of that change for you with just a few lines of configuration.
4.  **Consistency:** Your data structure is defined in one place (the Model), which makes your codebase much easier to maintain.

## SQLAlchemy vs. Peewee

While SQLAlchemy is the "Enterprise Grade" choice, there is another popular ORM: **Peewee**.

| Feature | SQLAlchemy | Peewee |
| :--- | :--- | :--- |
| **Philosophy** | "Data Mapper" (Complex but powerful) | "Active Record" (Simple and Django-like) |
| **Async Support** | ✅ Great (Use `v1.4+` or `v2.0`) | ⚠️ Limited (Requires strict wrappers) |
| **Learning Curve**| Steep | Easy |
| **Best For** | Massive, complex projects | Smaller scripts, personal tools |

*We will focus on SQLAlchemy because it is the standard for job-ready Python developers.*

> [!NOTE]
> **SQLAlchemy 1.4 vs 2.0**
> In 2023, SQLAlchemy released version 2.0, which changed a lot of syntax.
> This book uses the **modern 2.0 style** (using `Session.execute(select(...))`).
> If you see old tutorials using `session.query(Model)`, know that is the "Old Style" (1.x).

> [!WARNING]
> **Scrapy Doc Gap: Sync ORMs in an Async World**
> Scrapy is an asynchronous engine (it can do many things at once), but standard SQLAlchemy is **synchronous** (it stops and waits for the database). 
> 
> If your database query takes 0.5 seconds, your whole spider stops moving for that half-second. Over thousands of items, this can slow your crawl to a crawl. For high-performance projects, you should use Scrapy's `twisted.enterprise.adbapi` or SQLAlchemy’s newer `async` support to prevent the database from "blocking" your spider’s speed.

## SQLAlchemy Fundamentals: Defining Models

To use SQLAlchemy, we first define a base class. Then, we create classes that represent our tables.

```python
from sqlalchemy import create_engine, String, Float
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = 'products'
    
    # Modern "Type Hint" style (SQLAlchemy 2.0)
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(255))
    price: Mapped[float] = mapped_column(nullable=True)
    
    def __repr__(self):
        return f"<Product(name='{self.name}', price={self.price})>"
```

### Connection Strings
The "engine" needs a URL to find the database.

*   **SQLite:** `sqlite:///crawls.db` (local file)
*   **PostgreSQL:** `postgresql+psycopg2://user:pass@localhost:5432/mydb`
*   **MySQL:** `mysql+pymysql://user:pass@localhost/mydb`

## Creating the Database Schema

Once you have your models defined, SQLAlchemy can build the database for you! You don't need to write a `CREATE TABLE` script.

```python
engine = create_engine('sqlite:///products.db')
Base.metadata.create_all(engine)
```

## CRUD Operations: Create, Read, Update, Delete

To interact with the database, we use a **Session**. Think of a session as a "transactional workspace."

### 1. Create (Insert)
```python
new_product = Product(name='Scrapy Pro Camera', price=1299.99)
session.add(new_product)
session.commit()
```

### 2. Read (Query)
```python
# Get all products
all_products = session.query(Product).all()

# Get products with a price over 500
expensive_products = session.query(Product).filter(Product.price > 500).all()
```

### 3. Update
```python
product = session.query(Product).filter_by(name='Scrapy Pro Camera').first()
product.price = 999.99
session.commit()
```

### 4. Delete
```python
session.delete(product)
session.commit()
```

## Relationships in ORM

This is where ORMs really shine. Imagine a "Blog" where one "Author" has many "Articles."

In raw SQL, you would have to write complex `JOIN` queries. In SQLAlchemy, you just define the relationship, and you can access the articles like a simple Python list:

```python
# Once defined, you can just do:
author = session.query(Author).first()
for article in author.articles:
    print(article.title)
```

## Chapter Summary

**What we covered:**
- **ORM** (Object-Relational Mapping) translates database tables into Python classes.
- **SQLAlchemy** is the industry standard ORM for Python.
- **Models** are Python classes that define the structure (columns and types) of your tables.
- **Sessions** are used to perform CRUD operations (Create, Read, Update, Delete).
- Using an ORM makes your code more readable, maintainable, and robust.

**Key code:**
```python
class MyModel(Base):
    __tablename__ = 'table_name'
    id = Column(Integer, primary_key=True)
```

**Previous chapter:**
[Chapter 20: SQL Databases with Scrapy](./chapter_20_sql_databases_with_scrapy.md)

**Next chapter:**
Now that we understand how to use SQLAlchemy, it's time to integrate it into our Scrapy projects. In the next chapter, we're going to build an **ORM Integration with Scrapy** creating a reusable, professional pipeline that handles database storage with total elegance.

---

**End of Chapter 21**
