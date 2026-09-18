# Draft response for issue #37 — not posted

Local draft. Review, edit and post it yourself; delete this file afterwards.

---

Thanks for the prompt. The maintainer team has read the [policies for community
extension projects](https://github.com/crossplane/crossplane/blob/main/GOVERNANCE.md#policies-for-community-extension-projects).
Self-assessment against each one below. We are compliant on four of six, with
one real gap and one small one, both with remediation plans.

**CNCF policies** — ⚠️ We abide by CNCF policies and the project is Apache 2.0
licensed, but our `CODE_OF_CONDUCT.md` does not actually state a code of
conduct. It is two lines of template residue carried over from upjet and refers
to the wrong project:

```
## Code of Conduct

Upjet is under [the Apache 2.0 license](LICENSE) with [notice](NOTICE).
```

*Remediation:* replace it with a pointer to the [CNCF Code of
Conduct](https://github.com/cncf/foundation/blob/main/code-of-conduct.md),
matching crossplane/crossplane. Small PR, we will raise it shortly.

**Project health** — ✅ The project is actively maintained. Five releases in the
last thirteen months, most recently
[v1.3.0](https://github.com/crossplane-contrib/provider-upjet-alibabacloud/releases/tag/v1.3.0)
in June 2026; six distinct commit authors in the past year, drawn from both
Upbound and Alibaba Cloud; commits landing continuously, the most recent today.
There are currently 10 open issues and 16 open PRs, the latter inflated by an
in-flight series of our own; we will keep that queue moving rather than let it
accumulate.

**Registry** — ✅ Packages are published to `ghcr.io/crossplane-contrib` and are
publicly pullable. The family base and all 21 subpackages carry tags through
`v1.3.0`:

```
provider-family-alibabacloud      v1.0.0, v1.1.0, v1.2.0, v1.3.0
provider-alibabacloud-vpc         v1.0.0, v1.1.0, v1.2.0, v1.3.0
provider-alibabacloud-cr          v1.3.0            (group added later)
provider-alibabacloud-alikafka    none              (merged after v1.3.0)
```

Publishing runs through
`.github/workflows/publish-provider-packages.yaml`, which delegates to the
shared `crossplane-contrib/provider-workflows` family publish workflow using
`GHCR_PAT`. We also publish to `xpkg.upbound.io/crossplane-contrib` as an
additional registry, which the policy permits. The monolithic package is
deliberately not published at present.

One thing we intend to improve, though it is not a compliance gap: that workflow
is `workflow_dispatch` only, so releases depend on someone triggering it with the
right subpackage list. That is why the recently added `alikafka` group has no
package yet. We plan to drive it from the release tag instead.

**Maintainer list** — ✅ `OWNERS.md` is current with six maintainers, and all
six (`jastang`, `humoflife`, `stevendborrelli`, `xiahuai`, `shanye997`,
`xiaozhu36`) appear in the crossplane
[`.project` roster](https://github.com/crossplane/.project/blob/main/maintainers.yaml).
One inconsistency we will tidy up: `CODEOWNERS` lists only three of the six, so
automatic review assignment does not reach the Alibaba Cloud maintainers. That
is not a policy requirement but it is worth correcting.

**Project list** — ✅ The project is listed on the [Community Extension Project
list](https://docs.crossplane.io/latest/learn/community-extension-projects/).

**API group** — ✅ Resources use `*.alibabacloud.crossplane.io`. Namespaced
resources being added for Crossplane v2 compatibility will use
`*.alibabacloud.m.crossplane.io`, following the upjet convention.

## Summary

| Policy | Status |
|---|---|
| CNCF policies | ⚠️ Code of conduct file needs replacing |
| Project health | ✅ |
| Registry | ✅ |
| Maintainer list | ✅ |
| Project list | ✅ |
| API group | ✅ |

The only outstanding item is the code of conduct file; a PR for it, and a related
tidy-up aligning `CODEOWNERS` with `OWNERS.md`, are on the way. We are happy to
close this issue once the code of conduct change has merged.
