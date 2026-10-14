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
| Behavioral unit tests | 17 passed |
| Ruff lint and format | Passed |
| YAML parsing and composite Bash syntax | Passed |
| GitHub runner composite execution | Passed, including mapping and simple directory inputs |
| Fork GitHub dry run on all 34 source skills | Passed |
| GitHub live OAuth M2M publishing | Passed: 34 created, 354 uploads |
| GitHub live simple-directory rerun | Passed: 34 unchanged, 0 uploads, 0 removals |
| GitHub OIDC against staging | Not tested: account profile needs reauthentication |

JSON evidence is stored locally in `reports/staging-create.json`,
`reports/staging-rerun.json`, and `reports/staging-roundtrip.json`. Reports are
ignored by Git and contain no tokens. GitHub run artifacts are also saved under
`reports/github-create/` and `reports/github-rerun/`; all 34 GitHub bundle hashes
match the original local source snapshots. The retained custom skill has only test
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

## Published GitHub POC

- Action: https://github.com/sunishsheth2009/uc-skills-action
- Release: https://github.com/sunishsheth2009/uc-skills-action/releases/tag/v0.1.0
- Source fork: https://github.com/sunishsheth2009/databricks-agent-skills
- Action CI (including both quickstart modes):
  https://github.com/sunishsheth2009/uc-skills-action/actions/runs/38638946151
- Live directory-input rerun:
  https://github.com/sunishsheth2009/databricks-agent-skills/actions/runs/38639239894
- Live staging publish:
  https://github.com/sunishsheth2009/databricks-agent-skills/actions/runs/38639123780
- Fork dry run, all 34 real source folders:
  https://github.com/sunishsheth2009/databricks-agent-skills/actions/runs/38638250459

The fork contains the additional manually dispatched workflow
`.github/workflows/publish-uc-skills-poc.yml`. Inherited release and maintenance
workflows are disabled in the fork. Source skill contents and licensing remain
unchanged. The POC workflow pins the v0.1.0 action commit.

A dedicated workspace service principal named `uc-skills-github-poc` has access
only to skill publishing in `e2e_testing.sunish_uc_skills_github_poc` (plus catalog
usage and workspace access). The GitHub `staging` environment has the workspace
URL, client ID, destination schema, and an OAuth secret that expires after one day.
The secret was sent directly to GitHub's secret store and was never committed.
The principal creates and owns the GitHub-published skills. The original local
integration schema and its ownership remain unchanged.

ml-inference staging rejects traffic from public GitHub runners. Live staging
runs therefore use temporary repository-specific self-hosted runners on the
connected development machine. Each runner accepted one trusted manual POC job and
unregistered afterwards. No runner service remains installed. Repeating a live
fork run requires another connected runner and a valid publishing credential. The runner uses the machine's installed corporate CA
bundle and configured Python package mirror. These settings belong to the POC
infrastructure and do not appear in the public customer quickstart.

The staging account profile has an expired refresh token, so this POC validates
OAuth M2M rather than GitHub OIDC. OIDC support comes from SDK unified authentication
and needs its own account-level federation setup and integration validation.

This is a personal-account reusable-action POC. Official Databricks distribution
and Marketplace publication have not been performed. In-place bundle updates are
not atomic. The 34-skill run demonstrates the API path and bounded concurrency;
it is not a benchmark for repositories with thousands of large skill bundles.
