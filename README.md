# EclipseStore Documentation Site

Build configuration for <https://docs.eclipsestore.io/> - the [Antora](https://antora.org/) playbook,
the supplemental UI, and the workflow that publishes the site.

**The documentation text does not live here.** It lives in
[eclipse-store/store](https://github.com/eclipse-store/store) under `docs/` on the branches `main`,
`release/1x`, `release/2x` and `release/3x`. To change wording, examples or page structure, open a
pull request against *that* repository. This repository only decides how those sources are turned
into a website.

## Repository layout

| Path | Purpose |
| --- | --- |
| `antora-playbook.yml` | The production playbook. Pulls the four branches of `eclipse-store/store` and writes to `docs/`. |
| `antora-playbook-local.yml` | Authoring playbook. Builds from a sibling `./eclipse-store` checkout into the gitignored `site/`. |
| `supplemental-ui/` | Header, footer, favicons, Whitney fonts and `site-extra.css` layered on top of the stock Antora UI. |
| `ui/ui-bundle.zip` | The vendored Antora default UI bundle. |
| `overlay/` | Files copied into `docs/` after every build that Antora does not generate. |
| `docs/` | **Generated.** The published website. Served by GitHub Pages. |
| `.github/workflows/publish-docs.yml` | The publishing automation. |

## Publishing

**Actions → Publish docs → Run workflow.**

The workflow builds the site, applies `overlay/`, commits `docs/` to `main` and asks GitHub Pages to
redeploy. If the build produces no content change it makes no commit.

The trigger is manual by design. Because the prose lives in another repository, **nothing rebuilds
the site automatically** - after a documentation change is merged in `eclipse-store/store`, someone
has to press *Run workflow*.

There is a `dry_run` input. With it enabled the workflow builds the site and uploads it as a
downloadable artifact without committing anything - useful for checking a playbook or UI change
before it reaches the live site.

## Building locally

```bash
npm ci
npm run build     # -> docs/
npm run serve     # -> http://localhost:8080
```

To preview documentation that has not been merged yet, check out `eclipse-store/store` as a sibling
directory named `eclipse-store` and run `npm run build:local`. That writes to `site/`, which is
gitignored, so it can never be published by accident.

Node 22 is pinned in `.nvmrc`.

## `docs/` is generated

Never hand-edit anything under `docs/`. The playbook sets `output.clean: true`, so the whole
directory is deleted and rebuilt on every run and any manual change is lost. Pages removed upstream
disappear from the site automatically.

## `overlay/`

Antora produces exactly six top-level entries: `404.html`, `_/`, `manual/`, `robots.txt`,
`search-index.js` and `sitemap.xml`. Everything else the published site needs lives in `overlay/`
and is copied over the build output afterwards. Anything added to `overlay/` lands at the root of
the published site.

- **`overlay/.nojekyll`** - load-bearing. GitHub Pages runs a legacy Jekyll build, and Jekyll
  excludes underscore-prefixed directories. Without this file the entire `docs/_/` directory is
  dropped and the site loses all of its CSS, JavaScript and fonts.
- **`overlay/CNAME`** - `docs.eclipsestore.io`, keeping the custom domain consistent with the Pages
  settings.

The root `index.html` redirect is *not* in `overlay/` - it is generated from `site.start_page` in
the playbook, so it always follows the current landing page.

## GitHub Pages configuration

Source: branch `main`, folder `/docs`, legacy (Jekyll) build type, custom domain
`docs.eclipsestore.io`. The legacy build type only accepts `/` or `/docs` as the folder, which is
why the output directory is named `docs`.

## Updating the vendored UI bundle

The Antora default UI is committed rather than downloaded at build time, so builds are reproducible
and are not affected by GitLab availability. To pick up a newer UI:

```bash
curl -fSL -o ui/ui-bundle.zip \
  "https://gitlab.com/antora/antora-ui-default/-/jobs/artifacts/master/raw/build/ui-bundle.zip?job=bundle-stable"
npm run build
```

Review the resulting diff and open a pull request.

## Rollback

1. **Bad content, automation working** - revert the `Rebuild site` commit on `main`, then
   `gh api -X POST repos/eclipse-store/docs-site/pages/builds`. Live again in about a minute.
2. **Publishing scheme broken** - repoint Pages at the archived branch:
   ```bash
   gh api -X PUT repos/eclipse-store/docs-site/pages \
     -f 'source[branch]=site' -f 'source[path]=/'
   ```
3. **Workflow misbehaving** - Actions → *Publish docs* → *Disable workflow*. The site freezes in its
   current state; nothing is lost.

## The `site` branch

Until July 2026 the site was published by building locally and copying the output onto a separate
`site` branch. That branch is kept as a frozen archive and rollback target. It is no longer updated.

## Known quirks

- **Two files change on every build regardless of content**, and the workflow excludes both from
  change detection - otherwise every run would commit around 3 MB of noise:
  - `sitemap.xml` embeds the build timestamp in every `<lastmod>`.
  - `search-index.js` is not generated deterministically by `@antora/lunr-extension`. Three
    consecutive builds of identical content produced three different files.

  This cannot mask a real update: any documentation change also rewrites HTML, because every page
  embeds the navigation tree.
- `search-index.js` is around 3 MB. When reviewing a diff of the generated site, exclude it:
  ```bash
  git show --stat HEAD -- docs ':(exclude)docs/search-index.js' ':(exclude)docs/sitemap.xml'
  ```
