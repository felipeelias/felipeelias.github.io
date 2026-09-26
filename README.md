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

## Markdown linting

Install [pre-commit](https://pre-commit.com/#install), then enable the local hook
and run the same check as CI:

```bash
pre-commit install
pre-commit run markdownlint-cli2 --all-files --show-diff-on-failure
```

The markdownlint version is pinned in `.pre-commit-config.yaml`. Update that
revision to upgrade both local development and CI. Rules, file globs, and
exclusions live in `.markdownlint-cli2.yaml`; both environments check the full
configured set of Markdown files.

## Stack

- Jekyll 4 / Ruby 3.3
- SASS for styles, Kramdown (GFM) for Markdown
- Giscus for comments
- GitHub Actions for deployment
