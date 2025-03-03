# Chapter 4: Setting Up Your Environment

Think about the first time you try to install a new Python library. You're excited, you're ready, and you have a great idea for a project. You just type `pip install` and hit Enter, thinking you're doing exactly what the tutorial says.

Ten minutes later, your entire computer could be a mess.

It's easy to accidentally break a system tool that your laptop needs to run properly. One project might need an old version of a library, but another project has just updated it to a new version that isn't compatible. If your terminal starts spitting out red error messages like angry ghosts, you might spend the next four hours searching forums, trying to figure out why a "simple" installation turned your machine into a paperweight.

You might even feel like quitting in that moment. You'll think, "If I can't even install the tools, how am I ever going to write the code? Maybe I'm just not cut out for this."

Then you discover virtual environments. It's like someone has handed you a stack of clean, empty boxes. You can put one project in its own box, with its own specific tools, and it will never interfere with anything else. If you make a mistake in one box, you can just throw it away and start over without affecting the rest of your computer.

That discovery changes everything. It turns a chaotic mess into a professional development workflow.

In this chapter, you're going to learn how to build your own professional "box" so you can experiment with Scrapy with total confidence.

---

## Introduction

Before we can start building spiders, we need a solid foundation. In this chapter, we aren't just installing software. We are building a professional workspace where you can safely experiment, make mistakes, and build production-ready scrapers.

We'll start by making sure you have the right version of Python. Then, we'll talk about the most important tool in your arsenal: the **Virtual Environment**. Finally, we'll install Scrapy and set up your code editor so you're ready to start building your first real project.

## Why Virtual Environments Matter

Imagine you are a chef. You are making a delicate cake for a wedding, but you are also making a spicy chili for a party. If you use the same bowl for both without washing it, someone is going to be very unhappy.

Programming is the same. One project might need "Scrapy Version 2.0," but another might need "Scrapy Version 2.10." If you install them both on your whole computer at once, they will fight.

A **Virtual Environment** is your "clean bowl." It ensures:
1.  **Project Isolation:** What you do in one project stays in that project.
2.  **No "Dependency Hell":** You'll never break your computer by installing a new tool.
3.  **Portability:** It's much easier to share your work with others when your tools are organized.

## Installing Python

Scrapy is built on Python, so we need it installed on your machine. We recommend **Python 3.9** or newer.

