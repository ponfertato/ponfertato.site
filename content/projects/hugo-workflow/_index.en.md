---
title: "Hugo: Static Site Architecture & CI/CD"
description: "Practical Hugo workflow: multilingual support, cascade params, Hugo Pipes, custom shortcodes, Docker builds."
tags: ["hugo", "static", "docker", "ci-cd", "blowfish", "papermod"]
---

Using Hugo for static site generation delivers maximum performance with minimal overhead. All projects in the `*.potatoenergy.ru` stack share a unified methodology for configuration, building, and deployment.

**Core principles**:

- 🔹 Declarative configuration via `hugo.toml` and `config/_default/`
- 🔹 Multilingual support via `translationKey` without logic duplication
- 🔹 Parameter inheritance via `cascade` in front matter
- 🔹 Asset optimization via Hugo Pipes (minification, fallbacks, hashing)
- 🔹 Build environment isolation via Docker

---

## 🗂 Project structure

```
project/
├── config/_default/
│   ├── hugo.toml          # Base settings: baseURL, theme, outputs
│   ├── params.toml        # Global theme params and custom variables
│   ├── menus.toml         # Navigation configuration
│   └── languages.*.toml   # Localization: displayName, copyright, author
├── content/
│   ├── {section}/
│   │   ├── _index.{ru,en}.md  # Listing page with section description
│   │   └── {slug}/
│   │       ├── index.{ru,en}.md  # Content with translation via translationKey
│   │       └── assets/           # Page-bundled resources
│   └── _index.{ru,en}.md    # Homepage
├── layouts/
│   ├── _default/            # Default templates: single.html, list.html
│   ├── partials/            # Reusable components: head, footer, svg
│   ├── shortcodes/          # Custom shortcodes: audio, toggle, icon
│   └── shortcodes/          # Theme template overrides (if needed)
├── static/                  # Files copied as-is: robots.txt, favicons
├── assets/                  # Hugo Pipes sources: CSS/JS for processing
└── docker-compose.yml       # Local development environment
```

---

## 🌐 Multilingual without duplication

Translation linking is handled via `translationKey` in front matter:

```yaml
# content/blog/post/_index.ru.md
title: "NixOS Configuration"
translationKey: "nixos-setup"
lang: ru

# content/blog/post/_index.en.md
title: "NixOS Configuration"
translationKey: "nixos-setup"
lang: en
```

**Benefits**:

- ✅ Unified `slug` for both languages - clean URLs without `/ru/`, `/en/` in path
- ✅ Automatic `hreflang` generation for search engines
- ✅ Theme language switcher works out of the box
- ✅ Fallback to default language if translation is missing

**Configuration in `hugo.toml`**:

```toml
defaultContentLanguage = "en"
[languages]
  [languages.ru]
    disabled = false
    languageName = "Русский"
    weight = 1
  [languages.en]
    disabled = false
    languageName = "English"
    weight = 2
```

---

## 📦 Cascade: parameter inheritance

Parameters set in a section's `_index.md` automatically apply to all nested pages via `cascade`:

```yaml
# content/projects/_index.ru.md
---
title: "Projects"
cascade:
  showDate: false
  showReadingTime: false
  layout: "single"
  tags: ["projects"]
---
```

**Result**:

- All articles in `/projects/*` inherit `showDate: false`, `layout: "single"`
- No need to duplicate params in every file
- Easy to change behavior of an entire section by editing one file

**Parameter priority**:

```
Page > Section _index > config/_default/params.toml > theme defaults
```

---

## 🎨 Custom shortcodes

Shortcodes allow embedding complex content without modifying templates.

### Example: `layouts/shortcodes/audio.html`

```go
{{ $src := .Get "src" }}
{{ $caption := .Get "caption" }}
<figure class="audio">
  <audio controls preload="metadata">
    <source src="{{ $src }}" type="audio/mpeg">
  </audio>
  {{ with $caption }}<figcaption>{{ . | markdownify }}</figcaption>{{ end }}
</figure>
```

### Example: `layouts/shortcodes/toggle.html`

```go
{{ $title := .Get "title" }}
{{ $content := .Inner }}
<details>
  <summary>{{ $title }}</summary>
  {{ $content | markdownify }}
</details>
```

**Benefits**:

- ✅ Encapsulation of HTML/CSS/JS inside shortcode
- ✅ Reusability anywhere in content
- ✅ Security: content rendered via `markdownify`, scripts escaped

---

## ⚡ Asset optimization via Hugo Pipes

