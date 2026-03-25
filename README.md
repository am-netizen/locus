# Locus — Jekyll blog

Site: [https://am-netizen.github.io/locus](https://am-netizen.github.io/locus).  
GitHub Pages builds this repo with Jekyll. There is **no theme**; layout and styles live in `_layouts/`.

---

## Publish a new post

### 1. Choose the filename (this sets the URL slug)

Create **`_posts/YYYY-MM-DD-<slug>.md`**.

- **`YYYY-MM-DD`** — post date (used in the URL and ordering).
- **`<slug>`** — everything after the date and the hyphen, without `.md`.  
  Example: `_posts/2025-03-22-aws-cloud-incident-response-exfilcola.md` → slug **`aws-cloud-incident-response-exfilcola`**.

The live post URL will look like:

`https://am-netizen.github.io/locus/2025/03/22/aws-cloud-incident-response-exfilcola/`  
(paths follow `_config.yml`: `permalink: /:year/:month/:day/:title/` and `baseurl: "/locus"`).

### 2. (Optional) Add files for that post under `assets/`

If you need images, downloads, queries, evidence, etc., put them in:

**`assets/<slug>/`**

Use the **same** `<slug>` as in the filename. Example:

```text
assets/aws-cloud-incident-response-exfilcola/
  attack-navigator/
  queries/
  evidence/
```

Nothing else is required at the repo root for post-specific files.

### 3. Front matter

At the top of the post file:

```yaml
---
layout: post
title: "Your post title"
tags: [Incident Response]
---
```

Optional lines (add only if you need them):

| Field | When to use |
|--------|-------------|
| `excerpt: "Short summary"` | Shown on the homepage when `show_excerpts: true` in `_config.yml`. If omitted, Jekyll uses the first paragraph. |

The **first** entry in `tags` is shown on the homepage and above the post title.

### 4. If you use `assets/<slug>/`: link files directly

Reference files with **`relative_url`** so `baseurl` (`/locus`) is applied on GitHub Pages:

```liquid
![Description]({{ '/assets/aws-cloud-incident-response-exfilcola/evidence/screenshot.png' | relative_url }})
```

```liquid
[Download layer]({{ '/assets/aws-cloud-incident-response-exfilcola/attack-navigator/layer.json' | relative_url }})
```

Posts with no extra files can skip this and use plain markdown text/content.

### 5. Body

Write the rest in **Markdown** (Kramdown).

### 6. Ship it

Commit and push to **`main`**. The homepage listing updates automatically; you do not edit `index.html` per post.

---

## Excerpts on the homepage

`_config.yml` has `show_excerpts: true`. Use `excerpt:` in front matter for a custom blurb, or rely on the first paragraph.

---

## Repo map

| Path | Role |
|------|------|
| `_posts/` | Post sources (`YYYY-MM-DD-slug.md`) |
| `_layouts/default.html` | Site chrome (header, footer, CSS) |
| `_layouts/post.html` | Post title block + content wrapper |
| `assets/<slug>/` | Static files for one post (optional) |
| `index.html` | Homepage (intro + `site.posts` loop) |
| `about.md` | About page |
| `_config.yml` | `title`, `url`, `baseurl`, `permalink`, etc. |

---

## Optional: preview locally

Add a `Gemfile` with the `github-pages` or `jekyll` gem, run `bundle install`, then:

```bash
bundle exec jekyll serve
```

Open the URL Jekyll prints (with this repo’s settings, often **`http://127.0.0.1:4000/locus/`**).
