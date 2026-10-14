# POC results

Validated on October 14, 2026 using the exact publisher invoked by `action.yml`.
Local live runs authenticated with the explicitly selected `ml-inference` CLI
profile. No Databricks compute was started.

## Source and staging assets

- Source: `databricks/databricks-agent-skills`, `skills/`, commit
  `e7a6f218af2fc25b4b896d1e8a0cb4858b20cb75`.
- Source bundle size: 34 skills, 354 files, 3,246,308 bytes.
- Workspace: ml-inference staging.
- Isolated schema: `e2e_testing.sunish_uc_skills_action_poc`, in a catalog owned by
  the requesting user. Creation under `main` was denied, so existing grants were
  left intact and the isolated schema was created in the owned catalog instead.
- Additional integration-test skill: `publisher_roundtrip` in the same schema.
  Staging assets are retained for inspection; no existing skill was deleted.

## Results

| Check | Result |
|---|---|
| Validate the 34 original skill folders | Passed |
| Create, upload and finalize all original skills | 34 created, 354 uploads, 0 failures |
| Rerun against identical checkout | 34 unchanged, 0 uploads, 0 removals |
| Map `.claude/skills/custom-folder` to a differently named UC skill | Passed |
| Retain frontmatter name `roundtrip-guide` with target `publisher_roundtrip` | Passed |
| Upload/download nested files and all 256 binary byte values | Passed |
| Reject stale files when pruning is disabled | Passed, before writes |
| Change SKILL.md, add a reference and remove an obsolete reference | Passed: 2 uploads, 1 removal |
| Preserve effective owner and grants during update | Passed |
| Rerun after the update | Unchanged |
| Behavioral unit tests | 13 passed |
| Ruff lint and format | Passed |
| YAML parsing and composite Bash syntax | Passed |
| GitHub runner composite execution | Pending repository publication |
| GitHub OIDC against staging | Pending federation setup and GitHub runner test |

JSON evidence is stored locally in `reports/staging-create.json`,
`reports/staging-rerun.json`, and `reports/staging-roundtrip.json`. Reports are
ignored by Git and contain no tokens. The retained custom skill has only test
instructions, a small binary asset, and a reference file.

## Reproduce locally

With the skills repository checked out alongside this action repository, put this
mapping in the source repo as `skills-poc.yaml`:

```yaml
version: 1
directories:
  - path: skills
    schema: e2e_testing.sunish_uc_skills_action_poc
```

Run from this action repository:

```bash
.venv/bin/python publish.py --source-directory ../uc-skills-poc-source \
  --mapping-file skills-poc.yaml --profile ml-inference --prune-files \
  --report reports/staging-publish.json
.venv/bin/python tests/live_roundtrip.py --profile ml-inference \
  --schema e2e_testing.sunish_uc_skills_action_poc \
  --report reports/staging-roundtrip.json
```

The roundtrip check creates or updates only its named POC skill. It asserts
byte-for-byte convergence and ownership/grant preservation. It never deletes a
whole skill.

## GitHub POC handoff

A local clone is prepared at `../uc-skills-poc-source` with the additional workflow
`.github/workflows/publish-uc-skills-poc.yml`. The managed GitHub account could
read the public source but GitHub rejected its fork request with HTTP 403 because
Enterprise Managed Users cannot fork this content. Personal-account approval is
pending; no fallback identity was used and neither POC repository was published.

After the GitHub identity is selected:

1. Fork `databricks/databricks-agent-skills` and publish the separate action repo.
2. Replace `OWNER/uc-skills-action@v0.1.0` in the fork's new POC workflow with the
   reviewed action commit SHA. Push only the added POC workflow to the fork.
3. Run the action repository CI and the fork POC workflow with `dry_run: true`.
4. Provision a dedicated staging service principal and federation policy scoped
   to this fork's `staging` GitHub environment. Give it workspace access and the
   required catalog/schema and skill grants. If testing updates to the user-owned
   POC skills, explicitly grant read/write privileges to that principal, or use a
   separate service-principal-owned POC schema.
5. Set the `staging` environment variables `DATABRICKS_HOST`,
   `DATABRICKS_CLIENT_ID`, and `UC_SKILLS_SCHEMA` and verify the exact GitHub OIDC
   subject/audience for the new fork. The publisher uses SDK unified authentication;
   validate the staging discovery/token endpoint rather than assuming production
   defaults. Run the POC workflow with `dry_run: false`, then rerun unchanged.
6. Keep source skill licensing intact in the fork. The separate action contains
   newly authored publisher code and fixture content under its MIT license.

This is ready for review as a reusable-action POC. Official Databricks distribution
and Marketplace publication have not been performed. In-place bundle updates are
not atomic. The 34-skill run demonstrates the API path and bounded concurrency;
it is not a benchmark for repositories with thousands of large skill bundles.
