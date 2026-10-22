---
title: How This Site Is Built
description: Markdown files, a small PHP static site generator, and one GitHub Actions workflow. The whole publishing pipeline behind melihucar.com.
date: 2026-10-21T13:01Z
tags: [Verse, AstroPHP, Cloudflare Pages, Static Sites]
---

This is the first article written for this site rather than linked from somewhere else, so it seems fair to start with the site itself: what it is made of, how a Markdown file becomes a page at melihucar.com, and what I deliberately left out.

## The short version

Every page here is a Markdown file in a Git repository. When I push to `main`, a GitHub Actions workflow builds the site with [Verse](https://github.com/astrophp/verse) and publishes the result to Cloudflare Pages. There is no server, no database, and no admin panel.

![Markdown is pushed to main; GitHub Actions runs verse build; dist/ is deployed to Cloudflare Pages and served at melihucar.com.](pipeline.svg "From a Markdown file to melihucar.com")

## Content is just files

The repository is small enough to describe in full:

```text
verse.yaml
content/
  about.md
  projects.md
  articles/
    how-this-site-is-built/
      page.md
      pipeline.svg
.github/workflows/deploy.yml
```

`verse.yaml` holds the site-wide settings: title, URL, description, and social links. Everything else is content. An article is either a single Markdown file or, like this one, a folder with a `page.md` and the assets it uses. Keeping the diagram next to the text means I can move or rename the article without hunting down broken image paths.

Each file starts with a little front matter:

```yaml
---
title: How This Site Is Built
description: Markdown files, a small PHP static site generator, and one GitHub Actions workflow.
date: 2026-10-21
tags: [Verse, AstroPHP, Cloudflare Pages, Static Sites]
---
```

Some articles here live on Medium. For those, the front matter has an `external_url` and the article list links straight to it. Writing a page natively or pointing to it elsewhere is one line of difference. From now on, this site is where articles are published first; Medium is there for reach.

## Why Verse

Verse is a static site generator I'm building under [AstroPHP](https://github.com/astrophp). I spend most of my time in PHP and Laravel, and I wanted the tool behind my own writing to live in the same ecosystem: templates are Blade, content is Markdown, and the output is plain HTML.

The other reason is simpler: building things is fun, and it teaches you more than using them. Writing a static site generator means deciding how Markdown becomes HTML, how links between pages resolve, and what a build should refuse to publish. Those are small questions, but answering them yourself is a good way to learn.

## The whole deploy pipeline

This is the entire workflow:

```yaml
name: deploy

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  deployments: write

concurrency:
  group: deploy-${{ github.head_ref || github.ref_name }}
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: intl, mbstring
          coverage: none

      - name: Install Verse
        run: |
          composer global require astrophp/verse --no-interaction --no-progress
          echo "$(composer global config bin-dir --absolute --quiet)" >> "$GITHUB_PATH"

      - run: verse build

      # main deploys to production; any other branch gets a preview URL.
      - uses: cloudflare/wrangler-action@v4.1.3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
          preCommands: wrangler pages project create melihucar-com --production-branch=main || true
          command: pages deploy dist --project-name=melihucar-com --branch=${{ github.head_ref || github.ref_name }}
```

The job installs PHP, installs Verse with Composer, and runs `verse build`, which writes the finished site to `dist/`. Wrangler, Cloudflare's CLI, then uploads that folder to Cloudflare Pages. Nothing is committed back to the repository; there is no `gh-pages` branch to keep in sync.

The branch name decides where a build lands. A push to `main` goes to production at melihucar.com, while a pull request gets its own preview URL, so I can read a new article on the real stack before merging it. The `concurrency` block cancels an older run for the same branch when a newer push arrives.

The Cloudflare API token and account ID live in repository secrets. The `deployments: write` permission lets Wrangler record each deploy as a GitHub deployment, which is how the preview link shows up on the pull request.

### Why not GitHub Pages

The first version of this pipeline deployed to GitHub Pages, and the build side worked fine. The custom domain did not: GitHub got stuck at the DNS verification step for melihucar.com, and without a verified domain it would not let me turn on HTTPS. Rather than keep waiting on it, I moved the deploy step to Cloudflare Pages. There, HTTPS was on as soon as the domain was attached, and branch previews came along for free. The build itself did not change at all; only the last step did.

## Writing locally

Locally, the loop is two commands:

```bash
verse serve
verse check
```

`verse serve` previews the site and rebuilds on every save. `verse check` validates the site without writing anything, which is a quick sanity pass before pushing. Articles marked `draft: true` show up in the preview with a badge but are never part of a real build, so unfinished work can sit in the repository safely.

## What I left out

I want this site to stay slim: HTML and CSS first, everything else only when it earns its place.

- **Comments.** They would mean a third-party script or a server, and neither fits a site that is just files. If something here is worth discussing, I'd rather hear about it on [LinkedIn](https://www.linkedin.com/in/melihucar/) or in a [GitHub issue](https://github.com/melihucar/melihucar.com/issues).
- **A JavaScript framework.** JavaScript is used only where a feature needs it, such as the copy button on code blocks.

> [!NOTE]
> A newsletter is planned for a future release. Until then, [LinkedIn](https://www.linkedin.com/in/melihucar/) and [GitHub](https://github.com/melihucar) are the best places to follow along.

## What comes next

The articles here will mostly be about software architecture, AI systems, and the Laravel ecosystem: following a problem from the first question to the decisions that remain. This one had a second job too. It is the first page to go through the entire pipeline above, so if you're reading it at melihucar.com, that part works.
