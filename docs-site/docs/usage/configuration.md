---
title: "Project Configuration (kingfisher.yaml)"
description: "Set project-wide defaults for kingfisher scan via a kingfisher.yaml file. Most CLI flags can live in config; CLI flags always win over config values."
---

# Project Configuration (`kingfisher.yaml`)

Long `kingfisher scan` invocations get awkward in CI. A project-local
`kingfisher.yaml` lets you set most flags as defaults so the actual command
stays short. The file is **additive** for list/map values and **default-only**
for scalars: a config value applies only when the user did not pass the
matching `--flag`. CLI flags always win.

## Discovery

- `--config FILE` overrides everything; an explicit path that fails to parse is fatal.
- Otherwise Kingfisher walks up from the current working directory looking for `kingfisher.yaml`. Missing config is silent.

## Precedence

```
CLI flag  >  environment variable  >  kingfisher.yaml  >  built-in default
```

For list-typed values both sources are concatenated, so passing
`--skip-word EXAMPLE` and listing `EXAMPLE` again in `kingfisher.yaml` is safe
but redundant.

## Generating a config from an existing CLI invocation

Don't write the YAML by hand. If you already have a long `kingfisher scan`
command, run the same flags under `kingfisher config init` and capture the
output:

```bash
# Print to stdout, redirect to a file
kingfisher config init \
  --confidence high \
  --redact \
  --exclude vendor/ \
  --skip-word EXAMPLE \
  --format sarif \
  --output ./kingfisher.sarif \
  --alert-min-confidence high \
  --alert-webhook https://hooks.slack.com/services/T0/B0/AAA \
  --tls-mode lax \
  > kingfisher.yaml

# Or write directly:
kingfisher config init [...flags...] --out kingfisher.yaml
# Pass --force to overwrite an existing file.
```

Only flags you actually supply appear in the output; clap defaults are
omitted to keep the file minimal. Scan-target inputs (paths, `--git-url`,
GitHub/GitLab/etc. flags, S3/GCS buckets) are stripped — they describe
*what* this run scans and don't belong in shared project policy.

## Schema (top-level sections)

| Section      | What it sets                                                        |
|--------------|---------------------------------------------------------------------|
| `scan`       | confidence, redact, only-valid, turbo, jobs, etc.                   |
| `rules`      | `--rule` ruleset selection, `--rules-path`, `--load-builtins`       |
| `validation` | timeout, retries, RPS limits (global + per-rule)                    |
| `filters`    | skip-words / skip-regex / exclude / max-file-size / archive depth   |
| `output`     | `--format`, `--output` (report destination)                         |
| `baseline`   | `--baseline-file`, `--manage-baseline`                              |
| `alerts`     | per-webhook entries + global `--alert-*` defaults                   |
| `global`     | TLS mode, internal-IP allow-list, endpoint overrides                |
| `git`        | clone dir, keep-clones, repo-clone-limit, include-contributors      |

A complete worked example, with every field annotated, lives in
[`docs/CONFIG.md`](https://github.com/mongodb/kingfisher/blob/main/docs/CONFIG.md).

## Example: a minimal CI config

```yaml
# kingfisher.yaml — checked into the repo root
scan:
  confidence: high
  redact: true
output:
  format: sarif
  path: ./kingfisher.sarif
filters:
  exclude:
    - vendor/
    - "**/node_modules/**"
    - "**/__snapshots__/**"
alerts:
  defaults:
    min_confidence: high
  webhooks:
    - url: https://hooks.slack.com/services/T0/B0/AAA
      format: slack
```

```bash
kingfisher scan .                          # auto-discovers ./kingfisher.yaml
kingfisher scan . --config /etc/kf.yaml    # explicit path
kingfisher scan . --confidence low         # CLI overrides the config value
```

## What is *not* config-overridable

Scan-target inputs are intentionally CLI-only — they describe *what* this
invocation is scanning, not project policy:

- positional paths, `--git-url`
- `--github-user` / `--github-org`, `--gitlab-user` / `--gitlab-group`, and the equivalent Gitea / Bitbucket / Azure / Hugging Face flags
- `--s3-bucket`, `--gcs-bucket`, `--docker-image`
- `--jira-url`, `--confluence-url`, `--slack-query`, `--teams-query`, `--postman-*`

Auth tokens are also intentionally not in YAML; they continue to come from
env vars (`KINGFISHER_GITHUB_TOKEN`, etc.) so secrets stay out of
checked-in config files.

## Caveats

- **`scan.jobs` and the Tokio runtime.** The Tokio runtime is sized from the CLI value of `--jobs` *before* `kingfisher.yaml` is loaded, so config-only `scan.jobs` will resize the scanner's job pool but not the underlying async worker pool. If you want both to match, pass `--jobs N` on the CLI.
- **Subcommand scope.** Project config only applies to `kingfisher scan`. `validate`, `revoke`, `access-map`, `view`, and `rules` ignore `kingfisher.yaml`; pass their flags on the CLI directly.

## Validation

`kingfisher.yaml` is rejected at startup if it has unknown fields, malformed
URLs in webhook entries, invalid regex, out-of-range numeric values, or
`endpoints` that don't follow `provider=url`. Use `--config /path/to/file.yaml`
to surface parse errors when iterating; auto-discovered configs that fail to
parse are also fatal.
