# Publish Git skill folders to Unity Catalog

Add this GitHub Action to a repository to publish `SKILL.md` folders as Databricks
Unity Catalog skills whenever you merge changes. Publishing runs on the GitHub
runner using the Skills and Files APIs. No Databricks notebook, job, or compute is
required. This repository is a POC, not an official Databricks supported action.

## Get started: one workflow file

Once your Databricks administrator has connected GitHub to a publishing service
principal, you only need one workflow file. No mapping file is needed when the
skill folders all go to the same schema.

1. Copy [examples/quickstart.yml](examples/quickstart.yml)
   ([download](https://raw.githubusercontent.com/sunishsheth2009/uc-skills-action/v0.1.0/examples/quickstart.yml)) into your repository as
   `.github/workflows/publish-skills.yml`.
2. Change `skills-directory: skills` to your skills folder and
   `target-schema: acme.team_skills` to your UC schema. If you change the folder,
   update the workflow's `paths` filter too.
3. In GitHub, open **Actions > Publish skills to Databricks > Run workflow**, or
   merge a skill change to `main`.

The action maps `skills/<folder>/SKILL.md` to `acme.team_skills.<folder>` and
publishes supporting files too. No fork, GitHub App installation, notebook, or
Databricks job is needed. Customers reference the hosted action:

```yaml
- uses: sunishsheth2009/uc-skills-action@v0.1.0
  with:
    skills-directory: skills
    target-schema: acme.team_skills
    prune-files: 'true'
```

For custom names or several destination schemas, use the mapping example below.
Pin a reviewed release commit SHA for production. The current personal-account
release is a POC; official Databricks docs should reference an approved
Databricks-owned action once it has been reviewed and transferred.

### One-time administrator setup

This setup is required for any automated publisher, including the notebook.
Skill authors should not have to repeat it.

1. Enable UC skills in the workspace and create the destination catalog/schema.
   Give the publishing service principal workspace access, `USE CATALOG`,
   `USE SCHEMA`, and `CREATE VOLUME`. Existing skills also need `READ VOLUME`
   and `WRITE VOLUME` for that publisher. New skills are owned by the publisher;
   share them separately with `READ VOLUME` grants.
2. Configure [GitHub workload identity federation](https://docs.databricks.com/aws/en/dev-tools/auth/provider-github)
   for the service principal, repository, and GitHub environment `production`.
   Match GitHub's exact OIDC subject, which can include immutable numeric IDs.
3. In the repository's **Settings > Environments > production**, add environment
   **variables** `DATABRICKS_HOST` (workspace URL) and `DATABRICKS_CLIENT_ID`
   (publishing principal's application ID). Neither value is a secret. The SDK
   automatically exchanges GitHub's short-lived OIDC token with Databricks.

If your organization already has a service principal for OAuth CI, reuse that
setup with [examples/quickstart-oauth.yml](examples/quickstart-oauth.yml). Store
`DATABRICKS_CLIENT_SECRET` as a GitHub environment **secret** and the workspace URL
and client ID as environment variables. This avoids needing a new federation
policy, while requiring secret rotation. The publishing permissions are the same.

For a workspace with private networking or IP restrictions, use a self-hosted
GitHub runner with access to the workspace. GitHub-hosted `ubuntu-latest` runners
can be used when the workspace is publicly reachable and permits those requests.

An organization can centralize the administrator setup and reusable workflows
so each skill repository only chooses its folder and destination schema.

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
| `skills-directory` | unset | Publish all immediate skill folders without a mapping file |
| `target-schema` | unset | Destination catalog.schema for skills-directory |
| `mapping-file` | unset | Custom mappings; falls back to skills.yaml if directory inputs are absent |
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
with `uses: sunishsheth2009/uc-skills-action@<commit>`. There is no GitHub App installation.
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
