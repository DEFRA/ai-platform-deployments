# ai-platform-deployments

Machine-written state: the `ai-platform-infra` release each environment runs, at `environments/{env}/platform.json`. The GitOps worker and the release workflow write it through `submit.yml` (a signed commit to `main`, no pull request); a push to `main` applies the platform. Contract: `docs/gitops-contract.md` in `ai-platform-infra`.

## Review first

- **A pin is data, not code.** One whole `platform.json`, in canonical form (keys sorted, two-space indent, LF, one trailing newline), under 60 KB, no secret and no personal data.
- **Pin rules.** `release` is a tag `vX.Y.Z` that resolves to `sha`, or a SHA only where the registry entry allows it. `previousRelease` equals the release replaced, so `git log` is a linear ledger. Pins are replaced, never deleted.
- **One environment per pull request**, and only files under `environments/`. A pull request that changes nothing under `environments/` (`.github/`, the README) passes `validate-desired-state / validate` once approved, and `.github/` needs a code-owner review by `aice-platform-admins`. There is no bypass.
- **Blast radius.** A merge applies that environment's platform: check the release notes of the new tag, and that the tenants state is not left on an older release than the platform (tenants applies only at the platform's release).
- `allow_destroy` and `force_unlock` are admin-only break-glass inputs.

## Conventions

- Nobody edits a pin by hand. Change it by dispatching `submit.yml`, or by break-glass when the portal is down.
- Break-glass: branch `break-glass/{incident}`, whole file in canonical form (`node ai-platform-infra/checks/bin/aip-checks.js canonical --file <file>`), `previousRelease` set to the release replaced, commit trailer `Break-glass: {incident}`, PR labelled `break-glass`, two approvals from people other than the author.
- To retry a failed apply, an engineer re-runs the failed jobs of the latest portal or push run, or dispatches `reconcile-all.yml`. A fresh hand dispatch of `apply.yml` and the break-glass inputs need the admin role (actor and whoever re-ran it); a re-run of a run no longer at the head of `main` is refused as `superseded`.
- Workflow callers reference `ai-platform-workflows` at `@v2`. A new major arrives as a Dependabot PR and merges only with a code-owner review by `aice-platform-admins` (`.github/` is code-owned).

## Pull requests

`submit.yml` commits to `main` as the App with no pull request, so every pull request here is human: break-glass, or a change to `.github/` or the README. The steps below apply to them.

1. Branch `<type>/<description>` (`break-glass/{incident}` for incidents); conventional commits.
2. Run `aip-checks` canonical and validate against the pinned release before opening the PR.
3. State the environment, the old and new release, and what the apply will change.
4. **Request Copilot review**: `gh pr edit <n> --add-reviewer "@copilot"`. When it lands, fix every comment or reply with the reason it does not apply, push, then resolve each thread (GraphQL `resolveReviewThread`); re-request after a substantial push and repeat until nothing is unresolved. Then request a human reviewer. Copilot is advisory and never replaces the required approvals.
5. Do not merge your own PR. No person bypasses the `main` ruleset; engineers bypass only the path push rulesets, to push `break-glass/*` branches.
