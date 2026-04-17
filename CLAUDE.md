# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal blog built with **Hugo** static site generator, using the PaperMod theme. Deployed to GitHub Pages via GitHub Actions.

## Commands

All commands use native Hugo:

- **Local development server**: `hugo server` (with live reload)
- **Build production**: `hugo --minify` (with minification)
- **Build with garbage collection**: `hugo --minify --gc`
- **Clean build output**: `rm -rf public/`

## Architecture

- **content/** - Blog posts and pages in Markdown
  - `content/posts/` - Blog posts
  - `content/about.md`, `archives.md`, `search.md` - Static pages
- **themes/PaperMod** - PaperMod theme (git submodule)
- **public/** - Generated static site (build output)
- **hugo.toml** - Main Hugo configuration
- **.github/workflows/hugo.yml** - GitHub Actions deployment to GitHub Pages

## Configuration

- Site is in Chinese (defaultContentLanguage = zh-cn)
- Theme: PaperMod with auto light/dark mode
- Features enabled: reading time, share buttons, post nav links, search (Fuse.js), table of contents
- Output: HTML, RSS, JSON (for search)

## Deployment

- Pushes to `main` branch trigger GitHub Actions build
- Automatically deploys to `gh-pages` branch which serves GitHub Pages
- Base URL: https://yinbit.github.io/
