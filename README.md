# Locus — Jekyll blog

Personal site at [https://am-netizen.github.io/locus](https://am-netizen.github.io/locus). GitHub Pages builds the site from this repo with Jekyll (no theme; layouts live in `_layouts/`).

## Publishing a new post

1. Create a file under `_posts/` named **`YYYY-MM-DD-short-title.md`**  
   Example: `_posts/2025-03-22-cloud-breach-notes.md`  
   The date is the post date; the slug becomes part of the URL.

2. Put this **front matter** at the very top (between the `---` lines):

   ```yaml
   ---
   layout: post
   title: "Your post title"
   tags: [Incident Response]
   ---
   ```

   - **layout:** must be `post` for normal articles.
   - **title:** shown on the homepage and at the top of the post.
   - **tags:** optional list. The **first** tag is shown on the homepage and as the label above the post title.

3. Write the body in **Markdown** below the closing `---`.

4. **Commit and push** to `main`. GitHub Pages rebuilds automatically; the post appears on the homepage in the “Recent writing” list (newest first). No need to edit `index.html`.

### Files tied to one post (evidence, queries, images)

Keep everything for a given post under **`assets/<slug>/`**, where **`<slug>`** is the part of the post filename after the date.

Example: post `_posts/2025-03-22-aws-cloud-incident-response-exfilcola.md` → folder **`assets/aws-cloud-incident-response-exfilcola/`** (e.g. `attack-navigator/`, `queries/`, `evidence/`). That keeps the repo root clean and groups files with that post.

In that post’s front matter, set:

```yaml
assets_dir: aws-cloud-incident-response-exfilcola
```

Right after the front matter (first lines of the body), add:

```liquid
{% assign asset_base = '/assets/' | append: page.assets_dir %}
```

Link to files with the `relative_url` filter so `baseurl` stays correct:

```liquid
{{ asset_base | append: '/evidence/screenshot.png' | relative_url }}
```

Posts with no extra files can omit `assets_dir` and this block.

### Excerpts on the homepage

With `show_excerpts: true` in `_config.yml`, Jekyll uses the post excerpt on the index. By default that’s the first paragraph, or you can set it explicitly in front matter:

```yaml
excerpt: "One line summary for the listing."
```

## Editing static pages

- **Homepage intro + post list:** `index.html` (uses `layout: default`).
- **About page:** `about.md` (uses `layout: default`).
- **Shared chrome (header, footer, CSS):** `_layouts/default.html`.
- **Post chrome (title block, content wrapper):** `_layouts/post.html`.
- **Site title, URL, base path:** `_config.yml` (`url`, `baseurl`, etc.).

## Optional: preview locally

If you have Ruby and Bundler, you can add a `Gemfile` with the `github-pages` or `jekyll` gem, run `bundle install`, then:

```bash
bundle exec jekyll serve
```

Open the URL it prints (often `http://127.0.0.1:4000/locus/` when using this repo’s `baseurl`).
