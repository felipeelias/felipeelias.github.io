# felipeelias.github.io

Personal blog built with Jekyll 4 (Ruby 3.3), deployed via GitHub Actions to GitHub Pages.

## Commands

```bash
bundle install           # Install dependencies
rake preview             # Local server at localhost:4000
rake post title="Title"  # New post in _posts/
rake draft title="Title" # New draft in _drafts/
```

## Architecture

- **Config:** `_config.yml` — settings, navigation, plugins, defaults
- **Layouts:** `default.html` → `page.html` / `post.html`
- **Styles:** SASS in `_sass/`, compiled via `assets/css/main.scss`
- **Plugins:** jekyll-mentions, jekyll-seo-tag, jekyll-sitemap
- **Comments:** Giscus (per-post opt-in via `comments: true` in front matter)
- **Deployment:** Push to `main` → `.github/workflows/jekyll.yml`
