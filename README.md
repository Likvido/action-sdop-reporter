# SDOP/ENGP reporter

## Description

The **SDOP/ENGP reporter** GitHub Action reports a production deployment to Visma's ENGP/SDOP Jira gateway so Likvido's deployments show up in Visma's Engineering Performance (DORA) dashboards.

On every invocation it:

1. Looks up the SHA of the last successful run of the same workflow on the same branch (falling back to the repo's initial commit).
2. Walks `git log --first-parent` between that SHA and `HEAD` to compute number of changes, total delivery lead time, and deployment time.
3. Creates a `SDOP Deployment` issue in Visma's Jira via `POST /jira_gateway/create_issue` with component `1664-Likvido`, a per-service label (`sub-<repo>-<app-name>`), and the relevant custom fields populated.
4. Transitions the issue to `Released` on success or `Failed` on any other deployment status.

On non-success deployment statuses the metric custom fields are reported as `0` so the failed-deployment count is still recorded.

## Inputs

- `jira-token`: Bearer token for the Visma Jira gateway, generated in Hubble. (**Required**)
- `app-name`: Service name, e.g. `webapp-api`. Becomes part of the per-service Jira label. (**Required**)
- `repo-name`: The GitHub repository in `owner/name` form, typically `${{ github.repository }}`. Used to namespace the per-service label. (**Required**)
- `deployment-start`: ISO 8601 UTC timestamp captured immediately before the production deploy step. (**Required**)
- `deployment-status`: Outcome of the production deploy step: `success`, `failure`, `cancelled`, or `skipped`. (**Required**)
- `exclude-from-metrics`: If `true`, adds the `ENGPexclude` label so Visma ignores the ticket. Default `true`; flip to `false` after Visma has reviewed onboarding. (**Optional**)
- `dry-run`: If `true`, log the payload but don't call the Jira gateway. (**Optional**)
- `github-token`: Token used to query the GitHub API for the previous successful run. Defaults to `${{ github.token }}`. (**Optional**)
- `jira-host`: Visma Jira gateway host. Override only for testing. Defaults to `https://prod.integration-hub.visma.com`. (**Optional**)

## Outputs

- `issue-key`: Key of the created Jira issue (empty on dry-run).
- `from-sha`: The SHA used as the start of the lead-time range.

## Usage

This action is intended to be invoked by `likvido/action-deployment-pipeline` on the production branch. If you need to call it directly:

```yaml
- name: Record deployment start
  id: sdop_start
  shell: bash
  run: echo "ts=$(date -u +%FT%TZ)" >> "$GITHUB_OUTPUT"

- name: Build & deploy to production
  id: prod_deploy
  uses: likvido/action-release@v3.12
  with:
    ...

- name: Report to Visma ENGP/SDOP
  if: always()
  uses: likvido/action-sdop-reporter@v1
  with:
    jira-token: ${{ secrets.JIRA_SDOP_TOKEN }}
    app-name: webapp-api
    repo-name: ${{ github.repository }}
    deployment-start: ${{ steps.sdop_start.outputs.ts }}
    deployment-status: ${{ steps.prod_deploy.outcome }}
    exclude-from-metrics: 'true'   # flip to 'false' after Visma reviews onboarding
```

## Important notes

- **Production only**: ENGP only wants production data. Gate the calling step on the production branch.
- **`if: always()`**: invoke even on deploy failure so failed deployments are also reported (transition `Failed` instead of `Released`).
- **Unshallow checkout**: this action calls `git fetch --unshallow` itself, so calling workflows don't need `fetch-depth: 0`.
- **Token scope**: `jira-token` is a Visma Jira PAT generated in Hubble and stored as an org-level GitHub secret (`JIRA_SDOP_TOKEN`) scoped to the relevant repositories.

## Releasing new version

Either create the new release + new version tag directly in the GitHub UI, or create it like this:

1. Commit and push your changes
2. Create a new tag: `git tag -a -m "Description of this release" <version>`
3. Push the tag: `git push --follow-tags`

### Example

```
git tag -a -m "First version" v1
git push --follow-tags
```
