# Contributing

This blog is built with [Jekyll](https://jekyllrb.com/) and published via GitHub Pages. The sections below describe how to set up the environment, run the site locally while writing new posts, and publish changes.

## Prerequisites

- Ruby 3.2 (matches the version used by the GitHub Pages workflow in `.github/workflows/jekyll.yml`)
- Bundler (`gem install bundler`)
- Git

On Linux, `ruby-dev` / `build-essential` (Debian/Ubuntu) or the `base-devel` group (Arch) are required to compile native gem extensions. On macOS, Xcode Command Line Tools (`xcode-select --install`) are sufficient.

### Installing Ruby

The simplest approach is to use a version manager so the project's Ruby is isolated from the system Ruby.

With [rbenv](https://github.com/rbenv/rbenv):

```bash
rbenv install 3.2.4
rbenv local 3.2.4
```

With [asdf](https://asdf-vm.com/):

```bash
asdf plugin add ruby
asdf install ruby 3.2.4
asdf local ruby 3.2.4
```

Verify:

```bash
ruby --version   # ruby 3.2.x
```

## Installing dependencies

From the repository root:

```bash
bundle config set --local path 'vendor/bundle'
bundle install
```

This installs `github-pages` (which pins Jekyll and plugins to the versions GitHub Pages uses in production) into `vendor/bundle`, avoiding any writes to the system gem directory. `vendor/` and `.bundle/` are already excluded from the built site via `_config.yml`.

## Running the site locally

Start the development server:

```bash
bundle exec jekyll serve --livereload
```

The site is served at <http://127.0.0.1:4000/blog/> — note the `/blog` path, which comes from `baseurl` in `_config.yml`. `--livereload` rebuilds and refreshes the browser whenever you change a file.

Useful flags:

- `--drafts` — also render files in `_drafts/` (unpublished work-in-progress posts)
- `--future` — render posts dated in the future
- `--incremental` — faster rebuilds during editing (may occasionally miss cross-file changes; restart if something looks stale)
- `--host 0.0.0.0` — expose the server on the LAN (useful for testing on a phone)
- `--port 4001` — use a different port

Example used while drafting:

```bash
bundle exec jekyll serve --livereload --drafts --future --incremental
```

To produce a one-off production build without serving:

```bash
JEKYLL_ENV=production bundle exec jekyll build
```

The output is written to `_site/`.

## Writing a new post

1. Create a file under `_posts/` named `YYYY-MM-DD-title-in-kebab-case.md`. The date in the filename determines the publish date and the URL (see `permalink` in `_config.yml`: `/:year/:month/:day/:title/`).
2. Start the file with front matter:

   ```yaml
   ---
   layout: default
   title: "Your post title"
   date: 2026-04-18
   ---
   ```

3. Write the body in GitHub-Flavored Markdown (configured via `kramdown` with `input: GFM`). Fenced code blocks are highlighted by Rouge:

   ````markdown
   ```python
   def hello():
       print("hi")
   ```
   ````

4. Put images and other static assets under `assets/` and reference them with `{{ site.baseurl }}/assets/...` so the `/blog` base URL is applied correctly.

### Drafts

For work-in-progress posts, place the file in `_drafts/` (no date in the filename needed) and run the server with `--drafts`. Move the file to `_posts/` with a dated filename when it is ready to publish.

## Before committing

- Confirm the post renders correctly at <http://127.0.0.1:4000/blog/> and follow the link from the index page.
- Check the browser console for 404s on assets — these usually indicate a missing `{{ site.baseurl }}` prefix.
- Run a clean production build to catch errors that only surface outside of incremental mode:

  ```bash
  bundle exec jekyll clean
  JEKYLL_ENV=production bundle exec jekyll build
  ```

## Publishing

Pushing to `master` triggers `.github/workflows/jekyll.yml`, which builds the site with `JEKYLL_ENV=production` and deploys it to GitHub Pages. The live site is available at <https://amoilanen.github.io/blog>.

There is nothing to publish manually — a successful workflow run is the deploy.

## Troubleshooting

- **`bundle install` fails building native extensions** — install the system build toolchain (`build-essential` / `base-devel` / Xcode CLT).
- **`Could not find gem … in locally installed gems`** — run `bundle install` again; your local `Gemfile.lock` may have drifted.
- **Changes do not appear** — stop the server, run `bundle exec jekyll clean`, and start it again. Incremental builds occasionally miss changes to layouts or `_config.yml`. Changes to `_config.yml` always require a server restart.
- **Wrong URLs on a local build** — the site is served under `/blog/` because of `baseurl`. Always link assets with `{{ site.baseurl }}/…`.
- **Ruby version mismatch with CI** — the workflow uses Ruby 3.2. If a post builds locally on a newer Ruby but fails in CI, reproduce with `rbenv local 3.2.4` (or equivalent) and `bundle install` from scratch.
