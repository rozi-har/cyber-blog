# Rozi · Cyber & AI

Personal cybersecurity blog: CTF write-ups, AI security notes and lab projects.

**Live site:** https://cyber-blog.rozi-har.workers.dev

## Stack

- [Hugo](https://gohugo.io/) static site generator, bilingual (English default, French under `/fr/`)
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, included as a Git submodule
- Hosted on Cloudflare (Workers static assets), deployed automatically on every push to `main`
- Security headers (CSP, HSTS, etc.) defined in `static/_headers`

## Run it locally

```
git clone --recurse-submodules git@github.com:rozi-har/cyber-blog.git
cd cyber-blog
hugo server -D
```

The site is then available at http://localhost:1313/ (French at `/fr/`).
Requires the extended version of Hugo.

## Project structure

```
content/      pages and posts (*.en.md and *.fr.md)
archetypes/   templates for new content (e.g. CTF write-ups)
static/       static files, including the _headers file
themes/       theme (Git submodule)
hugo.toml     site configuration
wrangler.jsonc Cloudflare deployment configuration
```

## New write-up

```
hugo new content --kind writeup posts/my-writeup.en.md
```

Posts stay as `draft: true` until reviewed.

## About the content

All content is my own work and opinion. It never includes confidential information about employers, clients or private systems. Techniques are tested only on my own lab or on platforms that allow it (CTFs, training labs).

## License

Blog content © 2026 Rozi. The PaperMod theme keeps its own license (see its repository).