Hugo Pipes processes CSS/JS/Images at build time:

```go
{{ $css := resources.Get "css/main.css" }}
{{ $css = $css | resources.ToCSS | resources.Minify | resources.Fingerprint }}
<link rel="stylesheet" href="{{ $css.RelPermalink }}" integrity="{{ $css.Data.Integrity }}">
```

**Processing pipeline**:

1. `resources.Get` - load source file from `assets/`
2. `ToCSS` - compile SCSS/Sass (if used)
3. `Minify` - remove whitespace, comments, shorten names
4. `Fingerprint` - add hash to filename for cache-busting
5. `RelPermalink` - generate relative path for HTML

**Result**:

- Final file: `/css/main.min.abc123.css`
- `integrity` attribute for Subresource Integrity (SRI)
- Automatic cache invalidation on content change

---

## 🐳 Docker builds: isolation and reproducibility

### `docker-compose.yml` for local development

```yaml
services:
  hugo:
    command: server --appendPort=false --baseURL=http://localhost:1313 --buildDrafts --disableFastRender
    container_name: hugo
    image: ghcr.io/hugomods/hugo:base
    ports:
      - "1313:1313"
    restart: unless-stopped
    user: "1000:1002"
    volumes:
      - .:/src
```

**Parameter breakdown**:

| Parameter             | Description                                                    |
| --------------------- | -------------------------------------------------------------- |
| `--appendPort=false`  | Don't append `:1313` to `baseURL` - clean preview links        |
| `--buildDrafts`       | Include drafts (`draft: true`) in local builds                 |
| `--disableFastRender` | Full rebuild on changes - more reliable for template debugging |
| `user: '1000:1002'`   | Run as regular user - files created with correct permissions   |
| `volumes: - .:/src`   | Mount project into container - changes visible instantly       |

**Why `ghcr.io/hugomods/hugo:base`**:

- ✅ Official image with Hugo Extended support (required for Sass, WebP)
- ✅ Minimal size (~150 MB) - fast pull
- ✅ Stable version tags - reproducible builds

### Deployment: one command

```bash
# Local dev
docker compose up -d

# Production build
docker compose run --rm hugo --minify --gc

# Copy artifacts
rsync -avz public/ user@server:/var/www/site/
```

---

## 🔧 CI/CD: automation via GitHub Actions

### `.github/workflows/deploy.yml`

```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive # For themes as git-submodule
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.125.0"
          extended: true
      - run: hugo --minify --gc
      - uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: deploy
          key: ${{ secrets.SSH_KEY }}
          source: "public/*"
          target: "/var/www/site"
          rm: true
```

**Highlights**:

- `submodules: recursive` - auto-init themes as git submodules
- `--gc` - garbage collect unused resources (cache cleanup)
- `scp-action` - atomic deploy via SSH with old file removal

---

## 🛠 Debugging and diagnostics

```bash
# Check configuration
hugo config --format yaml | grep -A5 languages

# Preview generated paths
hugo --printPathWarnings

# Find broken links
hugo --minify 2>&1 | grep -i "error\|warning"

# Verify asset integrity
grep -r "integrity=" public/ | wc -l

# Local preview with drafts
docker compose run --rm hugo server --buildDrafts --bind 0.0.0.0
```

---

## ⚠️ Common issues

```bash
# Template not applied
→ Check filename: layouts/{section}/single.html, not layouts/{section}.html
→ Ensure front matter has no `layout: "other"`

# Translations not switching
→ Verify `translationKey`: must match in both versions
→ Ensure `defaultContentLanguage` is set in `hugo.toml`

# Assets not minified
→ Verify Hugo Extended is used: `hugo version | grep extended`
→ Ensure template includes `resources.Minify`

# Docker: files created as root
→ Add `user: '1000:1002'` to docker-compose.yml
→ Or run: `docker compose run --user $(id -u):$(id -g) ...`

# Browser cache not invalidating
→ Ensure template uses `resources.Fingerprint`
→ Check server headers: `Cache-Control: no-cache` for `*.css`, `*.js`
```

---

## Links

- 📘 [Hugo Documentation](https://gohugo.io/documentation/)
- 🎨 [Blowfish Theme Docs](https://blowfish.page/docs/)
- 🧩 [PaperMod Theme](https://github.com/adityatelange/hugo-PaperMod)
- 🐳 [Hugo Docker Images](https://github.com/hugomods/docker-hugo)
- 🔧 [Hugo Pipes Guide](https://gohugo.io/hugo-pipes/)
