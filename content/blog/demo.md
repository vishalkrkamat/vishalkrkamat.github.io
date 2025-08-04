+++
title = "Building My Personal Website with Zola"
date = 2025-06-02
description = "A walkthrough of how I built a minimal, fast, no-BS dev site using Zola."
draft = false

[extra]
toc = true

[taxonomies] 
tags = ["zola", "static site", "personal branding"]
+++

Welcome to my first proper blog post!  
This site is built with [Zola](https://www.getzola.org) — a **fast, minimal** static site generator written in Rust.  
I’ll walk you through how I set everything up, what I learned, and how you can do the same.

---

## 🚀 Why Zola?

I chose Zola because:

- It has **zero dependencies**
- It builds fast, even with many pages
- Templating with [Tera](https://tera.netlify.app/docs/) is clean
- Markdown support is rich
- You own everything — no bloat, no nonsense

---

## 📁 Project Structure

After running `zola init`, I structured my site like this:

```text
content/
├── _index.md          ← homepage
├── about.md
├── projects/
│   └── _index.md
├── blog/
│   ├── _index.md
│   └── demo-post.md  ← you are here!
```
