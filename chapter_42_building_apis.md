# Chapter 42: Building APIs with Your Scraped Data

Think about the first time a client asks you to "make the data available." You've successfully scraped 100,000 products. They're sitting beautifully in your PostgreSQL database. The client says, "Great! Now our mobile app needs to access them."

You might freeze. Your entire focus has been on *getting* the data. You never thought about *serving* it. Do you just give them the database password? Do you export CSV files every day? Do you set up some kind of sync system? You realize you've built half of a data pipeline, the input half. The output half is completely missing.

That's when you discover **APIs**.

Building an API transforms your scraped data from a static collection into a living, queryable service. Your client's mobile app can search products in real-time. Their website can show live prices. Their internal tools can pull exactly the data they need, exactly when they need it.

In this chapter, you're going to complete the pipeline.

---

## Introduction

An **API (Application Programming Interface)** is a way for programs to talk to each other. When you scrape data with Scrapy, you're consuming APIs (or raw HTML). Now you're going to create one, turning your database into a service that others can query.

In this chapter, we're going to build a professional API using **FastAPI**, one of the fastest and most modern Python web frameworks. We'll create endpoints for querying products, implement pagination and filtering, add caching for performance, and even set up webhooks to push data to clients when it changes.

## Why FastAPI?

There are many Python web frameworks, but FastAPI stands out for scrapers:

| Feature | FastAPI | Flask | Django |
|---------|---------|-------|--------|
| Speed | Very fast (async native) | Moderate | Moderate |
| Automatic docs | Yes (Swagger/OpenAPI) | No | No |
| Type hints | Required (catches bugs) | Optional | Optional |
| Async support | Native | Extension | Limited |
| Learning curve | Easy | Easy | Steep |

For serving scraped data, FastAPI's speed and automatic documentation are game-changers.

## Getting Started

```bash
pip install fastapi uvicorn sqlalchemy asyncpg
```

Create a simple API:

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="Product Scraper API",
    description="Access scraped product data",
    version="1.0.0"
)

@app.get("/")
async def root():
    return {"message": "Welcome to the Product API"}

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

Run it:
```bash
uvicorn main:app --reload
```

Visit `http://localhost:8000/docs` for automatic interactive documentation!

## Connecting to Your Database

Set up async database access with SQLAlchemy:

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/scraper_db"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
Base = declarative_base()

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

```python
# models.py
from sqlalchemy import Column, Integer, String, Float, DateTime
from database import Base

class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(255))
    price = Column(Float)
    url = Column(String(500), unique=True)
    scraped_at = Column(DateTime)
```

## Building CRUD Endpoints

### Read All Products (with Pagination)

```python
# main.py
from fastapi import FastAPI, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from database import get_db
from models import Product

@app.get("/products")
async def get_products(
    skip: int = Query(0, ge=0, description="Number of items to skip"),
    limit: int = Query(20, ge=1, le=100, description="Number of items to return"),
    db: AsyncSession = Depends(get_db)
):
    query = select(Product).offset(skip).limit(limit)
    result = await db.execute(query)
    products = result.scalars().all()
    
    return {
        "skip": skip,
        "limit": limit,
        "count": len(products),
        "products": [
            {
                "id": p.id,
                "name": p.name,
                "price": p.price,
                "url": p.url
            }
            for p in products
        ]
    }
```

### Read Single Product

```python
from fastapi import HTTPException

@app.get("/products/{product_id}")
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    query = select(Product).where(Product.id == product_id)
    result = await db.execute(query)
    product = result.scalar_one_or_none()
    
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    
    return {
        "id": product.id,
        "name": product.name,
        "price": product.price,
        "url": product.url,
        "scraped_at": product.scraped_at
    }
```

### Search Products

```python
@app.get("/search")
async def search_products(
    q: str = Query(..., min_length=2, description="Search query"),
    min_price: float = Query(None, ge=0),
    max_price: float = Query(None),
    db: AsyncSession = Depends(get_db)
):
    query = select(Product).where(Product.name.ilike(f"%{q}%"))
    
    if min_price is not None:
        query = query.where(Product.price >= min_price)
    if max_price is not None:
        query = query.where(Product.price <= max_price)
    
    result = await db.execute(query.limit(50))
    products = result.scalars().all()
    
    return {"query": q, "count": len(products), "products": products}
```

## Response Models (Pydantic)

For clean, validated responses, define Pydantic models:

