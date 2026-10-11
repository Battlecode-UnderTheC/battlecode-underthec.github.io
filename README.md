# battlecode-underthec.github.io

UnderTheC's blog on GitHub Pages:
https://battlecode-underthec.github.io/

GitHub Pages builds it with Jekyll on every push; nothing to build by hand.

## Writing a post

Add `_posts/YYYY-MM-DD-slug.md`:

```markdown
---
layout: post
title: "How we approached it"
description: "One sentence for the index and the post's subtitle."
---

Markdown (GitHub-flavoured): tables, fenced code with highlighting, images in `assets/`.
```

Posts are listed oldest first on the front page (a series reads in order), with previous/next
links between them. Put images and videos under `assets/` and link them with
`{{ '/assets/name.png' | relative_url }}`.

## Previewing locally

    bundle exec jekyll serve      # with the github-pages gem, or: docker run -p 4000:4000 -v $PWD:/srv/jekyll jekyll/jekyll jekyll serve

then open http://localhost:4000/.
