# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Jekyll-based documentation site configured for GitHub Pages using the "Minimal Mistakes" remote theme. The site features a modern, responsive design optimized for documentation with built-in search, multiple layout options, and extensive customization capabilities.

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

- `_config.yml` - Main Jekyll configuration with Minimal Mistakes theme settings
- `_data/navigation.yml` - Navigation menu configuration
- `_site/` - Generated static site (excluded from git)
- `index.md` - Homepage with splash layout
- `installation.md` - Installation guide with single layout
- `quickstart.md` - Quick start tutorial with TOC
- `api.md` - API reference documentation with wide layout
- `faq.md` - Frequently asked questions with sticky TOC
- `contributing.md` - Contributing guidelines
- `404.html` - Custom 404 error page

## Content Management

### Creating Documentation Pages
All documentation pages should be in Markdown (.md) format with appropriate front matter:

```yaml
---
layout: single  # or splash, archive, etc.
title: "Page Title"
permalink: /section/
toc: true
toc_label: "Contents"
toc_icon: "cog"
toc_sticky: true
---
```

### Available Layouts
- `single` - Standard documentation page with sidebar
- `splash` - Landing page with feature rows
- `archive` - Collection/list pages
- `wide` - Full-width content (use with `classes: wide`)

### Navigation
Configure navigation in `_data/navigation.yml`:
- `main` - Top navigation bar
- `docs` - Sidebar navigation (set in page defaults)

### Markdown Features
- Notices: `{: .notice}`, `{: .notice--info}`, `{: .notice--warning}`
- Buttons: `{: .btn .btn--primary}`, `{: .btn--success}`
- Image alignment: `{: .align-left}`, `{: .align-right}`
- Table of contents: Set `toc: true` in front matter

## GitHub Pages Configuration

The site is configured for GitHub Pages with:
- `github-pages` gem version ~> 232
- `remote_theme: "mmistakes/minimal-mistakes@4.24.0"` - Minimal Mistakes theme
- Multiple Jekyll plugins (feed, sitemap, SEO tags, paginate)
- Lunr.js search functionality
- Responsive design with multiple breakpoints
- Font Awesome icons support

## Important Notes

- The `_site/` directory contains generated files and should not be edited directly
- Changes to `_config.yml` require restarting the Jekyll server
- The site uses the GitHub Pages gem which locks Jekyll to a specific version compatible with GitHub's infrastructure