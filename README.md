# gha-wp-deploy

Reusable GitHub Actions workflows for deploying a WordPress theme to a shared
web host — over FTP or SFTP only, nothing else required on the server side.

## Why this exists

A lot of shared/cheap WordPress hosts don't give you full SSH access — no
`git`, `rsync`, `wp-cli`, or a way to run a build step on the server. FTP (or
basic SFTP) is often the only door in.

This repo holds the deploy logic — build the theme, diff it against the last
successful deploy, push the changed files over `lftp` — as a **reusable
GitHub Actions workflow**, so every project doesn't need its own copy-pasted
`.yml` and `.sh` files. One place to fix bugs and add features; every project
picks up the fix by bumping a version tag.

What it does, per deploy:

1. Builds the theme (`npm ci && npm run <build-command>`).
2. Works out what changed since the last **successful** deploy — using a
   marker file (`.deploy-sha`) written to the server after every successful
   run, so a run of failures in between doesn't lose track of what's stale.
3. Uploads only the changed/built files over FTP/SFTP (`lftp`), in chunks if
   configured, reconnecting between chunks.
4. Deletes files on the server that were removed from git.
5. Updates the `.deploy-sha` marker.

A separate manual-only workflow (`remote-cleanup.yml`) lets you delete a
single stray file/path on the server from the GitHub Actions tab (handy from
a phone) — for cases like a leftover build artifact uploaded before it was
git-ignored.

## What lives where

- **This repo** — the reusable workflows (the *how*). Nothing project-specific.
- **Each project repo** — a tiny caller workflow, plus its own
  `.github/config/deploy.json` (the *where*: host, remote path, themes,
  chunk size) and its own FTP credentials as repo secrets.

## Setup in a project repo

**1. Add `.github/config/deploy.json`:**

```json
{
    "staging": {
        "enabled": true,
        "uploadAll": false,
        "clean": false,
        "chunkSize": 100,
        "themes": ["your-theme"],
        "remotePath": "/public_html/staging.example.com/wp-content/themes",
        "ftpHost": "203.0.113.10",
        "protocol": "ftp"
    },
    "production": {
        "enabled": true,
        "uploadAll": false,
        "clean": false,
        "chunkSize": 100,
        "themes": ["your-theme"],
        "remotePath": "/public_html/wp-content/themes",
        "ftpHost": "203.0.113.10",
        "protocol": "ftp"
    }
}
```

| field       | meaning                                                             |
| ----------- | -------------------------------------------------------------------- |
| `enabled`   | set `false` to no-op a deploy without deleting the workflow           |
| `uploadAll` | force a full re-upload instead of diffing against the last deploy    |
| `clean`     | wipe the remote theme folder before uploading (implies `uploadAll`)  |
| `chunkSize` | files per FTP connection/chunk; `0` = one connection for everything  |
| `themes`    | theme folder name(s) under `themes/` to deploy                       |
| `remotePath`| absolute path on the server to the `themes` parent directory         |
| `ftpHost`   | FTP/SFTP host                                                        |
| `protocol`  | `ftp` or `sftp`                                                      |

**2. Add repo secrets** (Settings → Secrets and variables → Actions):
`STAGING_FTP_USERNAME`, `STAGING_FTP_PASSWORD`, `PRODUCTION_FTP_USERNAME`,
`PRODUCTION_FTP_PASSWORD` — whichever environments you use.

**3. Add caller workflows** — see [WORKFLOWS.md](WORKFLOWS.md) for the exact
files to copy in, and for this repo's own workflow source if you want to read
or fork it.

## Versioning

Tag releases as `vX.Y.Z` (SemVer), and keep a moving major tag (`v1`)
pointing at the latest `v1.x.y`:

```sh
git tag v1.1.0
git tag -f v1 v1.1.0
git push origin v1.1.0 v1 --force
```

Projects pin `uses: <owner>/gha-wp-deploy/.github/workflows/deploy.yml@v1` to
get fixes automatically, or an exact `@v1.1.0` to stay frozen. A breaking
change (renamed/removed input) bumps to `v2` so nothing pinned at `@v1` moves
until it opts in.
