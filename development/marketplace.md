---
layout: page
title: Marketplace internals
parent: Development
nav_order: 8
summary: "How the marketplace reaches local and cloud users."
icon: 🏪
---

# Marketplace internals

The user side is on [Abilities](/abilities). This page is how the copies of the marketplace are kept.

The marketplace is its own repos: [kajaio/marketplace](https://github.com/kajaio/marketplace) (public) and
`kajaio/darkmarket` (private), each holding `abilities/`, `personas/` and `datasets/` at its root. There's no
publishing flow and no registry service. The owner commits a file, and each consumer (the terminal, the API,
the MCP sandbox) downloads the repos it's configured with on its own schedule. The copies are separate and
can sit at different commits: the terminal never talks to the API's copy, and the API never reads anyone's
disk.

All three use one fetcher, `@kaja/nasi`'s `sources.ts`:

- A source is a GitHub `owner/repo`, `owner/repo#ref` (default `main`), or a folder on disk (development).
- It asks GitHub for each repo's commit (`Accept: application/vnd.github.sha`), then downloads that commit's
  tarball (capped at 50 MB) and unpacks it with `Bun.Archive`, so nothing needs `git` or `tar`. A public
  repo comes from `codeload.github.com`; a token (needed for a private one) is only ever sent to
  `api.github.com`.
- Only `abilities/`, `personas/` and `datasets/` are kept, so a repo's CI files and README never reach users.
- Sources merge in order. A later one replaces an earlier one's whole ability folder, persona file or
  dataset, so two repos' files never mix inside one ability.

```mermaid
---
config:
  look: handDrawn
  theme: neo-dark
---
flowchart LR
    REPO[["<b>GitHub repos</b><br><small>kajaio/marketplace, kajaio/darkmarket</small>"]]

    subgraph LOCAL["your machine"]
        CACHE["merged cache<br><small>tarballs, later wins</small>"]
        FOLDER["~/.config/kaja/marketplace/<br><small>personas pick abilities</small>"]
    end

    subgraph CLOUD["the Kaja API"]
        SYNC["MarketplaceService<br><small>hourly, at start, on demand</small>"]
        PG[("Postgres<br><small>ability table</small>")]
    end

    SANDBOX["MCP sandbox<br><small>stdio manifests, at start</small>"]

    REPO -->|"kaja abilities update"| CACHE --> FOLDER
    REPO -->|"GitHub API + tarballs"| SYNC --> PG
    REPO -->|"public repo"| SANDBOX
```

Each kind has one manifest format, checked by the schemas in [`@kaja/schema/abilities`](/development/schema).
Names are lowercase letters, digits and single hyphens, up to 64 characters, and must match the folder
(skills) or file name (everything else). A file that fails validation is skipped with a warning by whoever
reads it. Binary files, hidden files and backups (`name.bak.ext`, `name.bak.2.ext`) are never served to the model.

## The terminal's copy

```mermaid
---
config:
  look: handDrawn
  theme: neo-dark
---
sequenceDiagram
    participant K as kaja abilities update
    participant G as GitHub
    participant C as merged cache
    participant F as marketplace folder

    K->>G: each source's commit, then its tarball
    K->>C: merge the sources, later ones winning
    C-->>K: the merged folder and each source's commit
    K->>F: sync against .sync-lock.json
    F-->>K: added, updated, backed up, removed, kept
```

- The sources are settings.toml's `[marketplace] sources` (default `kajaio/marketplace`), and a private one
  needs secrets.toml's `[marketplace] github_token`. Under `KAJA_PROFILE=dev` the default is a
  `../marketplace` checkout beside the Kaja source, read as a folder, so uncommitted edits sync too.
- A failed download leaves both the cache and the marketplace folder as they were.
- Each sync compares three things per file: the upstream copy, the user's copy, and the hash the previous
  sync recorded in `.sync-lock.json`:

```mermaid
---
config:
  look: handDrawn
  theme: neo-dark
---
flowchart TD
    A(["a file upstream"]) --> B{"user has it?"}
    B -->|no| ADD["copy it: added"]
    B -->|yes| C{"same as upstream?"}
    C -->|yes| SAME["nothing to do"]
    C -->|no| D{"same as what the last<br>sync wrote?"}
    D -->|"yes: untouched"| UPD["replace it: updated"]
    D -->|"no: edited"| BAK["save theirs as name.bak.ext,<br>then replace: backed up"]

    G(["a file the last sync wrote,<br>now gone upstream"]) --> H{"still untouched?"}
    H -->|yes| RM["delete it: removed"]
    H -->|"no: edited"| KEEP["keep it: kept"]
```

The sync keeps the executable bit on scripts and prunes folders it emptied. At startup the terminal loads
every valid ability and persona in the folder; each persona's `abilities` list decides what a chat gets.

## The API's copy

```mermaid
---
config:
  look: handDrawn
  theme: neo-dark
---
sequenceDiagram
    participant T as trigger
    participant S as MarketplaceService
    participant G as GitHub
    participant D as Postgres

    T->>S: startup, hourly, or admin button
    S->>G: GET /commits/ref for each source (sha only)
    alt same commits as the last sync
        S-->>T: nothing changed
    else a source moved
        S->>G: download each commit's tarball
        S->>S: merge, extract and validate every file
        S->>D: one transaction: upsert each ability,<br/>mark the missing ones removed
        S->>D: record the commits in marketplace_sync
    end
```

- **Triggers.** Once at API start-up (in the background), every hour on the hour, and the admins' **Sync
  now** button. Only one sync runs at a time.
- **Cheap when idle.** The check is one GitHub API call per source. The tarballs are only downloaded when a
  source's head moved. **Sync now** always downloads them, because a new API build may accept abilities the
  old one skipped on those same commits. A folder source (development) is re-read on every sync.
- **Which repos.** `MARKETPLACE_SOURCES` (comma-separated, default `kajaio/marketplace`), plus
  `MARKETPLACE_GITHUB_TOKEN` for a private one such as `kajaio/darkmarket`. The recorded commit is each
  source's `owner/repo#ref@sha`, comma-separated.
- **Failures** are recorded in the single `marketplace_sync` row, shown in the admin panel, and reported to
  Sentry. The previous catalog stays in place.
- **Nothing is deleted.** An ability that leaves the folder gets `removed_at`. Users' selections survive and
  come back if it returns. `updated_at` only moves when the content hash changes, and that hash covers just
  what the agent sees.

An ability that can't run in the cloud is skipped at sync time, and one that stops qualifying is hidden from
the catalog:

- a skill with `scripts/`;
- an HTTP tool on a non-public host, or needing a key when the server has no `USER_SECRET_KEY`;
- an MCP server that has no `tools` allowlist, is on a non-public host, is `stdio` and needs a key, or shares
  a name with an HTTP tool (a key belongs to one name). A keyless `stdio` one is always stored, and offered
  only while the API has an [MCP sandbox](https://github.com/kajaio/kaja/tree/main/apps/sandbox#readme)
  configured;
- anything whose manifest no longer parses.

## The sandbox's copy

The [MCP sandbox](https://github.com/kajaio/kaja/tree/main/apps/sandbox#readme) only needs the stdio
`mcp.toml` manifests. It fetches `MARKETPLACE_SOURCES` (default `kajaio/marketplace`) once at startup into
`SANDBOX_STATE_DIR/marketplace`, and keeps the last copy when that fails. So a changed manifest needs a
restart, not a new image. `MARKETPLACE_DIR` points it at a folder instead (development, tests).
Community sandboxes get no token, so a private repo's stdio abilities run only on a sandbox started with it.

## Checking the content

Kaja's `bun test:marketplace` loads a `../marketplace` checkout (or `KAJA_MARKETPLACE_DIR`) the way the hosts
do: every skill, HTTP tool, MCP server and code tool must load, stdio packages must be pinned, and the
sandbox image must run what it should. Kaja's CI clones the public repo there first, so a schema change that
breaks the marketplace fails in Kaja. The marketplace repo's own CI runs the same tests against a Kaja
checkout, and `tombi lint` against the JSON Schemas Kaja publishes in `config/schemas`.

## Loading

Whichever front door a turn comes in through, [`@kaja/nasi`](/development/nasi#abilities) does the loading.
It asks an `AbilityStore` (the folder on disk, or the Postgres copy) for every ability, and turns them into
tools, skills in the system prompt, and roster personas; the active persona's `abilities` list picks which a
turn uses. The prompt's skill and persona sections
are rebuilt every turn, which is why a change applies from the next message.

| Front door | Where the personas come from |
| --- | --- |
| terminal (local), local Telegram bot | the `~/.config/kaja/marketplace/` folder |
| terminal (cloud), cloud Telegram bot | the `ability` table (with the user's keys) |
| widget | the `ability` table, skills only, starting from the key's persona |

---

Next:

[Deployment](/development/deployment){: .btn .btn-green .fs-5 }
