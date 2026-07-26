---
title: Trigger a GitHub Pages Rebuild from Another Repository
date: 2026-07-26
---

# Trigger a GitHub Pages Rebuild from Another Repository

My personal website, josephali.dev, is hosted on Github pages, with it's source code stored in the `dig/josephali.dev` repository.

Whenever I push a change to the `main` branch, a Github actions workflow builds the Next.js application and deploys to Github pages.

Previously, my TIL notes lived inside the website repository. I wanted to move them into a separate repository `dig/notes`, while still rebuilding the website whenever a note was added or changed.

The solution was to use a `repository_dispatch` event that Github supports across repositories.

## Dispatching the Event

Inside the `dig/notes` repository, I created a Github actions workflow exclusively to send the dispatch event whenever a change is pushed to the `main` branch.

Here's what it looks like:

```yml
name: Notify site to rebuild

on:
  push:
    branches: [main]

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger josephali.dev rebuild
        uses: peter-evans/repository-dispatch@v4
        with:
          token: ${{ secrets.REBUILD_PAT }}
          repository: dig/josephali.dev
          event-type: notes-updated
```

The workflow uses the `peter-evans/repository-dispatch@v4` action to send a custom event called `notes-updated` to the website repository.

## Listening for the Event

In the `dig/josephali.dev` repository, I updated the existing workflow to listen for the dispatch event:

```yml
on:
  repository_dispatch:
    types: [notes-updated]
```

Now, whenever I push a change to the `main` branch of `dig/notes`, it sends the `notes-updated` event to the website repository and starts the build and deploy workflow.

Finally, I added this step to the website build workflow:

```yml
- uses: actions/checkout@v4
  with:
    repository: dig/notes
    path: tils
```

This clones the notes repository and places it at `{projectRootDir}/tils`. Exactly where the notes were located previously, no code changes needed!