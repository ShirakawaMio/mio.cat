# Mio's Little Space

Personal blog built with [Hexo](https://hexo.io) and [hexo-theme-meow](https://github.com/chanwj/hexo-theme-meow), deployed to GitHub Pages at [mio.cat](https://mio.cat).

---

## Local Development

```bash
npm install
hexo server        # Start dev server at http://localhost:4000
```

---

## Writing Content

### New Post
```bash
hexo new "My Post Title"
# → source/_posts/My-Post-Title.md
```

### New Page (e.g. About)
```bash
hexo new page about
# → source/about/index.md
```

### New Draft (hidden until published)
```bash
hexo new draft "Draft Title"
hexo publish "Draft Title"   # move to _posts when ready
```

### Post Front Matter
```yaml
---
title: My Post Title
date: 2026-01-01 12:00:00
updated: 2026-01-02 12:00:00
tags: [tag1, tag2]
categories: [Category, Subcategory]
cover: /images/cover.jpg   # optional post cover image
excerpt: A short summary.  # optional manual excerpt
---
```

---

## Meow Theme Features

### Navigation Menu
Edit `themes/meow/_config.yml` → `menu`:
```yaml
menu:
  articles || uc-article:
    archives: /archives || uc-archive
    categories: /categories || uc-category
    tags: /tags || uc-tag
  about: /about || uc-about
```
Format: `label: /path || icon-name`

### Banner & Subtitle
```yaml
banner:
  image:
    light: assets/images/your-light-bg.jpg
    dark: assets/images/your-dark-bg.jpg
  title:
    text: "「 Your Title 」"
  subtitle:
    enable: true
    text:
      - "Hello World"
      - "Welcome to my blog"
    effect: true   # typewriter effect
```

### Sidebar
```yaml
sidebar:
  info:
    avatar:        # path to your avatar image
    status:
      emoji: 🐱
      desc: "Online"
    desc: "Something about me"
    social:
      enable: true
      list:
        github: brand-github || https://github.com/ShirakawaMio
        email: brand-mail || your@email.com
```

### Comments (Waline)
```yaml
comment:
  enable: true
  type: waline
  waline:
    server_url: https://your-waline-server.vercel.app
    lang: en-US
```

### Music Player
```yaml
music:
  enable: true
  autoplay: false
  meting:
    server: netease      # netease / tencent / kugou
    type: playlist
    id: 123456789        # playlist ID
```

### Search
```bash
npm install hexo-generator-searchdb --save
```
```yaml
# _config.yml
search:
  path: search.xml
  field: post

# themes/meow/_config.yml
search:
  enable: true
  type: local
```

### Light/Dark Mode
```yaml
color:
  light:
    theme_color: '#FFB347'
  dark:
    theme_color: '#88C0ED'
  mode:
    default: light    # or dark
```

---

## Asset Management

Place images in `source/images/` and reference as `/images/filename.jpg`.

```bash
# Enable per-post asset folders
# _config.yml
post_asset_folder: true
# Then place assets next to the post file and reference as ./image.jpg
```

---

## Deployment

Pushing to `main` triggers GitHub Actions which builds and deploys to the `gh-pages` branch automatically.

```bash
git add .
git commit -m "New post: My Title"
git push
```

Manual build (local only):
```bash
hexo clean && hexo generate
```

---

## Useful Commands

| Command | Description |
|---|---|
| `hexo new "Title"` | Create a new post |
| `hexo new page name` | Create a new page |
| `hexo server` | Start local server |
| `hexo clean` | Clear generated files |
| `hexo generate` | Build static files |
| `hexo list post` | List all posts |

---

## References

- [Hexo Docs](https://hexo.io/docs/)
- [Meow Theme Docs](https://meow.jumaoo.top/categories/Docs文档/)
- [Meow GitHub](https://github.com/chanwj/hexo-theme-meow)
