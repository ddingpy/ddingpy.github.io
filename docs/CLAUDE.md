# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Jekyll-based documentation site configured for GitHub Pages using the "Just the Docs" remote theme. The site is optimized for documentation with built-in search, responsive design, and clean navigation.

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

- `_config.yml` - Main Jekyll configuration with Just the Docs theme settings
- `_site/` - Generated static site (excluded from git)
- `index.md` - Homepage documentation
- `installation.md` - Installation guide
- `quickstart.md` - Quick start tutorial
- `api.md` - API reference documentation
- `faq.md` - Frequently asked questions
- `contributing.md` - Contributing guidelines

## Content Management

### Creating Documentation Pages
All documentation pages should be in Markdown (.md) format with appropriate front matter:

```yaml
---
layout: default
title: "Page Title"
nav_order: 1
description: "Page description for SEO"
permalink: /section/
---
```

### Navigation
- `nav_order` - Controls the order in sidebar navigation
- `has_children: true` - Creates a parent section
- `parent: "Parent Title"` - Nests page under a parent

### Markdown Features
- Tables with `{: .table-wrapper}`
- Code blocks with syntax highlighting
- Table of contents with `{:toc}`
- Buttons with `{: .btn .btn-primary}`
- Labels with `{: .label .label-green}`

## GitHub Pages Configuration

The site is configured for GitHub Pages with:
- `github-pages` gem version ~> 232
- `remote_theme: "pmarsceill/just-the-docs@v0.3.3"` - Documentation theme
- `baseurl: "/docs"` - For GitHub Pages subpath
- Jekyll Feed plugin for RSS generation
- Jekyll SEO plugin for meta tags
- Full-text search functionality

## Important Notes

- The `_site/` directory contains generated files and should not be edited directly
- Changes to `_config.yml` require restarting the Jekyll server
- The site uses the GitHub Pages gem which locks Jekyll to a specific version compatible with GitHub's infrastructure