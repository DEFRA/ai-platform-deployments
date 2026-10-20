# ai-platform-deployments

Machine-written GitOps state for the AI Platform: the ai-platform-infra release each environment runs, at `environments/{env}/platform.json`. Nobody edits it by hand.
The Portal API dispatches `submit.yml` with the whole document; the workflow validates it at the
environment's pinned `ai-platform-infra` release, commits it as the GitHub App on `op/{operation_id}`,
opens the pull request and enables auto-merge. `validate-desired-state / validate` is the required check, and a merge
applies through `apply.yml` as the environment's deploy identity. No step waits for a person.

The contract is `docs/gitops-contract.md` in `ai-platform-infra`; the workflows are in
`ai-platform-workflows`.

| Workflow       | Trigger                        | Does                                                             |
| -------------- | ------------------------------ | ---------------------------------------------------------------- |
| `submit.yml`   | Dispatch from the Portal API   | Validate, commit as the App, pull request, auto-merge; `abandon` |
| `validate.yml` | Pull request                   | `validate-desired-state / validate`                              |
| `apply.yml`    | Merge to `main`, or a dispatch | Apply with the safeguards and a run report                       |
| `prune.yml`    | Nightly                        | Delete stale `op/*` branches                                     |

## Break-glass

When the portal is down and a change cannot wait (design B `#break-glass`):

1. Branch `break-glass/{incident}`; edit the whole file in canonical form
   (`node ai-platform-infra/checks/bin/aip-checks.js canonical --file <file>`); set `previousRelease` to the release replaced.
2. Commit with the trailer `Break-glass: {incident}`; open a pull request titled with the incident and
   labelled `break-glass`.
3. `validate` runs as normal and needs two approvals from people other than the author.
4. The merge applies like any other change.
5. Close the incident only when the file matches what the portal would have written.

Retry a failed apply with a fresh dispatch of `apply.yml`, never a re-run: a re-run reapplies its
original commit and is refused. `allow_destroy` and `force_unlock` on `apply.yml` are engineer-only
break-glass inputs and are reported in the run summary.

The callers in `.github/workflows/` reference `ai-platform-workflows` at `@v1`, so every release within a
major reaches them unchanged. A new major arrives as a Dependabot pull request; the required check refuses
any file outside `environments/`, so a platform engineer merges it through the ruleset's pull-request-only
bypass after review.
