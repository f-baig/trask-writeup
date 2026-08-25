# Trask/RaceLab write-up

A standalone Astro project for a long-form Trask project write-up.

## Local development

```sh
pnpm install
pnpm dev
```

Edit the article in `src/pages/index.astro`.

## Publishing

The included workflow publishes to GitHub Pages whenever `main` is pushed. After creating the GitHub repository, open **Settings → Pages** and choose **GitHub Actions** as the source.
