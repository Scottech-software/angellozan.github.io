# Instructions and Guidelines for AI Agents and Contributors

Welcome to Scott Lozano's Portfolio & Personal Blog repository. This document serves as a guide for automated AI assistants (agents) and human contributors who work on maintaining, developing, or writing content for this codebase.

---

## 1. Environment Setup & Dependency Management

To prevent global write permission errors and ensure reproducible builds across sandboxes and local environments, always configure Bundler to use a local vendor path.

### Installation Instructions
Run the following command before running any Jekyll build or serving commands:
```bash
bundle config set --local path 'vendor/bundle' && bundle install
```

---

## 2. Technical Architecture & Plugins

This site is built as a static website using **Jekyll**, themed with **`jekyll-theme-chirpy`**, and configured for multilingual support via **`jekyll-polyglot`**.

### Core Multilingual Details
- **Default Language**: English (`en`)
- **Supported Languages**: English (`en`) and Portuguese (`pt`)
- **Post/Page Directory Structure**:
  - English posts: `_posts/en/`
  - Portuguese posts: `_posts/pt/`
  - Translatable tabs/pages: `_tabs/` (e.g. `about.md` and `about.pt.md`)

### Custom Plugins and Overrides
1. **I18n Headers Patch** (`_plugins/i18n-headers-patch.rb`):
   - Overrides the `I18n_Headers` Liquid tag to prevent duplicate slashes in the generated alternate language link tags (e.g., generating `https://scottech.software/pt/` instead of `https://scottech.software//pt/`).
2. **Canonical & Fallback Canonical Settings**:
   - The site uses `fallback_canonical_to_default_lang: true` in `_config.yml` to specify how localized fallback pages map to their canonical default language pages.
   - To prevent duplicate indexing of fallback pages, `canonical=false` is passed directly to the Jekyll SEO tag in `_includes/head.html`.

---

## 3. Development Workflow & Testing

Always verify all HTML, link paths, and build scripts before submitting changes.

### Clean & Build Site
To build the site locally under the production environment:
```bash
bundle exec jekyll build
```

### Run Local Development Server
To serve the site locally:
```bash
bundle exec jekyll serve
# OR use the provided tool script:
bash tools/run.sh
```

### Run HTMLProofer Checks
Always run HTMLProofer to verify local/internal links and HTML syntax, while ignoring local developer loopback addresses:
```bash
bundle exec htmlproofer _site --disable-external --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
# OR use the provided tool script:
bash tools/test.sh
```

---

## 4. Writing & Translating Posts

When adding or modifying a post:
1. Ensure there is a corresponding file in **both** `_posts/en/` and `_posts/pt/` with the exact same filename (e.g., `_posts/en/YYYY-MM-DD-title.md` and `_posts/pt/YYYY-MM-DD-title.md`).
2. Keep the front matter consistent. Set `lang: en` for English posts and `lang: pt` for Portuguese posts.
3. Keep the `permalink` structure uniform across languages so polyglot can link them correctly.
