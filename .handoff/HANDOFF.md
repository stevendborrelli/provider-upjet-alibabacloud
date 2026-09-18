# Handoff — provider-upjet-alibabacloud

Written 2026-09-18 for the agent continuing this work on another machine.
Repo: `crossplane-contrib/provider-upjet-alibabacloud`, fork
`stevendborrelli/provider-upjet-alibabacloud` (remote `origin`; upstream is
`upstream`).

There are three parallel workstreams. Read "Hard-won findings" before touching
anything — several of these cost real time to discover and are easy to trip over
again.

---

## 1. Open PRs

### Against `main` (the 1.x line) — all green, awaiting review

| PR | Branch | What |
|---|---|---|
| [#98](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/98) | `lint-followup` | golangci-lint comment correction + `GO_REQUIRED_VERSION` derived from go.mod |
| [#99](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/99) | `add-nlb` | `nlb` group, 1 resource |
| [#105](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/105) | `fix-codeowners` | CODEOWNERS ↔ OWNERS.md |
| [#108](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/108) | `add-sslcert` | `sslcertificatesservice` + ALB additional-cert attachment, 2 resources |
| [#109](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/109) | `add-sls` | `sls` group, 6 resources |
| [#110](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/110) | `add-rds` | `rds` group, 17 resources + 12 alias fixes |

All four resource PRs were 9/9 green at handoff. They all touch
`config/external_name.go`, `config/provider.go`, `config/common/short_group.go`
and the `Makefile` — additions in different places, so they should auto-merge,
but whichever lands last may need a rebase.

Already merged: #97 (golangci-lint pin), #101 (tooling modernization), #75
(alpine, by the maintainer).

### Against `release-2.0` (the Crossplane v2 / no-fork line) — failing, needs work

| PR | Branch | What | Failing |
|---|---|---|---|
| [#92](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/92) | `nofork-v2-beta4` | no-fork architecture on alicloud v2.0.0-beta4 | DCO, check-diff, report-breaking-changes |
| [#95](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/95) | `upjet-v2` | upjet v2 + crossplane-runtime v2 | DCO, check-diff |
| [#96](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/pull/96) | `namespaced-v2` | namespaced MRs for Crossplane v2 | DCO, check-diff |

These are a **stack**: `namespaced-v2` contains `upjet-v2` contains
`nofork-v2-beta4`. All three target `release-2.0`, so each diff is cumulative
until the ones below it merge. #96 shows ~3,400 files for that reason.

**`check-diff` failing on all three has not been investigated.** It is the next
thing to look at for this workstream. Likely causes: generated against a `main`
that has since moved (#101 changed the build submodule and the packaging CLI),
or genuine regeneration drift. Do not assume — read the job log.

### Parked

`ci-publish-ghcr` (pushed, no PR). Contains two genuine Makefile fixes plus
ci.yml changes that were **not** agreed. See §5.

---

## 2. Local state that does NOT transfer

- **`bundle/ali.bundle`** (3.7 MB, untracked) — the source material for the
  resource PRs. Copy it across if more groups are needed. It is an internal repo
  export: **new commits only, never cherry-pick, never add it as a remote.**
- **`issue37-self-assessment.md`**, **`pr92-replies.md`**, **`upstream-issue.md`**
  — untracked drafts. See §4 and §6.
- Scratch clone of the bundle lived at
  `$SCRATCH/ali` (bare). Recreate with
  `git clone --bare bundle/ali.bundle <dir>`.

---

## 3. Hard-won findings — read before working

**`pull-docs` silently reuses a stale docs clone.** It clones only when the
directory is *absent*:
```make
@if [ ! -d "$(WORK_DIR)/$(TERRAFORM_PROVIDER_SOURCE)" ]; then ... --branch "v$(TERRAFORM_PROVIDER_VERSION)"
```
Switching `TERRAFORM_PROVIDER_VERSION` in an existing checkout regenerates
`provider-metadata.yaml` from the **previous** version's docs. This produced a
bogus 74-file diff (`+3046/-374` in metadata plus API/CRD churn across eight
unrelated groups) that would have been committed unnoticed. **Always
`rm -rf .work/aliyun .work/terraform` when changing provider versions**, and
always check `git diff --stat` by directory before committing after a regen.

**`main` does not regenerate cleanly.** On a pristine worktree, `make generate`
modifies four committed files:
```
apis/oss/v1alpha1/zz_accountpublicaccessblock_types.go
apis/quotas/v1alpha1/zz_templateservice_types.go
package/crds/oss.…_accountpublicaccessblocks.yaml
package/crds/quotas.…_templateservices.yaml
```
A docstring renders as `` `` `` locally where the committed file has `“`. CI's
check-diff passes on `main`, so CI reproduces the committed form and local
environments do not; the differing input was never identified. **Revert these
four files after every regen** to keep diffs scoped:
```sh
git checkout upstream/main -- apis/oss apis/quotas \
  package/crds/oss.alibabacloud.crossplane.io_accountpublicaccessblocks.yaml \
  package/crds/quotas.alibabacloud.crossplane.io_templateservices.yaml
```
Worth a separate issue.

**`ugrep` is the default `grep` here and is unreliable** with `-o` and character
classes. It silently returned nothing for patterns that clearly match (alpine
version extraction, resource-name extraction). Several analyses were wrong until
redone. **Use Python for any non-trivial extraction.**

**ghcr.io anonymous probing rate-limits, and failure looks like absence.** A
sweep of 21 packages reported "all missing"; they all existed. The token endpoint
starts returning empty tokens, which reads as 403/0 tags. **Pace probes ~3 s
apart** and re-check anything that reports zero.

**Bundle configs fail lint on import grouping.** Every `config/<group>/config.go`
from the bundle needs the local import separated:
```go
import (
	"github.com/crossplane/upjet/pkg/config"

	"github.com/crossplane-contrib/provider-alibabacloud/config/common"
)
```

**`make lint` needs Go 1.26.x.** golangci-lint 2.12.2 embeds a go1.26.2
type-checker and cannot parse the Go 1.27 stdlib. Use
`GOTOOLCHAIN=go1.26.4 make lint`.

**`$2` in a Makefile `$(shell awk …)` is eaten by make.** `awk '{print $2}'`
becomes `awk '{print }'` and prints the whole line. Use `$$2`. Note
`provider-upjet-aws` and `-gcp` both carry this bug in `GO_REQUIRED_VERSION`;
#98 fixes it here.

**Multi-word make variables must be quoted when passed to a sub-make.**
Unquoted, the second word becomes a make *goal*:
`make[6]: No rule to make target 'xpkg.upbound.io/crossplane-contrib'`.

---

## 4. Governance — issue #37 compliance

[Issue #37](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/issues/37)
asks for a self-assessment against the
[community extension project policies](https://github.com/crossplane/crossplane/blob/main/GOVERNANCE.md#policies-for-community-extension-projects).

Assessment (verified, not assumed):

| Policy | Status |
|---|---|
| CNCF policies | ⚠️ `CODE_OF_CONDUCT.md` is 2 lines of upjet template residue stating a *licence*, naming the wrong project |
| Project health | ✅ 5 releases in 13 months, 6 authors in the past year |
| Registry | ✅ packages are on `ghcr.io/crossplane-contrib` through v1.3.0 |
| Maintainer list | ✅ OWNERS.md current; all 6 maintainers in `crossplane/.project` roster |
| Project list | ✅ listed in the docs |
| API group | ✅ `*.alibabacloud.crossplane.io` |

A draft response is in `issue37-self-assessment.md` (**not posted** — it speaks
for the maintainer team). **Outstanding:** the one-line code-of-conduct PR
pointing at the CNCF CoC, matching `crossplane/crossplane`.

Note the registry finding was initially assessed as non-compliant and **that was
wrong** — publishing happens via
`.github/workflows/publish-provider-packages.yaml`, which delegates to
`crossplane-contrib/provider-workflows` with `GHCR_PAT`, not via `ci.yml` or the
Makefile's `XPKG_REG_ORGS`.

---

## 5. Publishing — deliberately deferred

The maintainer asked to skip this. Findings, so it need not be re-derived:

- `ci.yml`'s `publish-artifacts` job **never publishes**. It runs
  `make -j2 build.all` and uploads GitHub artifacts. The "Login to Upbound" step
  authenticates a push that does not happen. `provider-nop` has
  `run: make publish BRANCH_NAME=…`; this repo has nothing.
- Releases only happen when someone manually dispatches
  `publish-provider-packages.yaml` with the right subpackage list. **This is why
  `alikafka` (merged in #84) has no published package** — and it will silently
  repeat for `nlb`, `sls`, `rds` and `sslcertificatesservice`. Worth an issue.
- Copying `provider-nop`'s multi-registry `XPKG_REG_ORGS` **does not work here**:
  `XPKGS` is empty because this is a *family* provider, so packages come from
  `crossplane xpkg batch` via `--family-package-url-format`, which takes a single
  registry.

Branch `ci-publish-ghcr` holds the two correctness fixes (a dedicated
`XPKG_FAMILY_REG_ORG`, and quoting the recursive make pass) plus the un-agreed
ci.yml changes. The Makefile fixes are worth keeping regardless.

---

## 6. Other outstanding items

**DCO on the release-2.0 stack.** None of those commits are signed off. Verified
commands (the merge commit needs `--rebase-merges`; the DCO app skips merge
commits):
```sh
git checkout nofork-v2-beta4 && git rebase --rebase-merges --signoff upstream/release-2.0 && git push --force-with-lease origin nofork-v2-beta4
git checkout upjet-v2        && git rebase --signoff nofork-v2-beta4 && git push --force-with-lease origin upjet-v2
git checkout namespaced-v2   && git rebase --signoff upjet-v2        && git push --force-with-lease origin namespaced-v2
```
Rebase onto the *branch below*, not `upstream/release-2.0`, or the SHAs diverge
between branches. Not run yet because it rewrites history under review.

**Upstream issue draft** in `upstream-issue.md`, unfiled. Asks
`aliyun/terraform-provider-alicloud` for a `NewProvider(sps ...conns.ServicePackage)`
constructor so per-group binaries can link only their own resources — measured
40 MB vs 298 MB. Worth filing while their v2 API is in beta.

**PR #92 review replies** in `pr92-replies.md` — already posted, kept for
reference.

**2.0 forward-port of the new groups.** All four use
`delete(r.TerraformResource.Schema, …)`; `sslcertificatesservice` also sets
`Schema["cert"].Sensitive = true`. Safe on 1.x (throwaway map), but on
`release-2.0` these mutate the live schema the CRUD functions execute against.
For nlb/sls/sslcert the fix is the existing `generationOnlySchemaOmissions` table
on that branch. **RDS is different and not mechanical:** its 12 deletions prevent
a *runtime* `ConflictsWith` failure on `Computed` fields that Terraform populates
regardless of the CR, so a generation-only fix may not suppress it. Needs a real
reconcile against a live RDS instance.

**Unvalidated on `release-2.0`:** family packaging with the namespaced example
layout (`examples/{cluster,namespaced}/`) and two CRD groups per subpackage.
`aws` and `gcp` both moved to `crossplane xpkg batch`; that stack still assumes
`up xpkg batch`. Needs a real `make build`.

---

## 7. Conventions used

- Commit messages: plain prose, explain *why*, quote evidence (command output,
  error text). Always `--signoff`.
- PR bodies: state what was verified and how; call out anything not verified;
  flag consequences for the other workstream.
- Never commit `generator` (a stray build artifact), `bundle/`, or the draft
  `.md` files. `/generator` is gitignored.
- Verify before claiming: `make generate` twice for idempotency, `go vet ./...`,
  `go test ./config/... ./internal/...`, `GOTOOLCHAIN=go1.26.4 make lint`.
