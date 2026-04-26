# Allen's Dev Journal

Personal blog at <https://allenyl30.github.io>, built with [Hugo](https://gohugo.io) and the [Stack](https://github.com/CaiJimmy/hugo-theme-stack) theme. Hosted on GitHub Pages, deployed automatically via GitHub Actions.

## Stack

| Component | Version |
|---|---|
| Hugo (extended) | 0.160.1 |
| Theme: hugo-theme-stack | v3.34.2 (forked at [ALLENYL30/hugo-theme-stack](https://github.com/ALLENYL30/hugo-theme-stack)) |
| Hosting | GitHub Pages |
| Analytics | Cloudflare Web Analytics (cookieless) |

## Repo layout

```
.
├── content/post/        articles, one folder per post (index.md + cover.jpg)
├── layouts/             site-level overrides for theme partials
├── themes/              hugo-theme-stack as a git submodule
├── hugo.yaml            site configuration
└── .github/workflows/   build & deploy pipeline
```

## Local development

Requires Hugo extended ≥ 0.147.

```bash
git clone --recurse-submodules https://github.com/ALLENYL30/ALLENYL30.github.io.git
cd ALLENYL30.github.io
hugo server -D       # http://localhost:1313
```

If you cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## Writing a new post

```bash
mkdir content/post/my-new-post
# create content/post/my-new-post/index.md with TOML front matter
# drop cover image at content/post/my-new-post/cover.jpg
git add content/post/my-new-post
git commit -m "add my-new-post"
git push                      # GitHub Actions handles the rest
```

Front matter template — copy any post in `content/post/` as a starting point.

## Deployment

Every push to `main` triggers `.github/workflows/hugo.yaml`, which builds the site with Hugo and deploys it to GitHub Pages. Live within ~2 minutes.

## License

Posts are licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) unless stated otherwise.
