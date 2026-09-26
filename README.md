# felipeelias.github.io

[![Deploy](https://github.com/felipeelias/felipeelias.github.io/actions/workflows/jekyll.yml/badge.svg)](https://github.com/felipeelias/felipeelias.github.io/actions/workflows/jekyll.yml)
[![CI](https://github.com/felipeelias/felipeelias.github.io/actions/workflows/ci.yml/badge.svg)](https://github.com/felipeelias/felipeelias.github.io/actions/workflows/ci.yml)
[![Website](https://img.shields.io/website?url=https%3A%2F%2Ffelipeelias.github.io)](https://felipeelias.github.io)

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
