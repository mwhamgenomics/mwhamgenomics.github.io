---
extends: post.html
title: Publishing to Pages with GitHub Actions
date: 2024-06-16
category: programming
tags: ['web development']
external_links:
    - title: GitHub Actions
      link: 'https://github.com/features/actions'
      description: Homepage for GitHub Actions
    - title: Git Hub Actions repos
      link: 'https://github.com/actions'
      description: Git repos for individual actions
---

A long time ago at this point, I re-did the way this site was published to GitHub Pages, using a combination of
markdown, custom Python and janky Git-based workflow where I was building the output HTMLs to a separate branch for
publishing. Fortunately, several advancements have been made since then and indeed since the old days of publishing with
Jekyll, namely better repo publishing options, and GitHub Actions.

GitHub Actions is similar to TravisCI or GitLab runners, where you write a yaml file of actions to be performed upon
changes made to the repo. The main difference from existing third-party solutions is integration with the rest of
GitHub.

Actions include custom shell commands (meaning that you can do virtually anything) and pre-built actions, which you can
browse and view documentation for at https://github.com/actions. Below we ill use these to automatically update a GitHub
Pages site whenever we make changes to the source.

## Building an Actions workflow

We start by creating a Yaml file at `.github/workflow/build.yaml`:

```yaml
name: Some build step for a project

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false
```

The `on` block controls when the action will run, and the `workflow_dispatch` will allow it to be run manually.
`permissions` controls the access given to the repo by the container that will eventually run the action. We don't
really want concurrency since there's only one repo and one Pages site to build.

Next, we create one or more jobs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v4

      - name: set up Python
        uses: actions/setup-python@v3
        with:
          python-version: 3.11

      - name: dependencies
        run: python -m pip install git+https://github.com/mwhamgenomics/Pykyll.git

      - name: run build
        run: python -m pykyll

      - name: upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './build'

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Here, we create two jobs, `build` and `deploy`. Firstly, `build` will use an Ubuntu container to run a series of steps.
These can be pre-built actions, e.g. the one that contain `uses`, or custom shell commands. In this job, we check out
the repo, set up Python and dependencies, and build the static site to an output directory. Finally, we create an
artifact, which is how GitHub Actions stores and transfers data and files. `upload-pages-artifact` will by default
create an artifact called `github-pages`.

`deploy` will run after `build` due to the use of `needs`. We use `environment` to set some environment variables
required by the pre-built deploy-pages action. This action takes an artifact, by default called `github-pages` (also the
default artifact name output by `upload-pages-artifact`, which is how it picks it up), and publishes it to Pages.

Finally on pushing to the repo, or adding it via the Actions tab, you can navigate to Actions and view your builds, kick
off new ones and debug runs that failed. I can guarantee that you'll need the latter - welcome to CI, I guess.