```python
# schemas.py
from pydantic import BaseModel
from datetime import datetime
from typing import List, Optional

class ProductBase(BaseModel):
    name: str
    price: float
    url: str

class ProductResponse(ProductBase):
    id: int
    scraped_at: datetime
    
    class Config:
        from_attributes = True

class ProductListResponse(BaseModel):
    skip: int
    limit: int
    count: int
    products: List[ProductResponse]
```

Use in endpoints:
```python
@app.get("/products", response_model=ProductListResponse)
async def get_products(...):
    ...
```

## Adding Caching with Redis

Don't hit the database for every request:

```python
import aioredis
import json

redis = aioredis.from_url("redis://localhost")

@app.get("/products/{product_id}")
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    # Check cache first
    cache_key = f"product:{product_id}"
    cached = await redis.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    # Cache miss - get from database
    product = await fetch_product_from_db(product_id, db)
    
    if product:
        # Cache for 5 minutes
        await redis.set(cache_key, json.dumps(product), ex=300)
    
    return product
```

> [!TIP]
> **Scrapy Doc Gap: Invalidating Cache on New Scrapes**
> When your Scrapy spider updates data, your API cache becomes stale. The cleanest pattern is to:
> 1. Have your Scrapy pipeline publish to a Redis pub/sub channel when items are updated
> 2. Have your API subscribe to that channel and invalidate relevant cache keys
> 
> This ensures your API always serves fresh data within seconds of a new scrape.

## Rate Limiting

Protect your API from abuse:

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/products")
@limiter.limit("100/minute")
async def get_products(request: Request, ...):
    ...
```

## API Authentication

Protect sensitive endpoints:

```python
from fastapi import Security
from fastapi.security import APIKeyHeader

API_KEY_HEADER = APIKeyHeader(name="X-API-Key")
VALID_API_KEYS = {"secret-key-123", "another-key-456"}

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)):
    if api_key not in VALID_API_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key

@app.get("/admin/stats")
async def get_stats(api_key: str = Depends(verify_api_key)):
    return {"total_products": 100000, "last_scrape": "2026-02-02"}
```

## Webhooks: Pushing Data to Clients

Instead of clients polling your API, push updates to them:

```python
import httpx
from typing import List

# Store webhook URLs (in production, use a database)
webhook_subscribers: List[str] = []

@app.post("/webhooks/subscribe")
async def subscribe_webhook(url: str, api_key: str = Depends(verify_api_key)):
    webhook_subscribers.append(url)
    return {"message": f"Subscribed {url}"}

async def notify_subscribers(event: str, data: dict):
    async with httpx.AsyncClient() as client:
        for url in webhook_subscribers:
            try:
                await client.post(url, json={"event": event, "data": data})
            except Exception as e:
                print(f"Failed to notify {url}: {e}")

# Call this from your Scrapy pipeline when prices drop
async def on_price_drop(product_id: int, old_price: float, new_price: float):
    await notify_subscribers("price_drop", {
        "product_id": product_id,
        "old_price": old_price,
        "new_price": new_price
    })
```

## Dockerizing Your Stack

Package everything together:

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db/scraper
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: scraper
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7
  
  scrapy:
    build: ./scraper
    depends_on:
      - db
    # Run on schedule with cron or external scheduler

volumes:
  postgres_data:
```

## The Complete Data Pipeline

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Scrapy    │───▶│  PostgreSQL  │◀───│   FastAPI   │
│   Spider    │    │   Database   │    │     API     │
└─────────────┘    └──────────────┘    └─────────────┘
       │                  │                   │
       │                  │                   │
       ▼                  ▼                   ▼
  Fetch Data         Store Data         Serve Data
  Clean Data         Index Data         Cache Data
  Validate           Deduplicate        Paginate
```

## Chapter Summary

**What we covered:**
- FastAPI is ideal for serving scraped data (fast, async, auto-documented).
- Pagination and filtering prevent overwhelming clients with data.
- Pydantic models ensure consistent, validated API responses.
- Redis caching dramatically improves performance for repeated queries.
- Rate limiting protects your API from abuse.
- API keys provide simple authentication for protected endpoints.
- Webhooks push updates to clients when data changes.
- Docker Compose packages your entire stack for deployment.

**Key code:**
```python
@app.get("/products")
async def get_products(skip: int = 0, limit: int = 20, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Product).offset(skip).limit(limit))
    return result.scalars().all()
```

**Previous chapter:**
[Chapter 41: Debugging and Profiling Spiders](./chapter_41_debugging_and_profiling.md)

**Next chapter:**
You've mastered the complete cycle: scraping, storing, and serving data. In the final section, we'll discuss **Legal and Ethical Considerations**, ensuring your powerful skills are used responsibly.

---

**End of Chapter 42**
