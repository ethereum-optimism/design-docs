# Fork PR CI Access: Design Doc

|                    |                                                    |
| ------------------ | -------------------------------------------------- |
| Author             | Adrian Sutton                                      |
| Created at         | 2026-04-24                                         |
| Initial Reviewers  | _TBD_                                              |
| Need Approval From | _TBD_                                              |
| Status             | Draft                                              |

## Purpose

External contributors to `ethereum-optimism/optimism` get no automatic CircleCI feedback on their PRs. They learn whether their change compiles, lints, or passes tests only after a member of the `engineering` team authorizes the PR via the `bailiff` bot, or if they belong to a small number of allowlisted partner orgs.

This is a poor contributor experience and a bottleneck on a small group of maintainers. Every fork PR should automatically get the broadest practical subset of CircleCI feedback, without exposing any secret or privileged resource to fork-authored code.

## Summary

Enable automatic CircleCI on all fork PRs by relying on CircleCI's platform-level secret isolation, not on trust in the PR author. Two separable pieces:

1. **Serialize `mise install`** into one upstream job so GitHub API rate-limit pressure does not scale with pipeline fanout. Independently valuable for trusted CI; prerequisite for fork CI to be reliable.
2. **Enable fork PRs to run the full `main` workflow** with per-job `when:` gates hiding jobs that are meaningless without secrets, and with context expression restrictions as defence in depth.

Bailiff is retained as the escape hatch for running the full secret-requiring suite before merge. The merge gate continues to require the bailiff-authorized run.

## Problem Statement + Context

### Current behaviour

- **CircleCI project settings.** "Build forked pull requests" is **OFF**. Per the [CircleCI OSS docs][circleci-oss], "By default, CircleCI does not build PRs from forked repositories." CircleCI receives the fork PR webhook, ignores it, and no pipeline is triggered. "Pass secrets to builds from forked pull requests" is also OFF but moot today because no fork pipelines exist.
- **Bailiff's mechanism.** Bailiff is a GitHub bot, not a CircleCI authorization tool. On `/ci authorize <full-commit-hash>` from an `engineering`-team member, it pushes the fork's exact commit to `external-fork/<sha256>` on the main repo. CircleCI sees a normal internal branch push and runs the full workflow with all contexts injected.
- **Partner-org allowlist.** A small number of orgs (e.g. `defi-wonderland`) are allowlisted in bailiff and auto-authorized on open. This is a per-org trust decision we do not want to deepen; automatic fork CI should make the allowlist an optimization, not a necessity.
- **Runner constraint.** `ops/book/src/ci/pr-authorization.md` cites self-hosted runners as a reason forks cannot run jobs. This is out of date — all executors are cloud (standard `resource_class` values only).

### What the CircleCI platform enforces once fork builds are enabled

Fork PR configs come from the fork's HEAD commit — attacker-controlled on both setup and continuation configs. The security model therefore depends entirely on platform-level guarantees, not on anything in `.circleci/`.

The master switch is **"Pass secrets to builds from forked pull requests"**, which must stay OFF. Per the [CircleCI OSS docs][circleci-oss], with it disabled CircleCI "hides four types of configuration data" from fork builds:

> Environment variables set through the application. Deployment keys and user keys. Passphraseless private SSH keys you have added to CircleCI to access arbitrary hosts during a build. AWS permissions and configuration files.

Per the [CircleCI OIDC docs][circleci-oidc]:

> OIDC tokens will only be generated for forked builds if the Pass secrets to builds from forked pull requests setting is enabled.

So a single setting gates context env vars, project env vars, SSH/deploy keys, AWS keys, OIDC tokens, and cache sharing.

