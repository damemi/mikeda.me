# mikeda.me

Personal blog built with [Hugo](https://gohugo.io/) and deployed to GitHub Pages.

## Prerequisites

- [Git](https://git-scm.com/)
- Hugo **Extended** v0.140.2 or newer (matches the version used in CI)

## Install Hugo

### macOS (Homebrew)

```bash
brew install hugo
```

### Linux / manual install

Download the extended binary from the [Hugo releases page](https://github.com/gohugoio/hugo/releases). The site CI uses v0.140.2:

```bash
HUGO_VERSION=0.140.2
curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
mkdir -p ~/.local/hugo
tar -C ~/.local/hugo -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
export PATH="$HOME/.local/hugo:$PATH"
```

Verify the install:

```bash
hugo version
```

## Local development

Clone the repo and start the development server:

```bash
git clone https://github.com/damemi/mikeda.me.git
cd mikeda.me
hugo server
```

Open [http://localhost:1313](http://localhost:1313). The server reloads automatically when you edit content.

To build the static site locally:

```bash
hugo --gc --minify
```

Output is written to `public/` (gitignored).

## Creating a new post

Use Hugo's `new` command to scaffold a post from the `archetypes/posts.md` template:

```bash
hugo new posts/my-post-title.md
```

This creates `content/posts/my-post-title.md` with front matter like:

```yaml
---
title: "My Post Title"
date: 2026-07-02T12:00:00-04:00
slug: "my-post-title"
description: ""
draft: true
---
```

Edit the file and write your content below the front matter in Markdown.

### Front matter fields

| Field | Description |
|-------|-------------|
| `title` | Post title shown on the site |
| `date` | Publication date |
| `slug` | URL path (posts are served at `/<slug>/`) |
| `description` | Short summary for SEO and listings |
| `draft` | Set to `false` when ready to publish |

Optional fields used by some existing posts:

```yaml
categories:
  - Uncategorized
tags:
  - kubernetes
  - golang
```

### Preview a draft

Drafts are hidden by default. To preview them locally:

```bash
hugo server -D
```

### Publish

Set `draft: false` in the post's front matter, then commit and push to `main`. GitHub Actions builds and deploys the site automatically.

## Project layout

```
content/
  _index.md          # About page (home)
  about.md
  posts/             # Blog posts
archetypes/
  posts.md           # Template for new posts
themes/minimal/      # Site theme
hugo.yaml            # Hugo configuration
```

## Deployment

Pushes to `main` trigger the [GitHub Actions workflow](.github/workflows/hugo.yaml), which builds with Hugo and deploys to GitHub Pages. No manual deploy step is needed.
