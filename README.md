# felipeelias.github.io

Personal blog about software engineering, leadership, and AI. Built with Jekyll 4, deployed to GitHub Pages.

## Setup

```bash
bundle install
```

## Development

```bash
rake preview             # Local server at localhost:4000
rake post title="Title"  # New post
rake draft title="Title" # New draft
rake optimize_images     # Convert images to WebP
```

## Stack

- Jekyll 4 / Ruby 3.3
- SASS for styles, Kramdown (GFM) for Markdown
- Giscus for comments
- GitHub Actions for deployment