| Boundary                                           | Enforced by                                                                           | Fork-bypassable? |
| -------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------------- |
| Context env var injection                          | "Pass secrets" = OFF [[1]][circleci-oss]                                              | No               |
| Project-level environment variables                | Same setting [[1]][circleci-oss]                                                      | No               |
| Deploy keys, SSH keys, AWS credentials             | Same setting [[1]][circleci-oss]                                                      | No               |
| OIDC token issuance to fork pipelines              | Same setting [[2]][circleci-oidc]                                                     | No               |
| Per-context branch/group gating                    | Restricted-context expressions on `pipeline.git.branch` [[3]][circleci-contexts]      | No (fails closed) |
| Cache writes to the main repo's namespace          | Automatic: cache keyed on fork GitHub repository-id [[1]][circleci-oss]               | No               |
| Org-level config policy (OPA/Rego)                 | [Config Policies][circleci-policies] (Scale plan), stored in org settings             | No               |
| Per-job `when:` gates in the config                | Config file (fork-controlled)                                                         | **Yes**          |
| `save_cache` key strings                           | Config file                                                                           | Yes (writes land in fork's own namespace) |

The [cache isolation guarantee][circleci-oss]:

> PRs from the same fork repo will share a cache... Two PRs in different fork repos will have different caches. That means that a PR from a fork will not share a cache with the main repo main branch. [E]nabling the passing of secrets to build from forked pull requests will enable cache sharing between the original repo and all forked builds.

Fork jobs therefore cannot poison caches that trusted jobs later read, as long as "Pass secrets" stays off.

[Restricted contexts][circleci-contexts] are stronger still — per-context gates evaluated server-side at trigger time:

> Any errors evaluating an expression will fail closed and prevent use of the context... If a variable does not exist then the expression is immediately considered to have evaluated as `false`.

Example from the docs: `pipeline.git.branch == "main"`.

Everything the fork can modify in the config either (a) does nothing useful because no secrets inject, or (b) only affects the fork's own cache namespace.

### Why GitHub Actions is not an option

GHA is deliberately restricted to a small set of security-reviewed workflows. Expanding its surface is out of scope.

### The rate-limit problem

`mise install` runs at the start of most jobs. Each run bursts GitHub API calls to list releases and download binaries. The current config fans out many jobs in parallel that race to populate the cache on a cold branch, each consuming the shared 60 req/hr anonymous budget.

On trusted branches a warm cache hides this. Fork PRs live in their own cache namespace ([cache isolation][circleci-oss]), so they start cold relative to the main repo and the first pipeline per fork would blow the anonymous budget.

## Proposed Solution

Two sequenced phases, each landable independently.

### Phase 1 — Serialize `mise install`

Introduce one `install-mise` job at the top of the main workflow. It restores cache, installs only on miss, saves cache. Every other job gains `requires: [install-mise]` and replaces its local `mise install` with a restore of the same cache.

One install invocation per pipeline regardless of fanout. Cold cache: ≈30 API calls in a single burst, inside budget. Warm cache: a fast restore.

Repo-wide but mechanical. Independently valuable:
- Faster trusted CI (no N-way parallel install contention).
- Lower ambient GitHub API usage from the org.
- Single point of change for a future self-hosted mirror (see alternatives).

### Phase 2 — Enable fork PR CI

Builds on Phase 1. Order matters: the project-setting flip (2d) is what exposes fork pipelines; everything before it makes that flip safe.

**2a. Harden sensitive contexts with expression restrictions.** For every context holding a real secret — `oplabs-gcr-release`, `oplabs-network-optimism-io-bucket`, `runtimeverification`, `devin-api`, `slack`, `discord`, `circleci-api-token`, `circleci-repo-optimism` — add an [expression restriction][circleci-contexts] on `pipeline.git.branch` limiting injection to trusted branches (`main`, `develop`, release tags, `external-fork/*`). Defence in depth: even if "Pass secrets" were ever toggled on by mistake, a fork pipeline's branch would not match and the context would fail closed.

**2b. OIDC trust audit (defence in depth).** With "Pass secrets" OFF, fork pipelines receive no OIDC token and cannot exchange one for cloud credentials [[2]][circleci-oidc]. Not a hard prerequisite, but CircleCI's guidance is explicit: "you must check the `oidc.circleci.com/vcs-origin` claims in your policies to avoid forked builds having access to resources that they should not." Apply this to every IAM trust accepting this project's CircleCI OIDC tokens (at minimum the GCP trusts behind `oplabs-gcr-release` and `oplabs-network-optimism-io-bucket`) so a future accidental flip of "Pass secrets" is not an instant credential exposure.

**2c. `is_fork_pr` pipeline parameter and per-job gates.** Setup config detects fork pipelines (via `CIRCLE_PR_USERNAME` and PR head repo URL) and passes `is_fork_pr: true|false` to every continuation config. Jobs that are pointless without secrets are gated `when: not is_fork_pr`:
- `go-release-op-deployer`, `go-release-op-up` (GCR push)
- `publish-cannon-prestates` (GCS push)
- `kontrol-tests` (Runtime Verification compute token)
- `ai-contracts-test` (Devin API)
- `close-issue`, `stale-check` (GitHub write)
- `generate-flaky-report`, `ci-gate` (CircleCI API token)
- Slack / Discord notification jobs

These gates are UX hygiene, not security: a malicious fork can remove them, but the jobs then fail opaquely because contexts aren't injected (and after 2a, fail closed on attempt). The purpose is a clean, legible check list for contributors.

**2d. Flip CircleCI project settings.** Turn "Build forked pull requests" **ON**. Confirm "Pass secrets to builds from forked pull requests" is **OFF** and documented as permanently off — it's load-bearing for secret hiding, OIDC non-issuance, and cache isolation. Watch the first real fork pipelines; be prepared to flip back if anything surfaces.

### Out of scope but enabled

- Replacing bailiff with an automatic trust-expansion model.
- Removing partner-org allowlists from bailiff (kept as an optimization).
- Self-hosted mirrors of mise/foundry downloads (see alternatives).

### Resource Usage

- Phase 1 adds one job-hop (~30–60s on cache hit) to every pipeline. Cache-miss pipelines get faster because today's parallel installs contend.
- Phase 2 increases CircleCI credit consumption proportional to fork PR volume — modest at current rates.
- No new infrastructure.

### Impact on Developer Experience

- **External contributors**: zero CI signal today → full non-secret-dependent suite automatically.
- **Maintainers**: fewer bailiff authorizations for routine compile-and-lint feedback. Bailiff still required pre-merge for secret-gated jobs.
- **Trusted contributors**: one extra pipeline-start hop (Phase 1), offset by removal of install contention.

## Alternatives Considered

| Alternative | Why not chosen |
| --- | --- |
| **Duplicate `main` into a fork-specific workflow.** | `main.yml` is ~3,225 lines; two parallel workflows guarantee drift and review fatigue. |
| **Parameterize context attachment per job (`trusted: bool`).** | Touches every job definition; makes "what runs on a fork" implicit; context-on-by-default is a footgun. |
| **Mirror a safe subset to GitHub Actions.** | GHA surface is intentionally restricted; expanding it conflicts with existing policy. |
| **Pre-bake a CI base Docker image with mise + tools.** | Every `mise.toml` change would require rebuilding and publishing the image first. Too painful for day-to-day tool updates. |
| **Public mirror for mise tool downloads.** | Attractive long-term: removes GitHub as a CI dependency. Significant infra (GCS/CDN, sync job, mise config). Deferred; Phase 1 positions us to adopt it later as a single-file change in `install-mise`. |
| **Low-privilege machine-user GitHub token in plaintext in the config.** | Raises the rate limit from 60/hr anon to 5000/hr authenticated, but the token is public and abusable. Kept in reserve if post-Phase-1 rate limits still hurt. |
| **Accept rate-limit failures, rely on bailiff fallback only.** | Without Phase 1, new-fork first pipelines would be mostly red — worse signal than no CI. |
| **Remove bailiff entirely.** | Requires re-architecting every secret-gated job away from CircleCI contexts. Separate, large project. |

## Risks & Uncertainties

- **"Pass secrets" is the single load-bearing control.** Secret hiding, OIDC non-issuance, and cache isolation all hinge on it. Must be documented as permanently off; any toggle should require security sign-off. Step 2a (context expression restrictions) and 2b (OIDC trust conditioning) limit the blast radius if it ever flips.
- **Mise rate limits on a new fork's first pipeline.** Phase 1 caps each pipeline at ~30 API calls. Multiple concurrent new-fork pipelines could still hit the shared anonymous limit. The machine token and mirror options remain available.
- **Config gates are not security boundaries.** `when: not is_fork_pr` is intent, not enforcement. Every security argument must close through platform settings. To be called out clearly in code review and in `pr-authorization.md` so future edits don't accumulate a false sense of safety.
- **`mise install` now on the critical path.** A failing `install-mise` blocks the whole pipeline; today a single job's mise failure is isolated. Needs retry and clear diagnostics.
- **Fork cache cold starts.** A contributor iterating on one PR gets warm cache across pushes; different forks each pay their own cold-start tax. Acceptable, worth documenting.
- **Config Policies availability.** Scale-plan only. If available, a policy that forbids fork pipelines from referencing sensitive contexts is cheap belt-and-suspenders; if not, the other platform guarantees still stand.

## Open Questions

- Are we on the CircleCI Scale plan (required for Config Policies)?
- Final list of `when: not is_fork_pr`-gated jobs — any that should stay visible to forks even though they'll fail, for signal?
- Does any existing automation (merge queue, required status checks) need to distinguish "fork auto-CI passed" from "bailiff-authorized full CI passed"?
- Update `ops/book/src/ci/pr-authorization.md` as part of Phase 2 or a separate docs PR?

## References

[circleci-oss]: https://circleci.com/docs/guides/integration/build-open-source-projects/ "CircleCI — Building open source projects"
[circleci-oidc]: https://circleci.com/docs/openid-connect-tokens/ "CircleCI — Using OpenID Connect tokens in jobs"
[circleci-contexts]: https://circleci.com/docs/security/contexts/ "CircleCI — Contexts (security, restrictions)"
[circleci-policies]: https://circleci.com/docs/config-policies/config-policies-overview/ "CircleCI — Config Policies overview"

1. **[CircleCI — Building open source projects][circleci-oss]** — fork-PR behaviour, "Pass secrets" setting, hidden data types, cache isolation by fork repo-id.
2. **[CircleCI — OpenID Connect tokens][circleci-oidc]** — claim structure, fork-pipeline token issuance conditioned on "Pass secrets", `vcs-origin` conditioning guidance.
3. **[CircleCI — Contexts][circleci-contexts]** — security groups, project restrictions, expression restrictions, fail-closed evaluation.
4. **[CircleCI — Config Policies][circleci-policies]** — Scale plan, org-level OPA/Rego, trigger-time evaluation.
