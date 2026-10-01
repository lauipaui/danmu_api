# LogVar Danmu API — Fork Guide

[中文](README.md) | **English**

A JavaScript danmaku (on-screen comment) API compatible with DandanPlay-style search, matching, episode details and comment endpoints. This is a fork of [huangxd-/danmu_api](https://github.com/huangxd-/danmu_api); the original Chinese guide, credits and notices remain in [README.md](README.md). This guide covers the checked-out fork, not a promise that all third-party sources or hosts are currently operational.

## Features

- Search titles/episodes, match media filenames and fetch comments by ID or video URL.
- JSON or Bilibili-style XML output, segment-comment requests, filtering, deduplication, color/position conversion and optional Chinese character conversion.
- In-memory caching, local cache persistence in the Node deployment, and optional Upstash Redis persistence.
- A configuration/log/API-management UI and a buildable Forward widget.
- Node.js, Docker and adapter files for Vercel, Netlify, EdgeOne Pages and Cloudflare Workers.

Source availability depends on upstream services, authorization, regions and hosting limits. Use only content and accounts you are authorized to access, and respect platform terms.

## Requirements and local start

Use a supported Node.js LTS and npm; the Dockerfile uses Node 22. The upstream guide's historical Node 18 minimum is not a guarantee for every newer dependency. No `npm test` script is defined in this snapshot.

```sh
git clone https://github.com/lauipaui/danmu_api.git
cd danmu_api
npm install
cp config/.env.example config/.env
# Edit config/.env locally before starting: set a random TOKEN and separate ADMIN_TOKEN
npm start
```

The main HTTP listener binds to `0.0.0.0:9321`; `DANMU_API_PORT` overrides the port. **The Node entry also starts a proxy listener on `0.0.0.0:5321`.** Protect both ports rather than assuming only 9321 is exposed. Bind access through a firewall and authenticated HTTPS reverse proxy as appropriate.

System environment variables take precedence over `config/.env`. The Node entry watches that file for changes, but listener-port changes require a process restart. Container-injected environment variables override file values as well.

## Authentication and API use

The published default `TOKEN` is `87654321`, and requests can omit the token when that default is used. **It is not a production secret.** Configure an unpredictable custom token before making the service reachable.

Configure a client with a base URL such as:

```text
https://danmu.example.com/YOUR_PRIVATE_TOKEN
```

Append endpoint paths to that base URL:

| Method / path | Purpose |
| --- | --- |
| `GET /api/v2/search/anime?keyword=TITLE` | Search titles |
| `POST /api/v2/match` | Match a media title/filename; see the Chinese guide for request schemas |
| `GET /api/v2/search/episodes?anime=TITLE` | Search episodes |
| `GET /api/v2/bangumi/:animeId` | Title/episode details |
| `GET /api/v2/comment/:commentId?format=json` | Retrieve comments; `format=xml` selects XML |
| `GET /api/v2/comment?url=VIDEO_URL&format=json` | Retrieve by video URL; URL-encode the input |
| `POST /api/v2/segmentcomment?format=json` | Retrieve a segment using segment metadata returned by the comment API |
| `GET /api/logs` | Recent in-memory logs; protect access and redact before sharing |

Format priority is query parameter, environment setting, then JSON default. Player integration and detailed examples are retained in the Chinese guide. File/title naming and source order influence automatic matches; verify results instead of assuming every filename can be resolved.

## Docker

Build the current fork rather than accidentally running a separately published upstream image:

```sh
docker build -t danmu-api-local .
# Populate config/.env locally first. Publishing only to loopback limits direct access.
docker run -d --name danmu-api \
  -p 127.0.0.1:9321:9321 \
  -v "$PWD/config:/app/config" \
  --restart unless-stopped danmu-api-local
```

The source still starts port 5321 inside the container; it is not published in this example. Do not expose it without reviewing the proxy handler and access controls. A local build does not expose host ports until you publish them.

Mount `/app/config` for file-based hot reload. Mount `/app/.cache` if local cache persistence is needed. Use a private env file or secret storage; do not put real credentials in command examples or Git. `logvar/danmu-api:latest` is an **upstream image**, not a build of this fork; tags can change independently of this checkout.

## Serverless deployments

| Platform | Repository adapter/config | Notes |
| --- | --- | --- |
| Vercel | [`vercel.json`](vercel.json) | Import `lauipaui/danmu_api` explicitly, set secret env variables, redeploy after changes |
| Netlify | [`netlify.toml`](netlify.toml), [`netlify/functions/api.js`](netlify/functions/api.js) | Configure env values in the platform and verify generated routes |
| EdgeOne Pages | [`edgeone.json`](edgeone.json), [`node-functions/`](node-functions/) | Isolated/cold environments may lose in-memory IDs; optional Redis can help |
| Cloudflare Workers | [`wrangler.toml`](wrangler.toml), [`danmu_api/worker.js`](danmu_api/worker.js) | Verify request, CPU and outbound limits; provider restrictions can affect sources |

The one-click buttons retained in the Chinese guide point to **upstream** repositories. Change the import source to this fork if that is your intention. Platform plans, deployment regions and API behavior can change; no deployment was performed for this documentation update.

## Configuration reference

See [`config/.env.example`](config/.env.example) and [`danmu_api/configs/envs.js`](danmu_api/configs/envs.js) for the current fields. The [Chinese guide](README.md) retains the complete upstream variable table and filtering examples.

| Variables | Purpose / checked defaults |
| --- | --- |
| `TOKEN`, `ADMIN_TOKEN` | API token (public default `87654321`) and separate management token (empty by default) |
| `DANMU_API_PORT` | Node main listener, default 9321 |
| `SOURCE_ORDER`, `PLATFORM_ORDER` | Enabled sources/order and preferred match platforms |
| `MERGE_SOURCE_PAIRS` | Merge configured source groups |
| `OTHER_SERVER`, `CUSTOM_SOURCE_API_URL` | Fallback/custom source; enable `custom` in source order when used |
| `VOD_SERVERS`, `VOD_RETURN_MODE`, `VOD_REQUEST_TIMEOUT` | VOD search sources, `fastest`/`all` mode and request timeout |
| `BILIBILI_COOKIE`, `TMDB_API_KEY`, `PROXY_URL` | Optional provider credentials/proxy settings; keep private |
| `TITLE_TO_CHINESE`, `TITLE_MAPPING_TABLE`, `ANIME_TITLE_SIMPLIFIED` | Title translation/mapping/conversion |
| `ANIME_TITLE_FILTER`, `EPISODE_TITLE_FILTER`, `ENABLE_ANIME_EPISODE_FILTER`, `STRICT_TITLE_MATCH` | Search/episode filtering and matching rules |
| `BLOCKED_WORDS`, `GROUP_MINUTE`, `DANMU_LIMIT` | Comment filters, deduplication and sampling |
| `CONVERT_TOP_BOTTOM_TO_SCROLL`, `CONVERT_COLOR`, `DANMU_SIMPLIFIED_TRADITIONAL` | Output transformation |
| `DANMU_OUTPUT_FORMAT` | `json` or `xml`; query parameter overrides it |
| `RATE_LIMIT_MAX_REQUESTS` | Comment-endpoint limit; published default is 3 per IP per minute |
| `SEARCH_CACHE_MINUTES`, `COMMENT_CACHE_MINUTES` | Both default to **1 minute in current code**; older prose states 5 for comments |
| `REMEMBER_LAST_SELECT`, `MAX_LAST_SELECT_MAP` | Experimental selection memory; default capacity 100 |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | Optional persistent Redis REST backend |
| `DEPLOY_PLATFROM_ACCOUNT`, `DEPLOY_PLATFROM_PROJECT`, `DEPLOY_PLATFROM_TOKEN` | Platform-management fields; keep the actual `PLATFROM` spelling |
| `LOG_LEVEL` | Logging detail; review logs for private data before sharing |
| `NODE_TLS_REJECT_UNAUTHORIZED` | Keep certificate verification enabled; do not use `0` as a connectivity workaround |

A URL-path token can leak through access logs, browser history or Referer headers. Do not share the full client base URL, cookies, API credentials or provider configuration.

## Maintenance, tests and troubleshooting

```sh
node --test danmu_api/worker.test.js
node --test forward/forward-widget.test.js
npm run build-forward-widget
```

The Forward test resides under `forward/`, not `danmu_api/`. Read tests before running them if you need a strictly offline environment. These are available maintenance commands, **not a claim that this update ran every test or built/deployed the application**.

- No comments: first check the base API response, then player URL/spacing, title match and selected source. Inspect sanitized logs.
- Missing IDs after cold start: in-memory state is not durable. Consider the optional Redis backend and verify its credentials/availability.
- Slow or failing sources: check provider availability, authorized cookies, proxy settings and hosting limits individually.
- Update safely: pin the desired revision/image, back up local config/cache and maintain a rollback copy. Never commit those backups. Restore the prior revision/image and protected config on regression; this repository has no universal rollback command.

## Attribution and licensing

Upstream author/project: [huangxd-/danmu_api](https://github.com/huangxd-/danmu_api). Credits, contributor links, notices and detailed platform documentation remain in the Chinese guide. The upstream asks that the project not be promoted on mainland Chinese media platforms; preserve and review its notices.

The root [`LICENSE`](LICENSE) contains **AGPL-3.0**, while `package.json` currently labels the package `ISC`. This metadata inconsistency is not resolved by this documentation-only change. Confirm licensing with upstream before redistribution or network-service use, including applicable source-availability obligations. The project does not grant rights to third-party videos, comments, services or accounts.