### Windows Installation
1.  Go to [python.org](https://www.python.org/downloads/).
2.  Download the latest installer.
3.  **IMPORTANT:** When the installer opens, check the box that says **"Add Python to PATH."** If you miss this, your terminal won't know what "python" means.
4.  Click "Install Now."

### Mac Installation
Most Macs come with Python, but it's usually an old version.
1.  Go to [python.org](https://www.python.org/downloads/).
2.  Download the macOS installer and run it.
3.  Alternatively, if you use "Homebrew," just type `brew install python` in your terminal.

### Verifying Installation
Open your terminal (PowerShell on Windows, Terminal on Mac) and type:
```bash
python --version
```
(If that doesn't work, try `python3 --version`). 
If you see something like `Python 3.10.x`, you are ready to go!

## Understanding pip

Along with Python, you get a tool called **pip** (Python Install Package). Think of `pip` as an "App Store" for Python code. Whenever we want to add a third-party tool like Scrapy, we use `pip` to go and get it from the internet for us.

## Creating Your First Virtual Environment

Let's create a folder for our first project and set up our virtual environment inside it.

```bash
# Create a folder and go inside it
mkdir my_scraper_project
cd my_scraper_project

# Create the virtual environment
# On Mac/Linux:
python3 -m venv venv

# On Windows:
python -m venv venv
```

You'll see a new folder named `venv` in your project. This is your "clean bowl."

### Activating the Environment
You have to "step inside" the environment before you can use it.
```bash
# On Mac/Linux:
source venv/bin/activate

# On Windows:
venv\Scripts\activate
```
Once you do this, you should see `(venv)` appear at the beginning of your terminal prompt. This means it's working!

## Installing Scrapy

Now that you are inside your clean environment, we can install Scrapy without any fear.

```bash
pip install scrapy
```

> [!CAUTION]
> **Scrapy Doc Gap: Common Installation Nightmares**
> While `pip install scrapy` sounds simple, it often fails on modern Macs or Windows systems due to "Binary Dependencies." 
> 
> If you see errors about `lxml`, `cryptography`, or `Twisted`, don't panic. On Windows, the fastest fix is usually installing the "Build Tools for Visual Studio." On Mac, running `xcode-select --install` often solves the issue. The Scrapy docs assume your system is "ready," but in the real world, these missing libraries are the most common hurdle for new developers.

You'll see a lot of text fly by as `pip` downloads Scrapy and all the other tools it needs. Once it's done, verify it by typing:
```bash
scrapy version
```

## Setting Up Your Code Editor

You can write code in a simple text editor, but a professional **IDE** (Integrated Development Environment) makes life fifty times easier.

I highly recommend **Visual Studio Code (VS Code)**. It's free, light, and has amazing Python support.

### VS Code Setup
1.  Install VS Code from [code.visualstudio.com](https://code.visualstudio.com/).
2.  Open your `my_scraper_project` folder in VS Code.
3.  Install the "Python" extension (search for it in the Extensions tab on the left).
4.  **Crucial Step:** Press `Command+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows) and type "Python: Select Interpreter." Choose the one that points to the `venv` folder you just created. This tells VS Code to use your clean "bow" instead of the computer's messy "kitchen."

## Creating Your First Project

Scrapy is organized into "Projects." To start your very first one, type this in your terminal (make sure you are inside `my_scraper_project` and your `venv` is active):

```bash
scrapy startproject my_spider
```

Scrapy will create a bunch of files for you. These are the blueprints for your spider factory, and we'll explore them in the next chapter.

## Your Development Workflow

From now on, your daily routine as a scraper will look like this:
1.  Open your terminal.
2.  Go to your project folder: `cd my_scraper_project`.
3.  Activate your environment: `source venv/bin/activate`.
4.  Start coding.

## Modern Development: Docker-Based Workflow

While virtual environments work great for local development, production scrapers often run in Docker containers. Let me show you both approaches.

### Why Docker for Scraping?

**Benefits:**
- Identical environment everywhere (dev, staging, production)
- Easy deployment to cloud services
- Isolates system dependencies (like browser drivers)
- Simple scaling (spin up multiple containers)

**When to use Docker:**
- Production deployments
- Team collaboration (everyone has same environment)
- Complex dependencies (Playwright, Selenium)
- Distributed crawling

### Basic Dockerfile for Scrapy

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \\
    gcc \\
    libxml2-dev \\
    libxslt1-dev \\
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

CMD ["scrapy", "crawl", "myspider"]
```

### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  scrapy:
    build: .
    volumes:
      - ./:/app
      - ./data:/app/data
    environment:
      - SCRAPY_SETTINGS_MODULE=myproject.settings
    command: scrapy crawl myspider

  # Optional: Add database
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: scraping
      POSTGRES_USER: scraper
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

**Run with:**
```bash
docker-compose up
```

> [!TIP]
> **Pro Pattern: Development vs Production Dockerfiles**
> Use multi-stage builds:
> ```dockerfile
> # Development stage
> FROM python:3.11 as development
> RUN pip install scrapy ipython pytest
> 
> # Production stage
> FROM python:3.11-slim as production
> RUN pip install --no-cache-dir scrapy
> # Smaller image, no dev tools
> ```

## Dependency Management: Beyond requirements.txt

### The Problem with requirements.txt

A simple `requirements.txt`:
```
scrapy
requests
```

This installs the latest versions. Six months later, Scrapy 3.0 is released with breaking changes, and your spider breaks.

### Solution 1: Pin Everything

```bash
# Generate pinned requirements
pip freeze > requirements.txt
```

Result:
```
Scrapy==2.11.0
Twisted==23.10.0
lxml==4.9.3
# ... 50 more dependencies
```

**Pros:** Reproducible builds
**Cons:** Hard to update, security vulnerabilities linger

### Solution 2: Use pip-tools (Recommended)

```bash
pip install pip-tools
```

Create `requirements.in`:
```
scrapy>=2.11,<3.0
scrapy-playwright>=0.0.30
python-dotenv
```

Compile:
```bash
pip-compile requirements.in
```

This generates `requirements.txt` with all dependencies pinned, but you only maintain the high-level ones.

**Update dependencies:**
```bash
pip-compile --upgrade requirements.in
```

### Solution 3: Poetry (Modern Approach)

```bash
pip install poetry
poetry init
poetry add scrapy
```

Creates `pyproject.toml`:
```toml
[tool.poetry.dependencies]
python = "^3.11"
scrapy = "^2.11"

[tool.poetry.dev-dependencies]
pytest = "^7.4"
ipython = "^8.12"
```

**Benefits:**
- Automatic dependency resolution
- Separates dev and production dependencies
- Lock file for reproducibility
- Virtual environment management built-in

> [!IMPORTANT]
> **Scrapy Doc Gap: Dependency Conflicts**
> Scrapy depends on Twisted, which can conflict with other async libraries. Common issues:
> 
> **Problem**: `asyncio` and Twisted don't play nice
> **Solution**: Use Twisted's asyncio reactor:
> ```python
> # settings.py
> TWISTED_REACTOR = 'twisted.internet.asyncioreactor.AsyncioSelectorReactor'
> ```
> 
> **Problem**: `lxml` binary wheels fail on some platforms
> **Solution**: Install system dependencies first:
> ```bash
> # Ubuntu/Debian
> sudo apt-get install libxml2-dev libxslt1-dev
> 
> # macOS
> brew install libxml2
> ```

## Platform-Specific Setup Guides

### macOS (M1/M2 Apple Silicon)

**Special considerations:**
```bash
# Some packages need Rosetta or native ARM builds
# Install Homebrew dependencies
brew install python@3.11 libxml2 libxslt

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install Scrapy
pip install scrapy

# If lxml fails, try:
CFLAGS="-I$(brew --prefix libxml2)/include" \\
LDFLAGS="-L$(brew --prefix libxml2)/lib" \\
pip install lxml
```

### Windows

**Challenges:**
- Some packages need Visual C++ compiler
- Path issues with scripts

**Solution:**
```powershell
# Install Python from python.org (not Microsoft Store)
# Add to PATH during installation

# Create virtual environment
python -m venv venv
venv\\Scripts\\activate

# Install Scrapy
pip install scrapy

# If you get compiler errors:
# Download and install Microsoft C++ Build Tools
# https://visualstudio.microsoft.com/visual-cpp-build-tools/
```

### Linux (Ubuntu/Debian)

**Recommended approach:**
```bash
# Install system dependencies
sudo apt-get update
sudo apt-get install -y \\
    python3.11 \\
    python3.11-venv \\
    python3-pip \\
    libxml2-dev \\
    libxslt1-dev \\
    libffi-dev \\
    libssl-dev

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install Scrapy
pip install --upgrade pip
pip install scrapy
```

## Production Environment Setup

### Environment Variables (Never Hardcode Secrets!)

**Bad:**
```python
# settings.py - DON'T DO THIS!
DATABASE_URL = "postgresql://user:password@localhost/db"
API_KEY = "sk_live_abc123"
```

**Good:**
```python
# settings.py
import os
from dotenv import load_dotenv

load_dotenv()  # Load from .env file

DATABASE_URL = os.getenv('DATABASE_URL')
API_KEY = os.getenv('API_KEY')

# Validate required variables
if not DATABASE_URL:
    raise ValueError("DATABASE_URL environment variable not set")
```

**Create `.env` file:**
```bash
# .env (add to .gitignore!)
DATABASE_URL=postgresql://user:password@localhost/db
API_KEY=sk_live_abc123
SCRAPY_LOG_LEVEL=INFO
```

### Production Checklist

**Before deploying:**

```python
# settings.py - Production configuration

# Logging
LOG_LEVEL = 'INFO'  # Not DEBUG
LOG_FILE = '/var/log/scrapy/spider.log'
LOG_FORMAT = '%(asctime)s [%(name)s] %(levelname)s: %(message)s'

# Performance
CONCURRENT_REQUESTS = 16
DOWNLOAD_DELAY = 0.5
AUTOTHROTTLE_ENABLED = True

# Reliability
RETRY_ENABLED = True
RETRY_TIMES = 3
DOWNLOAD_TIMEOUT = 30

# Monitoring
STATSMAILER_RCPTS = ['alerts@yourcompany.com']

# Security
ROBOTSTXT_OBEY = True
USER_AGENT = 'YourBot/1.0 (+https://yoursite.com/bot)'
```

### Health Checks

Create a health check endpoint:
```python
# healthcheck.py
import sys
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

def health_check():
    try:
        settings = get_project_settings()
        # Test database connection
        # Test Redis connection
        # etc.
        return 0
    except Exception as e:
        print(f"Health check failed: {e}")
        return 1

if __name__ == '__main__':
    sys.exit(health_check())
```

## Chapter Summary

**What we covered:**
- Virtual environments keep your projects isolated and your computer safe.
- We installed Python 3.9+ and learned how to verify it.
- We used `pip` to download libraries.
- We created and activated a `venv` to act as our "clean bowl."
- We installed Scrapy and set up VS Code to recognize our environment.

**Key Reference Guide:**
| Task | Command (Mac/Linux) | Command (Windows) |
| :--- | :--- | :--- |
| Create Env | `python3 -m venv venv` | `python -m venv venv` |
| Activate Env | `source venv/bin/activate` | `venv\Scripts\activate` |
| Install Scrapy | `pip install scrapy` | `pip install scrapy` |
| Check Version | `scrapy version` | `scrapy version` |

**Remember:**
- Always activate your `venv` before you start working.
- If you get a "Scrapy not found" error, it's almost always because your `venv` isn't active.
- Don't be afraid of the terminal; it's just a conversation with your computer.

**Previous chapter:**
[Chapter 3: How Websites Work](./chapter_03_how_websites_work.md)

**Next chapter:**
The lab is ready. The environment is clean. The tools are installed. It's time to build your first real piece of code. In the next chapter, we are going to write **Your First Scrapy Spider**.

---

**End of Chapter 4**
