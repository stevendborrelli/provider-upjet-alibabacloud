# Upstream issue draft — aliyun/terraform-provider-alicloud

Target: https://github.com/aliyun/terraform-provider-alicloud/issues (v2 / SDK v2 branch)

Local draft, not part of the provider source. Delete or move before opening a PR
against this repo.

---

## Title

Expose provider construction from a subset of ServicePackages, so SDK consumers can link only the services they use

## Body

### Context

We maintain [crossplane-contrib/provider-upjet-alibabacloud](https://github.com/crossplane-contrib/provider-upjet-alibabacloud),
a Crossplane provider generated with [upjet](https://github.com/crossplane/upjet).
It exposes 189 `alicloud_*` resources as Kubernetes CRDs.

We have just migrated it to the "no-fork" architecture: instead of executing
`terraform` and the provider plugin as subprocesses, we import this provider as a
Go module and call `alicloud.Provider()`, driving the returned
`*schema.Provider`'s CRUD functions in-process. This is the same approach
`provider-upjet-aws`, `-azure`, `-gcp` and `-oci` take with their respective
providers.

Thank you for the v2 SDK v2 migration — `alicloud.Provider()` returning
`*terraform-plugin-sdk/v2/helper/schema.Provider` is what makes this possible at
all. The v1 line returns `terraform.ResourceProvider` (plugin SDK v1), which has
no path to this architecture.

### The problem

`Provider()` is the only exported constructor, and it is a single function that
builds the provider schema, the 783-entry `DataSourcesMap`, the 1187-entry
`ResourcesMap`, wires `ConfigureContextFunc`, and calls `assembleProvider`.

Because the resource and data source maps are composite literals referencing
every constructor, calling `Provider()` makes all 1970 of them reachable, and the
Go linker cannot prune any. A consumer that needs 189 resources links all 1187;
a consumer that needs one service still links all of them.

We publish per-service packages (a `vpc` provider, an `ecs` provider, and so on),
so this is the difference between a ~40 MB and a ~298 MB binary per package.

### Measurements

Built with `-ldflags="-s -w"`, darwin/arm64, against `v2.0.0-beta4`
(commit `d31d4a12`). Each row is a minimal `main` referencing only the named
symbols. Rows 2, 4 and 5 were produced by injecting a generated file into package
`alicloud` that exposes a trimmed map literal, via a local `replace`, so the only
difference between rows is which constructors are reachable.

| What is linked | Binary | Marginal over floor |
|---|---|---|
| `alicloud` imported, `Provider()` **never called** | **37 MB** | — |
| 3 resources (`alicloud_vpc`, `_vswitch`, `_route_table`) | 40 MB | 3 MB |
| 189 resources (what our provider exposes) | 221 MB | 183 MB |
| all 783 data sources, no resources | 223 MB | 186 MB |
| `Provider()` — 1187 resources + 783 data sources | 260 MB | 222 MB |

Two things worth drawing out:

- **The 37 MB floor is the important number.** When nothing references the map
  literals the linker prunes them completely, so the mechanism we need already
  works — it is only the single entry point that forces retention.
- **The cost is per service SDK, not per resource.** Resources and data sources
  cost 183 MB and 186 MB separately but only 222 MB together, because they reach
  the same service clients. This is why trimming resource *counts* achieves
  little (1187 → 189 saves only 15%) while trimming to a *single service* gets
  almost all the way to the floor. It is also why we are not asking for a way to
  skip data sources — that would save almost nothing on its own.

### What we would like

The `alicloud/provider/registry` + `alicloud/provider/conns` model you have
introduced is already exactly the right shape. `conns.SDKResource.Factory` is an
exported `func() *schema.Resource`, and `alicloud/service/ims` imports only
`alicloud/provider/conns` and `alicloud/provider/fwadapt` — nothing from package
`alicloud`, nothing from `connectivity`. A service package in that form is
independently importable, and a consumer importing only it links only it.

The gap is that provider-level configuration cannot be obtained without the
resource maps. Concretely, we would like something along these lines:

```go
// NewProvider returns a provider with the given service packages registered and
// the provider-level schema and ConfigureContextFunc wired. It does not
// reference the provider.go map literals, so a caller linking only some service
// packages does not pay for the rest.
func NewProvider(sps ...conns.ServicePackage) (*schema.Provider, error)
```

The essential property is that `NewProvider` must not reference the legacy
`ResourcesMap` / `DataSourcesMap` literals, directly or transitively — including
not going through `registry.ServicePackages()`, since that aggregator's
package-level slice reaches every registered service.

This is additive. `Provider()` keeps working unchanged, `main.go` and the
acceptance tests are untouched, and resources still declared in the map literals
are simply not reachable through `NewProvider` until their service migrates.

### Secondary request: migration order

`registry.ServicePackages()` currently holds `ims` and `alicloudfunction`, and
`ims` declares only a `DataSources` entry — so there are no `SDKResources` in the
registry yet. The benefit above only lands for a service once its resources move
out of the `provider.go` literal.

If it helps prioritisation, the services our users depend on are: CS/ACK, ALB,
ECS, VPC, OSS, KMS, RAM, CDN, PolarDB, Tair/KVStore, CR, SLB, AliDNS, OOS,
Quotas, PrivateLink, MessageService, CloudMonitorService, FC v3, ACK One.

One constraint worth capturing in a lint or a test as the migration proceeds: a
package under `alicloud/service/` must not import package `alicloud`. A single
such import re-links the map literals and erases the benefit for every consumer.
`alicloud/service/ims` is already clean in this respect.

### Happy to help

We are glad to open a PR for `NewProvider` if you would like, and to share the
harness used for the table above so you can reproduce the numbers. We are also
happy to validate migrated services against our 189-resource surface as they
land.

This benefits any consumer that embeds the provider as a library rather than
running it as a plugin, not just Crossplane.
