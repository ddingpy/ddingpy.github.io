# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Jekyll-based documentation site configured for GitHub Pages. The site uses the Minima theme and is set up to work with GitHub Pages gem version 232.

## Key Commands

### Local Development
```bash
# Install dependencies
bundle install

# Serve the site locally with auto-reload
bundle exec jekyll serve

# Build the site
bundle exec jekyll build

# Serve with drafts visible
bundle exec jekyll serve --drafts
```

### Dependency Management
```bash
# Update all gems
bundle update

# Update specific gem
bundle update jekyll
```

## Project Structure

- `_config.yml` - Main Jekyll configuration
- `_posts/` - Blog posts in YYYY-MM-DD-title.markdown format
- `_site/` - Generated static site (excluded from git)
- `index.markdown` - Homepage with `layout: home`
- `about.markdown` - About page example

## Content Management

### Creating Posts
Posts must be placed in `_posts/` with filename format: `YYYY-MM-DD-title.markdown`

Required front matter:
```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS +TIMEZONE
categories: category1 category2
---
```

### Creating Pages
Pages can be created in the root directory with front matter:
```yaml
---
layout: page
title: "Page Title"
permalink: /custom-url/
---
```

## GitHub Pages Configuration

The site is configured for GitHub Pages with:
- `github-pages` gem version ~> 232
- Minima theme version ~> 2.5
- Jekyll Feed plugin for RSS generation

## Important Notes

- The `_site/` directory contains generated files and should not be edited directly
- Changes to `_config.yml` require restarting the Jekyll server
- The site uses the GitHub Pages gem which locks Jekyll to a specific version compatible with GitHub's infrastructure