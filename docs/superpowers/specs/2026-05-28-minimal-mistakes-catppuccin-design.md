# Minimal Mistakes migration design for mrpurple666.github.io

## Goal

Replace the current hand-built single-page HTML site in `mrpurple666.github.io` with a standard Jekyll site that uses the Minimal Mistakes remote theme and the `catppuccin_mocha` skin.

## Current state

The current site is a static landing page rooted at `index.html` with custom assets under `assets/` and images under `images/`. Content is embedded directly in the HTML in sections such as intro, work, and about.

## Selected approach

Use a clean Jekyll replacement in the existing `mrpurple666.github.io` repository.

This design intentionally does not vendor the full theme and does not attempt to preserve the current one-page JavaScript shell. The new site will follow Minimal Mistakes conventions so it remains maintainable and easy to extend on GitHub Pages.

## Target structure

The repository will be reorganized into a standard Minimal Mistakes-backed Jekyll site:

- `_config.yml` for site identity, theme selection, skin, plugins, author metadata, and navigation settings.
- `index.md` or an equivalent homepage page with front matter and Markdown content.
- `_pages/` for standalone pages such as About, Projects, or Links.
- `assets/` for site-owned assets that remain necessary after the migration.
- `assets/images/` as the preferred long-term location for images used by page content.

The old static entrypoint and its layout-specific CSS and JavaScript are removed unless a specific asset is still needed as content.

## Theme and platform configuration

The site will use:

- `remote_theme: "mmistakes/minimal-mistakes"`
- `minimal_mistakes_skin: "catppuccin_mocha"`

The Jekyll configuration will also include the GitHub Pages-compatible plugin setup required for Minimal Mistakes remote theme usage. Site title, description, author metadata, and social links will move into `_config.yml` so the theme can render them consistently.

## Content migration

The current site content will be normalized into Jekyll pages instead of preserving the current custom section-switching behavior.

Recommended mapping:

- Intro content becomes the homepage introduction.
- About content becomes either a dedicated About page or a homepage section, depending on fit.
- Work content becomes a Projects page, with room to split into separate pages or posts later.
- Social and profile links move into theme-managed configuration.
- Existing images are reused and referenced from Jekyll content rather than from hardcoded HTML blocks.

This intentionally trades exact parity with the current one-page interaction model for a simpler, content-managed structure that is easier to maintain.

## Non-goals

This migration does not:

- Recreate the old single-page animated shell inside Minimal Mistakes.
- Vendor the entire Minimal Mistakes theme into the repository.
- Introduce custom styling beyond selecting the `catppuccin_mocha` skin unless required to complete the migration.

## Asset handling

Existing images may be kept temporarily in place during cutover, but references should move toward `assets/images/` paths for consistency with Minimal Mistakes documentation and future maintenance.

Old custom CSS and JavaScript should be removed once they are no longer used by any migrated content.

## Migration strategy

1. Add Jekyll and Minimal Mistakes configuration to `mrpurple666.github.io`.
2. Create the homepage and supporting pages using Jekyll front matter and Markdown.
3. Move site identity, social links, and author metadata into `_config.yml`.
4. Repoint content image references to stable site asset paths.
5. Remove the old static HTML shell and any unneeded theme-specific CSS or JavaScript from the previous site.

## Verification

After implementation, verify:

- the site builds locally with the expected Jekyll and GitHub Pages-compatible setup;
- the homepage renders with the Minimal Mistakes theme and `catppuccin_mocha` skin;
- navigation resolves correctly;
- images render correctly from their migrated paths;
- social/profile links render from configuration;
- the previous static landing page is no longer the active site shell.

## Risks and decisions

### Replacing in place

The migration is approved to happen directly in `mrpurple666.github.io` rather than in a staging subfolder. That keeps the repo clean, but it means the cutover must be deliberate and complete.

### Theme source

Using the remote theme is preferred over vendoring the theme because it minimizes maintenance burden and matches the intended GitHub Pages workflow.

### Exact design parity

Exact visual and interaction parity with the current landing page is intentionally not a requirement. The goal is a clean Minimal Mistakes site using the selected Catppuccin skin, not a theme port of the old implementation.
