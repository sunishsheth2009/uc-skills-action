# Publish Git skill folders to Unity Catalog

Add this GitHub Action to a repository to publish `SKILL.md` folders as Databricks
Unity Catalog skills whenever you merge changes. Publishing runs on the GitHub
runner using the Skills and Files APIs. No Databricks notebook, job, or compute is
required. This repository is a POC, not an official Databricks supported action.

## Start with your repository

1. Have an administrator enable UC skills in your workspace, create the target
   catalog/schema, and give a publishing service principal workspace access,
   `USE CATALOG`, `USE SCHEMA`, and `CREATE VOLUME`. For existing skills, the
   publisher needs `READ VOLUME` and `WRITE VOLUME`; creating as the same service
   principal makes it the owner. Share skills separately with `READ VOLUME` grants.
2. Configure [GitHub workload identity federation](https://docs.databricks.com/aws/en/dev-tools/auth/provider-github)
   for that service principal, your repository, and a GitHub environment such as
   `production`. Set `DATABRICKS_HOST` and `DATABRICKS_CLIENT_ID` as GitHub environment
   **variables**, not secrets. Match the federation policy to GitHub's exact OIDC
   subject; newer repos can include immutable numeric IDs. The SDK exchanges the
   OIDC token automatically. No long-lived token is needed.
3. Copy [examples/skills.yaml](examples/skills.yaml) to your repository root and
   edit the folders and targets. Copy [examples/publish-skills.yml](examples/publish-skills.yml)
   to `.github/workflows/publish-skills.yml`. Replace the action owner and pin a
   reviewed release commit SHA.
4. Run the workflow manually, or merge a change on `main`. Check the per-skill
   summary and downloadable JSON report.

## Folder mappings

```yaml
version: 1
skills:
  - path: skills/sql-guide
    target: acme.team_skills.sql-guide
  - path: .claude/skills/review
    target: acme.team_skills.code-review
```

Any folder in the checkout is supported; no marketplace manifest or agent-specific
repository structure is required. UC securable names and frontmatter bundle names
are independent. The original frontmatter is retained. Each folder must contain
valid `SKILL.md` frontmatter with a skill name and a description of at most 1024 UTF-8
bytes. Supporting files, including binary assets and nested directories, are
included. Paths must stay in the checkout, and symlinks are rejected. This POC
accepts lowercase UC identifiers containing letters, numbers, underscores, and
hyphens. Multiple folders cannot target the same skill.

To publish all immediate skill folders under one directory:

```yaml
version: 1
directories:
  - path: skills
    schema: acme.team_skills
```

This maps `skills/<folder>/SKILL.md` to `acme.team_skills.<folder>`. Non-skill child
directories are ignored. Use explicit entries for exceptions, and avoid overlapping
targets between explicit mappings and directory discovery. Zero-skill mappings fail.

To source skills from another repository, use `actions/checkout` with `repository`,
`ref`, and `path`, and pass that path as `source-directory`. Put the mapping file
inside that checkout, or copy a centrally managed mapping into it. GitHub checks
out private source repos with its own credentials; Databricks needs no Git
credential. A separate private source repo needs a scoped GitHub App token or other
read credential.

## Inputs and outputs

| Input | Default | Purpose |
|---|---|---|
| `mapping-file` | `skills.yaml` | Mapping relative to the source checkout |
| `source-directory` | `.` | Source checkout root |
| `dry-run` | `false` | Validate all bundles without authentication or API calls |
| `prune-files` | `false` | Remove stale files inside mapped skills for exact mirroring |
| `workers` | `4` | Concurrent skill publishers, from 1 to 8 |

The `report-file` output is an absolute path to a JSON report with the checked-out
source commit, bundle SHA-256s, per-skill status, and uploaded/removed file counts.
Failures set a nonzero exit code after other skills finish. Local validation errors
fail before remote writes. Reports omit API exception bodies and include error
types to avoid dumping credentials or upstream request details.

## Update behavior and limits

- Local folders are validated and snapshotted before publishing.
- Remote bytes are compared to the snapshot; unchanged files aren't uploaded and
  identical finalized bundles are skipped. This requires remote reads proportional
  to bundle size, avoids an external hash database, and detects out-of-band edits.
  Start with four workers and lower the value for large bundles or rate limits.
  The SDK handles eligible transient API retries. This POC holds all local bundles
  in memory; it has no repository-size benchmark or incremental hash index yet.
- Existing skills are updated in place, preserving ownership and grants.
- With `prune-files: false`, stale remote files cause that skill to fail before
  writes. With `true`, stale files are removed after uploads succeed. Empty remote
  directories can remain; they contain no bundle files.
- Removing a folder or mapping never deletes its UC skill. Whole-skill deletion is
  an explicit administrative operation. Renaming a target creates a new skill and
  leaves the old one in place.
- File uploads and finalization are separate operations. Updates are **not atomic**;
  consumers can observe mixed contents, and failed updates can leave partial
  changes. Retrying converges to the source bundle. `SKILL.md` is uploaded last.
  Finalization refreshes metadata and validates; it is not a transaction boundary.
- Use one writer per target and a GitHub `concurrency` group with
  `cancel-in-progress: false`. Groups do not coordinate different repositories;
  give each target one publishing repository.
- For PRs, run only `dry-run: true` without Databricks authentication. Publish from
  the trusted branch or a protected deployment environment.

## Customize or run locally

Fork this repository, or copy `action.yml`, `publish.py`, and `requirements.txt`
into `.github/actions/uc-skills` and use `uses: ./.github/actions/uc-skills` after
checkout. The Python publisher is shared by the action and local runs.

```bash
python3.12 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python publish.py --source-directory /path/to/repo --dry-run
.venv/bin/python publish.py --source-directory /path/to/repo \
  --profile your-workspace-profile --prune-files --report reports/publish.json
```

Customer workflows use SDK environment configuration, including GitHub OIDC and
OAuth M2M. Local runs can select a CLI profile with `--profile`. Don't commit tokens
or credentials. Staging OIDC setup is tested separately from local profile
authentication; local publishing success does not validate GitHub federation.

## Release for customer distribution

The repository root contains `action.yml`, so customers can consume it directly
with `uses: OWNER/uc-skills-action@<commit>`. There is no GitHub App installation.
Publishing to GitHub Marketplace is optional and requires repository-owner review
of the release and listing. For official distribution, transfer the reviewed
action into an approved Databricks-owned public repository, confirm licensing and
ownership, run staging integration tests, tag a release, and publish release notes
covering API compatibility. Keep customers on reviewed release SHAs. This POC does
not imply official support.

## Development

```bash
.venv/bin/pip install pytest ruff
.venv/bin/python -m pytest -q
.venv/bin/ruff check .
.venv/bin/ruff format --check .
```

CI runs behavioral unit tests and invokes the composite action in local-only
validation mode. Live integration results are documented in `POC.md` after testing.
