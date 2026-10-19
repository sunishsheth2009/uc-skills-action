# Suggested Databricks documentation section

This is a proposed replacement for the notebook-led section in the create-and-share
skills tutorial. The action currently lives in a personal POC repository. Before
publishing this wording in official docs, replace the POC links with the approved
Databricks-owned repository and release. The authentication link remains a one-time
administrator prerequisite.

---

## Publish skills from GitHub

Keep your skills in GitHub and publish changes to Unity Catalog automatically when
you merge them. The GitHub Action runs on a GitHub runner; you do not need a
Databricks notebook or job.

**Before you start:** ask your administrator to connect your GitHub repository to
Databricks using [workload identity federation](https://docs.databricks.com/aws/en/dev-tools/auth/provider-github)
and give the publishing service principal access to your skills schema. Your
repository's GitHub `production` environment must define the `DATABRICKS_HOST` and
`DATABRICKS_CLIENT_ID` variables. If your workspace restricts network access,
use a Linux self-hosted runner that can reach it. For the exact administrator
steps and an OAuth M2M alternative, see the [authentication guide](https://github.com/sunishsheth2009/uc-skills-action/blob/main/docs/authentication.md).

1. Copy the [publishing workflow](https://github.com/sunishsheth2009/uc-skills-action/blob/main/examples/quickstart.yml)
   into your repository as `.github/workflows/publish-skills.yml`.
2. Set the skill folder and destination schema:

   ```yaml
   - uses: sunishsheth2009/uc-skills-action@v0.1.0
     with:
       skills-directory: skills
       target-schema: acme.team_skills
       prune-files: 'true'
   ```

   Each `skills/<folder>/SKILL.md` becomes `acme.team_skills.<folder>`. The action
   includes supporting files and updates existing skills in place.
3. Open **Actions > Publish skills to Databricks > Run workflow** to publish your
   current skills. First leave **Validate folders only** checked to validate local
   content, then run again with it unchecked to publish. Validation alone does not
   test authentication or permissions. After that, merges to `main` that change
   the `skills` folder publish automatically.

For custom skill names or multiple destination schemas, use the
[folder mapping example](https://github.com/sunishsheth2009/uc-skills-action/blob/main/examples/skills.yaml).
If you change the source folder, update the workflow's `paths` filter as well.
For production workflows, pin the action to a reviewed release commit SHA.

The action preserves existing ownership and grants. Removing a source folder does
not delete the UC skill. With `prune-files: true`, files removed from a published
folder are also removed from that skill's bundle. Bundle updates are not atomic.
