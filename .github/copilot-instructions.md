# Copilot Instructions for pjsoni.github.io

## Project Overview
This is a **Jekyll-based static blog/portfolio site** hosted on GitHub Pages, built with the Minimal Mistakes theme. It showcases a personal tech portfolio with blog posts spanning from 2009 onwards and professional pages (about, resume, blog). The site is configured for custom domain hosting at `pravesh.me`.

## Key Architecture

### Core Structure
- **Build System**: Jekyll 4.3.3 with Ruby/Bundler - run `bundle exec jekyll serve` to build/preview
- **Theme**: Minimal Mistakes (local customization layer over remote theme)
- **Content Types**:
  - **Posts** (`_posts/`): Blog articles with YAML frontmatter including `layout: single`, tags, and categories
  - **Pages** (`_pages/`): Static pages (about.md, blog.md, resume.md, 404.md) with `permalink` in frontmatter
  - **Navigation**: Defined in `_data/navigation.yml` - add new main navigation items in the `main:` section
  - **Drafts** (`_drafts/`): Unpublished posts (not built by default)

### Content Configuration
- `_config.yml`: Site-wide settings (title, URL, locale, comments provider, etc.)
- `_data/navigation.yml`: Main navigation menu
- `_data/ui-text.yml`: UI labels and strings for translations

### Template & Styling Layers
- **Layouts** (`_layouts/`): Post/page templates (single.html, archive.html, homepage.html, etc.)
- **Includes** (`_includes/`): Reusable components (masthead, footer, sidebar, social-share, etc.)
- **Styling** (`_sass/minimal-mistakes/`): Scoped SCSS modules (animations, archive, base, navigation, sidebar, etc.) organized by component

## Development Workflow

### Local Setup & Running
```bash
bundle install           # Install gem dependencies
bundle exec jekyll serve # Build and run locally on http://localhost:4000
```
Changes to markdown files auto-refresh; changes to `_config.yml` require server restart.

### Publishing Content
1. **Blog Posts**: Add `.md` file to `_posts/` with filename format `YYYY-MM-DD-title.md`
2. **Pages**: Add `.md` to `_pages/` with `permalink` in frontmatter
3. **Frontmatter Pattern**:
   ```yaml
   ---
   layout: single
   title: "Your Title"
   tags: [tag1, tag2]
   category: [blog]
   ---
   ```

## Project-Specific Conventions

### Content Organization
- **Taxonomy**: Posts use `tags:` (multiple) and `category:` (typically single) for classification
- **Navigation**: Edit `_data/navigation.yml` to update main menu links - don't hardcode routes
- **Layout Choices**: Single blog posts use `layout: single`, multi-item pages use `layout: archive` or `layout: posts`

### Styling & Theme Customization
- Theme skin set to `"air"` in `_config.yml` (options: air, aqua, contrast, dark, dirt, neon, mint, plum, sunrise)
- Custom SCSS placed in `_sass/minimal-mistakes/` - files are prefixed with `_` (e.g., `_archive.scss`)
- Component variables defined in `_variables.scss`; mixins in `_mixins.scss`

### Image Assets
- Images stored in `assets/images/`
- Relative paths in markdown: `/assets/images/filename.ext`
- Inline image styling supported: `{: .align-left}`, `{:width="180px"}`

## Common Tasks

| Task | Location/Pattern |
|------|------------------|
| Add blog post | Create `_posts/YYYY-MM-DD-title.md` with single layout |
| Update navigation menu | Edit `_data/navigation.yml` |
| Change site title/URL | Edit `_config.yml` |
| Modify page template | Edit `_layouts/` `.html` file |
| Add reusable component | Create `_includes/component-name.html` and use `{% include component-name.html %}` |
| Customize styling | Add/edit `_sass/minimal-mistakes/_component.scss` |
| Manage permalinks | Set `permalink: /custom-url.html` in page frontmatter |

## GitHub Pages Deployment
- Site is pushed to GitHub Pages (custom domain: pravesh.me via CNAME file)
- Uses Jekyll build process automatically
- Ensure `Gemfile` dependencies are compatible with GitHub Pages environment

## Notable Patterns
- **Image alignment**: Use Markdown image syntax with Jekyll classes: `![alt](/path/to/image.jpg){: .align-left}`
- **Collection display**: `_layouts/archive.html` groups posts by taxonomy (tag/category)
- **Comments**: Disabled for this site (`provider: false`); could enable via disqus/discourse by updating config
