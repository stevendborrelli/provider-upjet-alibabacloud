# Draft replies to review comments on PR #92

Nothing posted. Local draft — delete before merging.

---

## 1. `config/provider.go:110` — schema deletes (also the top-level review comment)

You're right, and it's worse than "might". Confirmed by building the runtime
provider and diffing its schema against a pristine `alicloud.Provider()`:

```
pristine alicloud.Provider():
  oss_bucket.acl      present=true
  oss_bucket.policy   present=true
  slb_acl.entry_list  present=true
after config.GetProvider(ctx, false):
  oss_bucket.acl      present=false
  oss_bucket.policy   present=false
  slb_acl.entry_list  present=false
Resource.TerraformResource aliases ResourcesMap entry: true
```

Your reading of upjet is exactly right, and it still holds on v2.4.2
(`pkg/config/provider.go:436`): `terraformResource = p.TerraformProvider.ResourcesMap[name]`,
no copy. All three failures you named check out against beta4:

- `resource_alicloud_oss_bucket.go:532` — `oss.ACL(oss.ACLType(d.Get("acl").(string)))`,
  unconditional. Panics on every bucket create.
- `resource_alicloud_oss_bucket.go:932` — `if err := d.Set("policy", policy); err != nil`.
  Observe fails permanently.
- `resource_alicloud_slb_acl.go:140` — same shape for `entry_list`.

On the fourth: I initially couldn't reproduce it because I looked in
`resource_alicloud_slb_load_balancer.go`. You were pointing at
`config/slb/config.go:33`, which is in the `alicloud_slb_listener`
configurator, not `alicloud_slb_load_balancer` — my misread. It does hold:
`resource_alicloud_slb_listener.go:395` and `:702` both do
`scId = d.Get("ssl_certificate_id").(string)` inside `if protocol == Https`
when `server_certificate_id` is empty, so an HTTPS listener without
`serverCertificateId` panics instead of getting the intended
"required field is not set" error.

Fixed in 7feffca. The 68 omissions move into one table in
`config/schema_omissions.go`, registered only when `GetProvider` is building
the code-generation provider. That restores the pre-no-fork behaviour exactly:
the field stays in the runtime schema, the CRD never sets it, and `d.Get`
returns the zero value rather than nil. Regenerating produces byte-identical
output, so there are no CRD changes.

Also added `TestRuntimeSchemaIsPristine`, which compares every configured
resource's runtime schema against a pristine `alicloud.Provider()` and fails on
any removed field — verified it fails when a `delete` is reintroduced, so this
can't silently come back.

---

## 2. `config/provider.go:87` — two sources of truth for the schema

Worth a correction first: `provider-upjet-aws` does the opposite of using the Go
schema for both. Its `GetProvider` explicitly rebuilds from the JSON schema when
`generationProvider` is set, with the comment "use the JSON schema to
temporarily prevent float64->int64 conversions in the CRD APIs" — so generating
from the Go schema would itself churn the CRD APIs. Our structure already
matches theirs.

What aws does on top, and we were missing, is reconcile the known divergence:
the JSON schema does not faithfully carry `MaxItems`, so it runs
`traverser.NewMaxItemsSync` over the Go schema before generating. Added in
1be4795.

Measured impact: across the whole upstream provider there is exactly one
`MaxItems` divergence that the sync corrects,
`alicloud_ros_stack_instances.deployment_options`, which is not in our include
list. So regenerating produces no change today. Keeping it anyway, since the
divergence is a property of the two schema sources rather than of the current
include list.

On the guard test — agreed, and added as `TestSchemaSourcesAgree`. It asserts
every resource in the plugin SDK include list is present in both sources, which
is precisely the condition `NewProvider` turns into a startup panic, and it is
what would catch `go.mod` and `TERRAFORM_PROVIDER_VERSION` drifting onto
different upstream commits (see comment 3). I did not assert the two schemas are
field-for-field identical, because per the above they are intentionally not.

For the record, they agree today on the things that matter: 1187 resources
each, all 189 configured names present in both.

## 3. `Makefile:22` — registry pin vs `go.mod` commit

Correct, and nothing caught it. Worth adding why it is easy to miss: the
`v2.0.0-beta4` tag declares module path
`github.com/aliyun/terraform-provider-alicloud` with no `/v2`, so `go.mod` can
only reference it as the pseudo-version `v1.290.1-0.20260827115952-d31d4a12eca0`
behind a `replace`. The two identifiers for the same upstream commit look
nothing alike.

