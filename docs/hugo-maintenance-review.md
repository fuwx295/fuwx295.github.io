# Hugo and Stack upgrade

Release prepared and validated on November 7, 2026. The release includes the English network troubleshooting post, its redesigned cover, and the Hugo / Stack upgrade. Deployment status is recorded in the repository's GitHub Actions history.

## Versions

| Component | Before | Validated version |
|---|---|---|
| Local Hugo CLI / preview | `0.166.0` | Homebrew `0.169.0` extended/deploy |
| GitHub Pages build | `0.141.0` | Pinned to `0.169.0` |
| Stack theme | `v3.30.0` | `v4.0.3`, submodule commit `3e123a30b79b5d52a3a8e88a9dd678fcfd28e418` |

These are the latest stable [Hugo](https://github.com/gohugoio/hugo/releases/tag/v0.169.0) and [Stack](https://github.com/CaiJimmy/hugo-theme-stack/releases/tag/v4.0.3) releases checked on this date. The migration follows the official [Stack v4 guide](https://stack.cai.im/guide/upgrade-to-v4).

## Changes

- Migrated `languageCode` to `locale` and `languageName` to `label`.
- Migrated the sidebar avatar to the v4 path format and added the favicon under `assets`, preserving the existing root favicon URL.
- Removed obsolete image configuration and renamed `imageProcessing.cover` to `imageProcessing.thumbnail`.
- Moved the footer override and music shortcode to `layouts/_partials` and `layouts/_shortcodes`. The music shortcode now uses the page store instead of the nested scratch object initialized by the old theme.
- Disabled automatic generation of `assets/jsconfig.json`, so builds do not rewrite that tracked editor configuration.
- Updated the Pages workflow to Hugo `0.169.0` and the `Asia/Shanghai` time zone.
- Added CI checks for the installed Hugo version and the exact Stack `v4.0.3` submodule tag.
- Set explicit URLs on the MIT Lab 3 and Lab 4 posts to preserve their published `key/value-service` paths; newer automatic slug generation replaces that slash with a hyphen.
- Saved the Links page's existing GitHub logo in its page bundle. Stack v4 fetches remote images during builds; using the local copy avoids failures when a local proxy resolves the remote host to a restricted address.
- Added the personal background to the new article: intermittent website failures, the Wi-Fi-first workaround, and the eventual discovery of the extra cable.
- Reworked the article around a TL;DR, three evidence checks, the minimal fix, cache cleanup, verification, and a retrospective.
- Replaced the text-heavy diagram cover with a panoramic editorial illustration, optimized as a JPEG. Generation details and final prompts are in [network-cover-prompt.md](network-cover-prompt.md).

## Validation

- Production build with Hugo `0.169.0` extended and Stack `v4.0.3`: passed.
- Existing published sitemap URLs are preserved.
- Article cover, avatar, favicon, stylesheet, and table-of-contents targets: passed.
- Homepage, archives, About, Links, RSS, sitemap, and the search index: passed.
- Browser search for `gateway`: returns the new article and opens it correctly.
- Waline server configuration and comment markup are retained; no test comment was submitted.

Stack `v4.0.3` still uses Hugo's default LibSass pipeline. Hugo reports one deprecation warning about that upstream dependency; builds and styling work. A future change can switch to [Dart Sass](https://gohugo.io/functions/css/sass/#dart-sass) with matching local and CI dependencies.

## Local preview

The development server runs at `http://localhost:1313/`, with the new article at `/p/two-devices-one-gateway-ip/`.

To restart it:

```sh
hugo version
hugo server --bind 127.0.0.1 --port 1313 --baseURL http://localhost:1313/ --disableFastRender --noHTTPCache
```

The release includes the `themes/hugo-theme-stack` submodule pointer together with the migrated configuration and templates. The Pages workflow checks out submodules recursively and checks the exact theme tag before building.
