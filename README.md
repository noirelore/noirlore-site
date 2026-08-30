# noirlore-site

The Hugo source for [noirlore.com](https://noirlore.com) — Noir Lore, long-form crime storytelling.

## Requirements

Hugo **extended**, version **0.147.2** — the version CI pins in `.github/workflows/deploy.yml`. Match it locally; a different minor version can render differently.

```sh
brew install hugo   # or download the pinned release directly
hugo version        # expect 0.147.2+extended
```

## Local development

```sh
hugo server --buildDrafts     # http://localhost:1313, drafts visible
hugo --minify                 # production build into public/
```

New posts are drafts by default and are **excluded from production builds**. A post stays invisible on the live site until you set `draft: false`.

```sh
hugo new stories/the-case-name.md
```

## Content model

One content section, plus taxonomies.

| Path | Purpose |
|---|---|
| `content/stories/` | Every case. One file per episode. |
| `content/about.md` | The About page. |

Grouping is done with the **`series`** taxonomy rather than extra sections, so a multi-part case stays together without new top-level URLs.

Front matter, per `archetypes/stories.md`:

| Field | Notes |
|---|---|
| `title` | Post title. |
| `date` | Drives ordering — newest first. |
| `draft` | `true` until ready to publish. |
| `series` | List. First entry renders as the label on cards and the post header. |
| `episode` | Integer. Renders as the "Episode N" marker. |
| `description` | The card summary. **Cards use this, not the body text.** |
| `image` | Optional cover image. |
| `tags` | Optional. |

Generated routes: `/stories/`, `/stories/<slug>/`, `/series/<series-slug>/`, `/tags/<tag>/`, `/categories/<category>/`.

## Editing without a terminal

Decap CMS is at [noirlore.com/admin/](https://noirlore.com/admin/), configured in `static/admin/config.yml`. It commits straight to `main`, which triggers a deploy.

The CMS collection folder must stay in sync with the content section. If you rename or move a section in `hugo.yaml`, update `config.yml` in the same commit — otherwise the CMS writes posts into a directory Hugo no longer reads and they silently never appear.

## Deployment

Push to `main`. `.github/workflows/deploy.yml` builds with Hugo, syncs `public/` to S3 with `--delete`, and invalidates CloudFront.

`public/` is **not** tracked in git — CI rebuilds it on every run.

## Infrastructure

The site is a private S3 bucket behind CloudFront. The bucket is not publicly readable and has no website-hosting configuration; CloudFront is the only reader.

| Resource | Value |
|---|---|
| AWS account | `khaos-root` profile |
| S3 bucket | `noirlore.com` (us-east-1, private) |
| CloudFront distribution | `EJYX7JBLKBD4L` |
| Origin Access Control | `E2KR5TZ3U0ENUP` |
| CloudFront Function | `noirlore-dir-index` (viewer-request) |
| Route 53 hosted zone | `Z06654323OBIL5FTVKWS3` |
| Domain registrar | GoDaddy, nameservers delegated to Route 53 |

Two details are easy to trip over and are explained in [`docs/adr/0001-cloudfront-oac-origin.md`](docs/adr/0001-cloudfront-oac-origin.md):

- A REST origin does **not** resolve `/stories/` to `/stories/index.html`. The `noirlore-dir-index` CloudFront Function does that rewrite. Changing it wrong takes every directory URL offline at once.
- S3 returns **403**, not 404, for a key that does not exist when the reader lacks `s3:ListBucket`. Both 403 and 404 are mapped to `/404.html` in the distribution's custom error responses.
