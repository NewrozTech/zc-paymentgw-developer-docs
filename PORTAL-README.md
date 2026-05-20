# ZiCharge Merchant API — Developer Portal

Static documentation site built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

---

## Folder structure

```
5-Merchant API Portal/
├── mkdocs.yml          # Site config — nav, theme, extensions
├── docs/               # Source markdown (edit here, never in site/)
│   ├── index.md        # Home / quick-start
│   ├── 01-overview.md
│   ├── 02-authentication.md
│   ├── 03-payment-flow.md
│   ├── 04-api-reference.md
│   ├── 05-ipn-callbacks.md
│   ├── 06-error-codes.md
│   ├── 07-testing-and-go-live.md
│   ├── 08-migration-notes.md
│   └── stylesheets/
│       └── extra.css   # ZiCharge brand overrides
└── site/               # Generated output — deploy this folder (do not edit)
```

---

## Prerequisites

```bash
pip install mkdocs-material
```

---

## Local development

```bash
cd "5-Merchant API Portal"
mkdocs serve
# Open http://127.0.0.1:8000
```

Live-reloads on every save to `docs/`.

---

## Build (production)

```bash
mkdocs build --clean
```

Output goes to `site/`. The `site/` folder is fully self-contained — no server-side logic required.

---

## Deployment options

| Option | Command / steps | Notes |
|--------|-----------------|-------|
| **GitHub Pages** | `mkdocs gh-deploy` | Pushes `site/` to the `gh-pages` branch automatically |
| **AWS S3 + CloudFront** | `aws s3 sync site/ s3://<bucket> --delete` | Set `index.html` as the default root object in CloudFront |
| **Vercel / Netlify** | Point root to this folder; build command `mkdocs build`; publish dir `site` | Zero-config, free tier available |
| **Nginx / Apache** | Copy `site/` to your web root | Static files only, no server config needed |

---

## Updating docs

1. Edit the relevant `.md` file inside `docs/`.
2. Run `mkdocs serve` to preview locally.
3. Run `mkdocs build --clean` to regenerate `site/`.
4. Deploy `site/` to your chosen host.

Adding a new page:
1. Create `docs/XX-new-page.md`.
2. Add it to the `nav:` section in `mkdocs.yml`.
3. Rebuild.

---

## Brand / styling

Colors and typography overrides are in `docs/stylesheets/extra.css`.
Primary: `#1B6CA8` (ZiCharge blue) — Accent: `#00B386` (ZiCharge teal).
Both light and dark mode palettes are configured.
