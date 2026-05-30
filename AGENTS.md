# bbs — Hugo blog site

**Hugo static site** — personal blog "小蜜蜂" at [xumf.net](https://xumf.net), deployed to GitHub Pages.

Theme: [PaperMod](https://github.com/adityatelange/hugo-PaperMod).

## Quick start

```bash
# Theme is not vendored — clone it first
git clone --depth 1 https://github.com/adityatelange/hugo-PaperMod themes/papermod

# Live preview (default http://localhost:1313)
hugo serve

# Production build
hugo
hugo --baseURL "https://xumf.github.io/"   # match CI
```

## Structure

| Path | Purpose |
|------|---------|
| `content/blog/` | Blog posts (markdown) |
| `content/work/` | Portfolio entries |
| `content/about/` | Static pages |
| `archetypes/` | `hugo new` templates: `blog.md`, `work.md`, `default.md` |
| `static/images/` | Site images |
| `public/` | Build output (gitignored) |
| `themes/` | Theme (gitignored, clone required) |

## Content conventions

- Default language is `zh`.
- New blog post: `hugo new blog/my-title.md` (uses `archetypes/blog.md`).
- Front matter includes `tags`, `draft`, `description`.
- Site is in Chinese; future agents should write content in Chinese unless told otherwise.
- PaperMod front-matter vars: `showToc`, `cover.image`, `cover.hidden`, `searchHidden`.

## CI / Deploy

Two GitHub Actions workflows in `.github/workflows/`:
- `hugo.yml` — deploys to GitHub Pages via `actions/deploy-pages`. Runs Hugo v0.147.4 extended.
- `gh-pages.yml` — older; pushes `public/` to `gh-pages` branch.

Deploy triggers: push to `master` branch only.

## Notes

- `dev` branch exists on origin but default branch is `master`.
- No test/lint/typecheck tooling — just `hugo build`.
- No Node.js or build step beyond Hugo.
- Theme is set in `config.toml` (`theme = "papermod"`), no `--theme` flag needed.
