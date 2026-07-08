# abacu5.github.io

## First-time setup

1. Create a new repo on GitHub named exactly `abacu5.github.io` (must match your username for the free auto-hosted domain).
2. Unzip these files into that repo's folder.
3. Push:
   ```
   git init
   git add .
   git commit -m "Initial site"
   git branch -M main
   git remote add origin https://github.com/abacu5/abacu5.github.io.git
   git push -u origin main
   ```
4. On GitHub: Settings → Pages → Source → set to `main` branch, `/ (root)`. Site goes live at `https://abacu5.github.io` in a minute or two — no build step needed, GitHub builds Jekyll automatically.

## Adding a new writeup

Create a file in `_posts/` named `YYYY-MM-DD-short-slug.md`, e.g. `2026-07-10-htb-lame.md`:

```markdown
---
title: "HTB: Lame"
platform: "Hack The Box (retired)"
difficulty: "Easy"
os: "Linux"
excerpt: "One-line summary shown on the homepage."
---

## Recon
...

## Foothold
...

## Privilege Escalation
...
```

Delete the sample post in `_posts/` once you've added your real ones.

## Local preview (optional)

```
bundle install
bundle exec jekyll serve
```
Then open http://localhost:4000

## Migrating from Medium

Medium doesn't give a clean markdown export, so the fastest path is:
1. Open each post, copy the body text.
2. Paste into a new `_posts/*.md` file, reformat with markdown headers, fix any code blocks to use triple backticks.
3. Screenshots: save the image files into `assets/img/` and reference them as `![alt text](/assets/img/filename.png)`.
