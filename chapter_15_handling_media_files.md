# Chapter 15: Handling Media Files

Think back to the first time you try to build a "wallpaper downloader" for your desktop. You've found a beautiful website with thousands of high-resolution nature photos. You build a spider that grabs the direct URL for every image. You're proud of your list of ten thousand links.

Then you realize you have no idea how to actually *download* them.

You might try to use a simple Python script to download the images one by one. It works for the first fifty, and then your computer might start to crawl. You're downloading huge 4K images, and your script waits for each one to finish before starting the next. It could take three days to finish. Even worse, if your internet connection flickers for a second, the script could crash, and you'd have to figure out which images you've already downloaded so you don't start over.

It can feel as if you're trying to empty a swimming pool with a teaspoon. You're doing all the manual work of "saving" files, which is exactly the kind of thing Scrapy is supposed to do for you.

You might spend an entire afternoon writing a complex folder management system to organize images by category. You write code to check if a file exists, code to handle "broken" image links, and code to generate thumbnails. Your "simple" project could turn into a thousand-line monster that is mostly just file-handling code, and the actual scraping part is buried in the middle.

That's when you discover **Media Pipelines**.

It's like switching from a teaspoon to a high-pressure pump. You'll realize that Scrapy already has a professional, robust system for downloading millions of files. It handles the folders, the duplicates, the thumbnails, and the errors automatically. You can replace hundreds of lines of messy Python code with just five lines of configuration in your `settings.py`.

In this chapter, you're going to learn how to let Scrapy do the heavy lifting of file management.

---

## Introduction

Web scraping isn't just about text. Often, the value is in the media: product photos, research PDFs, or document downloads. While you *could* write your own download logic, Scrapy provides specialized **Media Pipelines** to handle this for you.

In this chapter, we're going to explore the **ImagesPipeline** and the **FilesPipeline**. We'll learn how to set them up, how to customize where files are stored, and how to automatically generate thumbnails and handle download errors. By the end of this chapter, you'll be able to build a professional media-harvesting system.

## Designing for Media: The Workflow

When you download media, the workflow is slightly different than text:
1.  **Extract the URL:** Your spider finds the URL of the image or file.
2.  **Yield the URL:** You put that URL into a specific field in your Item (like `image_urls`).
3.  **Automatic Download:** Scrapy's pipeline sees that field, downloads the file, and replaces the URL with a local file path.

## Downloading Images: The ImagesPipeline

The `ImagesPipeline` is a specialized tool for handling pictures. It automatically avoids downloading the same image twice and can even check if an image is actually an image (and not a broken link).

### Step 1: Install Pillow
Scrapy uses a library called `Pillow` to process images.
```bash
pip install Pillow
```

### Step 2: Update Your Item
You need two specific fields in your `items.py`:
```python
class ProductItem(scrapy.Item):
    name = scrapy.Field()
    image_urls = scrapy.Field() # This must be a LIST of URLs
    images = scrapy.Field()     # This will hold the results (file path, checksum, etc.)
```

### Step 3: Enable the Pipeline
In your `settings.py`, you need to enable the pipeline and tell Scrapy where to save the files:
```python
# settings.py
ITEM_PIPELINES = {
    'scrapy.pipelines.images.ImagesPipeline': 1,
}
IMAGES_STORE = 'my_downloads/images'
```

### Cloud Storage (S3 / GCS / Azure)

Saving files to your laptop's hard drive is fine for testing, but in production, you want them in the cloud.
Scrapy's media pipelines support this natively!

**Amazon S3 / Minio:**
```python
# settings.py
AWS_ACCESS_KEY_ID = '...'
AWS_SECRET_ACCESS_KEY = '...'
IMAGES_STORE = 's3://my-bucket/images/'
```

**Google Cloud Storage:**
```python
# settings.py
IMAGES_STORE = 'gs://my-bucket/images/'
GCS_PROJECT_ID = 'my-project'
```

No code changes are needed in your spider!

### Thumbnails

The `ImagesPipeline` can automatically generate thumbnails for you. This is perfect if you are building an e-commerce scraper and need image previews.

```python
# settings.py
IMAGES_THUMBS = {
    'small': (50, 50),
    'big': (270, 270),
}
```
Scrapy will save the original image *plus* the two thumbnails in subfolders.
```

## Custom Image File Names

By default, Scrapy saves images with a long, random-looking name (a SHA1 hash). This is great for avoiding duplicates, but it's terrible for humans.

To give your images nice names (like `sony-camera-01.jpg`), you can create a custom pipeline that inherits from the `ImagesPipeline`:

```python
from scrapy.pipelines.images import ImagesPipeline

