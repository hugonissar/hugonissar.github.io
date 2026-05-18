# hugonissar.github.io

Personal site and blog for [@hugonissar](https://github.com/hugonissar) — built
with Jekyll, deployed to GitHub Pages via GitHub Actions, with heavy SEO and
AI-citation optimization baked in.

- **Live:** https://hugonissar.github.io (once deployed)
- **Build:** [Jekyll 4.x](https://jekyllrb.com/) + [GitHub Actions](https://github.com/features/actions)
- **Hosting:** [GitHub Pages](https://docs.github.com/en/pages) (free, on github.io)

---

## Quick start — 10 minutes to live

### 1. Create the repo

GitHub Pages user sites have to be named `<username>.github.io`. Create a new
**public** repo on GitHub called **`hugonissar.github.io`** (exactly that —
no other name will publish to the root user URL).

Don't add a README, .gitignore, or license when creating it. You'll push these
in from local.

### 2. Clone this template into your local repo

From the directory where this README lives:

```bash
git init
git add .
git commit -m "Initial commit — Jekyll site skeleton"
git branch -M main
git remote add origin git@github.com:hugonissar/hugonissar.github.io.git
git push -u origin main
```

### 3. Enable GitHub Pages → Actions

In the repo on GitHub:

1. Go to **Settings → Pages**.
2. Under **Build and deployment**, set **Source** to **GitHub Actions**.
3. That's it. No branch selection needed — the workflow handles deployment.

### 4. Watch the first build

Go to the **Actions** tab. The `Build and deploy Jekyll site` workflow runs
on push and on the manual *Run workflow* button. The first build takes about
2–3 minutes. When it's green, your site is live at
**https://hugonissar.github.io**.

### 5. Verify with Google Search Console

1. Go to https://search.google.com/search-console.
2. Add a **URL prefix** property: `https://hugonissar.github.io/`.
3. Verify ownership — the easiest method is the *HTML tag* method. Paste the
   provided `<meta name="google-site-verification" content="...">` tag value
   into `_config.yml` under `google_site_verification:` (jekyll-seo-tag picks
   it up automatically), then push.
4. After verification, submit the sitemap at the URL **`/sitemap.xml`**.

That's the whole setup. Everything below is optional polish.

---

## Local development

```bash
# Install Ruby (3.3+) and bundler if you don't have them
# macOS: brew install ruby
# Ubuntu: sudo apt install ruby-full build-essential

bundle install
bundle exec jekyll serve --livereload
# → open http://localhost:4000
```

`--livereload` rebuilds on file changes. Drafts go in `_drafts/` and only show
up if you pass `--drafts`.

---

## Repository layout

```
.
├── .github/workflows/deploy.yml   # CI: builds Jekyll, runs HTMLProofer, deploys
├── _config.yml                    # Site identity, SEO defaults, plugins
├── _data/                         # YAML data files (currently empty — for future use)
├── _includes/                     # Reusable HTML partials
│   ├── header.html
│   ├── footer.html
│   └── json-ld-person.html        # Person schema (Knowledge Graph + LLMs)
├── _layouts/                      # Page templates
│   ├── default.html               # Wraps every page (head, header, footer)
│   ├── post.html                  # Blog post layout w/ BlogPosting schema
│   └── project.html               # Project page layout w/ SoftwareSourceCode schema
├── _posts/                        # Blog posts (Markdown, dated filenames)
├── _projects/                     # Project pages (one .md per repo)
├── assets/
│   ├── css/main.scss              # Single stylesheet, dark-mode aware
│   └── images/                    # favicon.svg, og-default.svg, ...
├── index.html                     # Home page (projects + recent posts)
├── blog.html                      # Blog index
├── about.md                       # About page (EEAT signal)
├── 404.html                       # Custom 404
├── robots.txt                     # Allows all AI crawlers explicitly
├── llms.txt                       # AI-agent context per llmstxt.org
├── Gemfile                        # Ruby dependencies
├── README.md                      # This file
├── SETUP.md                       # → see this file (you're reading it)
└── STRATEGY.md                    # SEO + content strategy playbook
```

---

## How to publish a new blog post

1. Create a new file in `_posts/` named `YYYY-MM-DD-slug.md`.
2. Add front matter — the more you fill in, the better the SEO:

   ```yaml
   ---
   title: "How to do X with BigQuery ML"
   date: 2026-06-01
   last_modified_at: 2026-06-01
   repo: GA4-Ecommerce-BQML-Purchase-Propensity   # optional — surfaces the GitHub CTA
   tags: [bqml, ga4, bigquery]
   description: >-
     One- or two-sentence meta description. ~155 characters. This is what
     shows up in Google snippets and OG cards. Make it count.
   excerpt: >-
     Three- or four-sentence preview shown on the blog index. The first
     paragraph of the body is used if you omit this.
   ---
   ```

3. Write the post in Markdown.
4. `git push`. The Actions workflow builds and deploys.

The post automatically:
- Gets the `BlogPosting` schema via `jekyll-seo-tag`
- Adds to `sitemap.xml`
- Adds to `feed.xml` (RSS)
- Links to the GitHub repo at the bottom if you set the `repo:` front-matter field
- Generates a clean `/blog/the-slug/` URL (no date in URL, better for sharing)

---

## How to add a new project

1. Create a file in `_projects/` named after the repo, e.g. `my-new-project.md`.
2. Front matter:

   ```yaml
   ---
   title: "My New Project"
   tagline: "One-line description that becomes the meta description."
   repo: my-new-project              # the GitHub repo name (under your user)
   language: Python
   license: MIT
   status: v0.1.0
   order: 3                          # display order on the home page
   description: >-
     Full meta description for the project page itself.
   ---
   ```

3. Write a Markdown overview. Keep it tight — the goal is to entice clicks
   through to GitHub, not duplicate the README.
4. Push. The project shows up at `/projects/my-new-project/` and on the home
   page.

---

## Custom domain (optional)

To use your own domain (e.g. `hugonissar.com`):

1. Create a file called `CNAME` at the repo root containing one line:
   `hugonissar.com` (or whichever).
2. In your DNS, add either:
   - **A records** to `185.199.108.153`, `.109.153`, `.110.153`, `.111.153`
     (GitHub Pages' IPs), or
   - A **CNAME** record for `www.hugonissar.com` → `hugonissar.github.io`.
3. In **Settings → Pages**, set the custom domain and tick **Enforce HTTPS**
   (it takes ~15 minutes after DNS propagates).
4. Update `url:` in `_config.yml` to the new domain.

---

## Files you should still customize

Defaults are sensible, but a few things are worth editing once:

| File | What to change |
|---|---|
| `_config.yml` → `author.twitter` | Add your handle if you have one, sans `@` |
| `_config.yml` → `author.email`   | Add if you want it in the RSS feed |
| `assets/images/favicon.svg`      | Replace the placeholder "H" monogram |
| `assets/images/og-default.png`   | Replace the SVG placeholder with a real 1200×630 PNG for better social-share previews |
| `about.md`                       | Personalize the bio |

---

## What's in the SEO strategy

See `STRATEGY.md` for the content playbook — which post formats to write,
which queries to target, and how to use the projects' READMEs as a keyword
seed list. Short version: comparison posts and definitive guides, all with
crystal-clear extractable structure for AI citation.

---

## License

The code in this repository (templates, workflow, styles) is MIT-licensed.
The written content (blog posts, project descriptions, about page) is
CC-BY-4.0 — attribution required, but you can quote freely.
