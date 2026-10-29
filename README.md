# abbasi26.github.io

Source for <https://abbasi26.github.io>: project overview and write-ups for the tools in [github.com/Abbasi26](https://github.com/Abbasi26).

Built with [Hugo](https://gohugo.io) (no theme, no JavaScript). RSS: `/index.xml` (all posts) and `/posts/index.xml`.

## Local preview

```sh
hugo server
```

## Deploy

`.github/workflows/pages.yml` builds on every push and pull request. It deploys to GitHub Pages only after Pages is enabled under **Settings → Pages → Build and deployment → Source: GitHub Actions**; before that the deploy job is skipped.

## Writing a post

Add `content/posts/<slug>.md` with `title`, `date`, `description`, `tags` and `repo`. The canonical URL for cross-posts is `https://abbasi26.github.io/posts/<slug>/`.
