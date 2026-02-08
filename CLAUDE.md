# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Zenn content repository for publishing technical articles and books on [zenn.dev](https://zenn.dev).

## Commands

```bash
# Start local preview server
npx zenn preview

# Create new article
npx zenn new:article

# Create new article with specific slug
npx zenn new:article --slug my-article-slug

# Create new book
npx zenn new:book
```

## Directory Structure

- `articles/` - Markdown files for articles (one file per article)
- `books/` - Book content (one directory per book with chapters)

## Article Format

Articles are Markdown files with YAML frontmatter:

```yaml
---
title: "記事タイトル"
emoji: "📘"
type: "tech"  # tech: 技術記事 / idea: アイデア
topics: ["javascript", "react"]
published: false
---
```

## Slug Naming Convention

Use descriptive, hyphenated slugs that reflect the article content:
- Good: `javascript-safe-coding-tips`, `react-hooks-best-practices`
- Avoid: random hashes like `730102fb2e2ade`

Slugs only need to be unique within this repository (not across all Zenn users).