class CustomImagePipeline(ImagesPipeline):
    def file_path(self, request, response=None, info=None, *, item=None):
        # Use the product name to create the filename
        # item['name'] comes from your spider!
        return f"{item['name'].replace(' ', '_').lower()}.jpg"
```

> [!IMPORTANT]
> **Scrapy Doc Gap: Custom Filenames with file_path**
> When overriding the `file_path` method, remember that the `response` argument might be `None` if the image is being served from Scrapy’s internal cache. 
> 
> If your naming logic depends on the response headers (like content-type), your spider might crash when re-running a crawl. Always write your `file_path` logic to rely on the `item` or the `request` URL rather than the `response` for maximum stability!
> If your naming logic depends on the response headers (like content-type), your spider might crash when re-running a crawl. Always write your `file_path` logic to rely on the `item` or the `request` URL rather than the `response` for maximum stability!

### Passing Usage Rights (Headers)

Some websites protect images by checking the `Referer` or `User-Agent`. If you can see the image in the browser but Scrapy downloads a 0-byte file, this is why.

You can modify the request *inside* the pipeline:

```python
class HeaderInjectionPipeline(ImagesPipeline):
    def get_media_requests(self, item, info):
        for image_url in item.get('image_urls', []):
            yield scrapy.Request(
                image_url, 
                headers={'Referer': item['url']} # Trick the server
            )
```

> [!WARNING]
> **Scrapy Doc Gap: The "checksum" Trap**
> The `ImagesPipeline` uses the MD5 hash of the image content to detect duplicates. However, if two different products use the **same** placeholder image (e.g., "Image Coming Soon"), Scrapy will treat them as duplicates and only download the first one.
> 
> If you need to keep *every* image (even duplicates), you must override the `file_path` method to include the product ID in the filename, significantly reducing the chance of collision, or subclass the pipeline to disable checksum checking.

## Downloading Files

The `FilesPipeline` works almost exactly like the ImagesPipeline, but it doesn't try to process the content as an image. This is perfect for PDFs, ZIP files, or documents.

In your `settings.py`:
```python
FILES_STORE = 'my_downloads/files'
ITEM_PIPELINES = {
    'scrapy.pipelines.files.FilesPipeline': 1,
}
```

## PDF Handling

PDFs are one of the most common file types you'll encounter. While the `FilesPipeline` will download them for you, you might need to extract text from *inside* the PDF. 

We'll talk about specialized libraries like `PyPDF2` or `pdfminer` in later chapters, but for now, remember that your spider's job is just to find the link. The pipeline will ensure the PDF is sitting safely on your hard drive ready for the next step.

## Large File Considerations

If you are downloading thousands of 50MB files, you need to be careful.
*   **Concurrency:** Don't download 100 files at once if your internet is slow.
*   **Storage:** Make sure you have enough disk space!
*   **Timeouts:** Large files take longer to download. You may need to increase the `DOWNLOAD_TIMEOUT` in your settings.

## Handling Download Errors

Sometimes a link is dead (`404`) or a server is too slow. Scrapy's media pipelines handle this by logging the error and moving on to the next item. You can see which files failed by looking at the `images` or `files` field in your final output it will contain the "status" of each download.

## Chapter Summary

**What we covered:**
- Media Pipelines (Images and Files) automate the downloading and organization of files.
- The `ImagesPipeline` requires the `Pillow` library and a specific `image_urls` field.
- `IMAGES_STORE` tells Scrapy where to save your "loot."
- You can customize filenames by overriding the `file_path` method in a custom pipeline.
- Large files require careful management of timeouts and storage space.

**Key code:**
```python
# settings.py
IMAGES_STORE = 'path/to/valid/folder'
ITEM_PIPELINES = {'scrapy.pipelines.images.ImagesPipeline': 1}
```

**Previous chapter:**
[Chapter 14: JavaScript-Rendered Websites](./chapter_14_javascript_rendered_websites.md)

**Next chapter:**
We've covered the "how" of scraping any individual site. But how do we find *everything* on a site efficiently? In the next chapter, we're going to explore **Sitemaps and Efficient Crawling** learning how to use a website's own roadmap to scrape thousands of pages in minutes.

---

**End of Chapter 15**
