+++
author = "yuhao"
title = "How to Start A Blog in 2024"
date = "2024-12-05"
description = "Comprehensive guide on how to set up and deploy a blog using Hugo and GitHub Actions."
tags = [
    "HUGO",
    "Github Action",
    "Blog",
    "themes",
    "markdown"
]
categories = [
    "Programming/Development",
    "Tutorials",
    "Practices"
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
image = "cover.jpg"
+++

# HUGO+GitHub Action deploy your own Blog

# Prerequisites

- Install Hugo (Extended version) in macos

```jsx
brew install hugo
```

- Git installed
- GitHub account

# Step 1: Create a New Hugo Site

```bash
# Create a new Hugo site
hugo new site myblog
cd myblog

# Initialize git repository
git init
```

# Step 2: Add a Theme

Choose and install a Hugo theme in [https://themes.gohugo.io/](https://themes.gohugo.io/). For example, using stack theme:

To use the latest version, you can clone the repository to `themes/hugo-theme-stack` by running the following command in the root directory of your Hugo site:

```jsx
git clone https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```

If you are already using Git for your site, you can add the theme as a submodule by running the following command in the root directory of your Hugo site:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

# Step 3: Configure Your Site

Create or modify hugo.toml/hugo.yaml with basic settings,for here, i just copy and paste the example hugo.yaml from exampleSite included in the theme. you can also transform between .yaml file and .toml file in following website.
[https://transform.tools/toml-to-yaml/](https://transform.tools/toml-to-yaml).

# Step 4: Create Content

HUGO primarily support Markdown and HTML file for content creation you can also generate a default post in command line.

```bash
# Create your first post
hugo new posts/my-first-post.md
```

# Step 5: Setup GitHub Actions

Create .github/workflows/deploy.yml in your repository:

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "latest"
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

# Step 6: Deploy

- Push your code to GitHub repository
- Enable GitHub Pages in repository settings
- Set the source to GitHub Actions

```bash
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/username/repository.git
git push -u origin main
```

Your blog should now be accessible at [https://yourusername.github.io/](https://yourusername.github.io/)

# Important Notes

- Make sure your repository name follows the format: [username.github.io](http://username.github.io)
- Wait a few minutes after pushing for GitHub Actions to complete the deployment
- Check the Actions tab in your GitHub repository for deployment status
