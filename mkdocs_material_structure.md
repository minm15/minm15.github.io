# MkDocs Material Technical Notes Website Structure

## Goal

Create a public technical notes website using MkDocs Material.

The website should be maintained mainly by editing Markdown files. The generated HTML should not be manually edited.

## Basic Project Structure

```text
my-notes/
├── mkdocs.yml
├── docs/
│   ├── index.md
│   ├── linux/
│   │   ├── index.md
│   │   ├── memory-management.md
│   │   └── file-system.md
│   ├── cpp/
│   │   ├── index.md
│   │   └── stl.md
│   ├── ros/
│   │   ├── index.md
│   │   └── ros2-bag.md
│   └── assets/
│       └── images/
│           └── example.png
├── requirements.txt
└── .github/
    └── workflows/
        └── deploy.yml
```

## Core Concepts

MkDocs projects are controlled by `mkdocs.yml`.

All documentation source files should be placed under the `docs/` directory.

Each article is a normal Markdown file.

For example:

```text
docs/linux/memory-management.md
```

will become a web page in the Linux section.

The root homepage should be:

```text
docs/index.md
```

## mkdocs.yml

The `mkdocs.yml` file controls:

- site name
- site URL
- theme
- navigation order
- Markdown extensions
- plugins

Example:

```yaml
site_name: My Technical Notes
site_url: https://USERNAME.github.io

theme:
  name: material

nav:
  - Home: index.md
  - Linux:
      - Overview: linux/index.md
      - Memory Management: linux/memory-management.md
      - File System: linux/file-system.md
  - C++:
      - Overview: cpp/index.md
      - STL: cpp/stl.md
  - ROS:
      - Overview: ros/index.md
      - ROS2 Bag: ros/ros2-bag.md

markdown_extensions:
  - admonition
  - toc:
      permalink: true
  - pymdownx.highlight
  - pymdownx.superfences
```

Important rule:

Paths inside `nav` are relative to the `docs/` directory, not the project root.

So this is correct:

```yaml
- Memory Management: linux/memory-management.md
```

This is incorrect:

```yaml
- Memory Management: docs/linux/memory-management.md
```

## Adding a New Article

To add a new article, create a new Markdown file under `docs/`.

Example:

```bash
mkdir -p docs/linux
touch docs/linux/page-cache.md
```

Then write the article:

```md
# Page Cache

Page cache is the Linux kernel mechanism used to cache file-backed data in memory.

## Basic Idea

- File data is managed in page-sized or folio-sized units.
- Read/write operations may use the page cache.
- mmap-based file access may also use the page cache.
```

Then add the page to `mkdocs.yml`:

```yaml
nav:
  - Linux:
      - Overview: linux/index.md
      - Page Cache: linux/page-cache.md
```

After that, the page will appear in the website navigation.

## Modifying an Article

To modify an article, directly edit the corresponding Markdown file.

Example:

```bash
vim docs/linux/page-cache.md
```

No generated HTML file needs to be edited manually.

## Deleting an Article

To delete an article, remove the Markdown file:

```bash
rm docs/linux/page-cache.md
```

Then remove the corresponding entry from `mkdocs.yml`:

```yaml
- Page Cache: linux/page-cache.md
```

If the file is deleted but still listed in `nav`, the build may fail or produce navigation errors.

## Adding Images

Place images under:

```text
docs/assets/images/
```

Example:

```text
docs/assets/images/page-cache-flow.png
```

Then reference the image in Markdown.

If the article is located at:

```text
docs/linux/page-cache.md
```

use:

```md
![Page cache flow](../assets/images/page-cache-flow.png)
```

If the article is located at:

```text
docs/index.md
```

use:

```md
![Example](assets/images/example.png)
```

## Local Preview

Install dependencies:

```bash
pip install -r requirements.txt
```

Run local development server:

```bash
mkdocs serve
```

Then open:

```text
http://127.0.0.1:8000
```

When Markdown files or `mkdocs.yml` are changed, MkDocs will rebuild the site automatically.

## requirements.txt

Example:

```txt
mkdocs
mkdocs-material
pymdown-extensions
```

## GitHub Pages Deployment

The normal update workflow should be:

```bash
git add .
git commit -m "Update notes"
git push
```

GitHub Actions should build the MkDocs site and deploy the generated static files to GitHub Pages.

The user should only maintain:

```text
mkdocs.yml
docs/**/*.md
docs/assets/**
```

The generated site output should not be committed unless the deployment strategy explicitly requires it.

## Article URL Mapping

Example mapping:

```text
docs/index.md
→ https://USERNAME.github.io/

docs/linux/page-cache.md
→ https://USERNAME.github.io/linux/page-cache/

docs/cpp/stl.md
→ https://USERNAME.github.io/cpp/stl/
```

## Maintenance Summary

Adding an article:

```text
1. Create a new .md file under docs/
2. Add it to mkdocs.yml nav
3. Commit and push
```

Modifying an article:

```text
1. Edit the corresponding .md file
2. Commit and push
```

Deleting an article:

```text
1. Delete the .md file
2. Remove it from mkdocs.yml nav
3. Commit and push
```

Adding an image:

```text
1. Put the image under docs/assets/images/
2. Reference it from Markdown using a relative path
3. Commit and push
```
