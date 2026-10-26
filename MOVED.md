# This blog moved to shissorobotics.com

Since 2026-10-26 the blog is served at https://shissorobotics.com (same URL paths: `/blog/YYYY/MM/DD/<slug>/`,
`/papers/`, `/experiments/`, `/architecture/`; the old home page is `/thesis/`).

- Source now lives in the site repo, under `blog-src/` (MkDocs Material, same theme and plugins).
- This repo keeps the original MkDocs source in `docs/` and `mkdocs.yml`, untouched, for history.
- GitHub Pages now serves `redirects/`: one page per old path with a canonical link and a meta refresh to the same
  path on shissorobotics.com (GitHub Pages cannot send a 301). `/` goes to `/thesis/`.