Addressed in 47e2f79: each pin now carries a comment pointing at the other, and
Renovate is configured to leave the dependency alone. That last part matters
more than the comment — the pseudo-version sorts *below* the real `v1.292.0`, so
an automated bump would look like an upgrade while actually moving us onto the
terraform-plugin-sdk v1 code line, which cannot build under no-fork at all.

The mechanical guard is `TestSchemaSourcesAgree` from comment 2, which fails if
the two pins drift far enough apart that a configured resource is missing from
either source.

## 4. `Makefile:343` — crddiff

Confirmed, and the cause is narrower than "uptest dropped it": `crddiff` was
never part of the `crossplane/uptest` binary. I checked v0.13.1, v1.1.2 and
v2.2.0 — all three declare exactly one command, `e2e`. The tool still lives in
the older `upbound/uptest` module at `cmd/crddiff`, which is why
`CRDDIFF_VERSION = v0.12.1` was sitting there unused.

`provider-upjet-aws` invokes it directly rather than through the uptest binary:

```make
changes_detected=$$(go run github.com/upbound/uptest/cmd/crddiff@$(CRDDIFF_VERSION) revision --enable-upjet-extensions ...)
```

Adopted that in 1be4795, which makes `CRDDIFF_VERSION` meaningful again, and
bumped `UPTEST_VERSION` to v2.2.0 at the same time. Verified end to end against
this branch:

```
$ crddiff revision --enable-upjet-extensions <base> package/crds/slb...listeners.yaml
- Deleted property: instancePort
- Deleted property: lbPort
- Deleted property: lbProtocol
exit: 1
```

So the gate is live and the CRD removals will now be reported rather than
passing silently.

One thing I deliberately did not change: the loop reports without failing the
target. That is identical in the aws Makefile, so it looks like intended
"report, don't block" behaviour rather than an oversight. Happy to make it
blocking if you would rather, though that would want to land after the
removals in this PR are acknowledged.

## 5. `zz_kubernetes_terraformed.go:128` — schema version bump

Confirmed on all four, and agreed it is inert under no-fork — upjet only reads
`GetTerraformSchemaVersion` on the CLI path.

You were right that the reporting was also down, and there turned out to be four
independent reasons rather than one:

- `config/generated.lst` did not exist, so `scripts/version_diff.py` had nothing
  to iterate.
- The `ci.yml` step was gated on `inputs.upjet-based-provider`, and `ci.yml`
  declares only `push`, `pull_request` and `workflow_dispatch` — no
  `workflow_call`, so `inputs` is always empty and the step never ran.
- The target read the previous version with a `sed` matching
  `TERRAFORM_PROVIDER_VERSION :=`, but this Makefile uses `?=`, so
  `PREV_PROVIDER_VERSION` was always empty.
- `scripts/version_diff.py` was not executable, so the target failed with
  "Permission denied" even once it got that far.

All four fixed in 47e2f79. `cmd/generator` now writes `config/generated.lst` the
way `provider-upjet-aws` does. Verified against the merge base:

```
Detected previous Terraform provider version: 1.281.0
Current Terraform provider version: 2.0.0-beta4
alicloud_cr_repo:0-1
alicloud_cs_edge_kubernetes:0-1
alicloud_cs_kubernetes:0-1
alicloud_cs_managed_kubernetes:0-1
```

Exactly the four you identified. Will also call them out in the PR description
alongside the CRD removals.

## 6. `hack/setup-family-providers.sh:27`

Good catch, and my change there was treating a symptom. `BASE_IMAGE_DIR` on line
9 was `cluster/images/provider-upjet-alibabacloud`, but the only directory in the
repo is `cluster/images/provider-alibabacloud`, so under `set -e` the
`cp "$BASE_IMAGE_DIR/Makefile"` on line 27 failed first and the script exited on
the first provider. Removing the `terraformrc.hcl` copy addressed a file that
does not exist without fixing why the whole path was wrong.

Fixed in 1be4795. `provider-upjet-aws` names its image directory `provider-aws`,
so the short form is the pattern, and line 9 now points at
`cluster/images/provider-alibabacloud`. Line 20 had the same inconsistency and
now writes to `provider-alibabacloud-$provider`, consistent with
`PROJECT_NAME` in the Makefile.

The `terraformrc.hcl` copy stays removed: under no-fork there is no Terraform
CLI in the image to configure, so there is nothing for it to copy.
