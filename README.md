# ai-platform-deployments

Machine-written GitOps state for the AI Platform: the ai-platform-infra release each environment runs, at `environments/{env}/platform.json`. Nobody edits it by hand.
The GitOps worker dispatches `submit.yml` with the whole document for the Portal API; the workflow validates it at
the environment's pinned `ai-platform-infra` release and commits it to `main` as the GitHub App (a signed commit with
`Operation-Id`, `Requested-By` and `Traceparent` trailers; no pull request). A push to `main` applies through
`apply.yml` as the environment's deploy identity, and every run sends its status to the portal as events.
No step waits for a person.

The contract is `docs/gitops-contract.md` in `ai-platform-infra`; the workflows are in
`ai-platform-workflows`.

| Workflow            | Trigger                                      | Does                                                                                         |
| ------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `submit.yml`        | Dispatch from the GitOps worker, or an admin | Admin check; validate at the pin; commit to `main` as the App; `write`, `remove` or `revert` |
| `validate.yml`      | Pull request                                 | `validate-desired-state / validate`                                                          |
| `apply.yml`         | Merge to `main`, or a dispatch               | Apply with the safeguards, a run report and events                                           |
| `reconcile-all.yml` | Nightly, or an engineer's dispatch           | Apply every environment from `main`; takes no environment and no break-glass input           |
| `pause.yml`         | Dispatch by an admin                         | Announce `AIP_PAUSED` on `{env}-events` as `paused` or `resumed`; writes nothing             |

Hand runs of `submit.yml`, `apply.yml` and `pause.yml` need the repository admin role
(`aice-platform-admins`); engineers re-run failed jobs of portal and push runs and dispatch `reconcile-all.yml`.
`.github/` is code-owned by `aice-platform-admins` (`.github/CODEOWNERS`), because a pull request runs its own
workflow files and the `reconcile-all.yml` exemption keys on that file.

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
original commit and is refused. `allow_destroy` and `force_unlock` on `apply.yml` are admin-only
break-glass inputs and are reported in the run summary.

The callers in `.github/workflows/` reference `ai-platform-workflows` at `@v2`, so every release within a
major reaches them unchanged. A new major arrives as a Dependabot pull request; it changes `.github/`, so it
merges only with a code-owner review by `aice-platform-admins` and the required check.
