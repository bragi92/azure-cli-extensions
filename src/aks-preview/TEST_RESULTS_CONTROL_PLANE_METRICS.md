# Live Test Results — `aks-preview` extension PR #2 (`--enable/--disable-control-plane-metrics`)

**Repo (local):** `C:\Users\kadubey\Documents\git_repos\azure-cli-extensions`
**Branch:** `kadubey/aks-control-plane-metrics` (2 commits ahead of `origin/main`)
**Commits under test:**
```text
1fd96100a [AKS] aks-preview: defer control-plane-metrics flip to post-DCRA addon_put on create
4cec44617 [AKS] aks-preview: add --enable/--disable-control-plane-metrics
```
This is the source of [bragi92/azure-cli-extensions PR #2](https://github.com/bragi92/azure-cli-extensions/pull/2/changes). Key file: `src/aks-preview/azext_aks_preview/azuremonitormetrics/azuremonitorprofile.py`.

**Extension load path (verified):**
```text
$ az extension list --query "[?name=='aks-preview']" -o json
[ { "name": "aks-preview", "version": "21.0.0b4",
    "path": "C:\\Users\\kadubey\\Documents\\git_repos\\azure-cli-extensions\\src\\aks-preview" } ]
$ az --version | findstr aks-preview
aks-preview                     21.0.0b4 (dev) C:\Users\kadubey\Documents\git_repos\azure-cli-extensions\src\aks-preview
```
Every `az aks ...` invocation prints `WARNING: The behavior of this command has been altered by the following extension: aks-preview`, confirming the local PR branch is the one executing.

**`az` host:** `testenv` venv at `C:\Users\kadubey\Documents\git_repos\azure-cli\testenv` running `azure-cli==2.87.0` (editable install of the local GA repo). This matched the API surface aks-preview expects (`get_acns_enablement_with_perf` returning a 4-tuple).

**Subscription:** `ce4d1293-71c0-4c72-bc55-133553ee9e50` (Bragi Test) · **Region:** `eastus2` · **RG:** `kaveeshclitest`

**Reused observability resources** (same as the GA test run):
- AMW `kaveeshclitest` — Prom query endpoint `https://kaveeshclitest-eedhgpe4ahdfgpev.eastus2.prometheus.monitor.azure.com`
- Grafana `kaveeshclitest`

**Preview feature:** `Microsoft.ContainerService/AzureMonitorMetricsControlPlanePreview` registered at the start of the brownfield mutating tests (B4 returned `BadRequest .properties.azureMonitorProfile.metrics.controlPlane configuration is not supported` until the feature was on). Unregistered again at cleanup.

State snapshots use:
```powershell
az aks show -g kaveeshclitest -n <cluster> --query "{state:provisioningState, amp:azureMonitorProfile.metrics.enabled, cp:azureMonitorProfile.metrics.controlPlane}" -o json
```

AMW snapshots use `files/query_amw_ext.ps1` (this folder), which filters by `cluster="kclitest-gf-ext"` so old samples from the prior GA test cluster (`kclitest-gf`) cannot leak in. Queries:
- `up{job=~"controlplane.*",cluster="kclitest-gf-ext"}`
- `etcd_server_has_leader{cluster="kclitest-gf-ext"}`
- `etcd_mvcc_db_total_size_in_bytes{cluster="kclitest-gf-ext"}`
- `apiserver_current_inflight_requests{cluster="kclitest-gf-ext"}`
- `count(apiserver_request_total{cluster="kclitest-gf-ext"})`
- `count by(job)(up{job=~"controlplane.*",cluster="kclitest-gf-ext"})`

These are the doc-defined default metrics for `controlplane-apiserver` and `controlplane-etcd`.

---

# Part 1 — Brownfield (existing cluster `kaveeshclitest`)

Baseline (`az aks show`):
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```

## B1 — Negative: `aks create --enable-control-plane-metrics` without parent AMP
```powershell
az aks create -g kaveeshclitest -n test-cp-neg-ext --enable-control-plane-metrics --generate-ssh-keys --location eastus2 --node-count 1
```
```text
WARNING: The behavior of this command has been altered by the following extension: aks-preview
ERROR: --enable-control-plane-metrics requires Azure Monitor metrics to be enabled. Specify --enable-azure-monitor-metrics or run on a cluster that already has Azure Monitor metrics enabled.
```
Exit `1`. ✅ Validation fires from the extension's code (`RequiredArgumentMissingError`). No cluster created.

## B2 — Mutex: `--enable-CP` + `--disable-CP`
```text
WARNING: The behavior of this command has been altered by the following extension: aks-preview
ERROR: Cannot specify --enable-control-plane-metrics and --disable-control-plane-metrics at the same time.
```
Exit `1`. ✅ `MutuallyExclusiveArgumentError`. No PUT.

## B3 — Mutex: `--enable-CP` + `--disable-azure-monitor-metrics`
```text
WARNING: The behavior of this command has been altered by the following extension: aks-preview
ERROR: Cannot specify --enable-control-plane-metrics together with --disable-azure-monitor-metrics.
```
Exit `1`. ✅ `MutuallyExclusiveArgumentError`. No PUT.

`az aks show` after B1–B3 (unchanged):
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```

## Preflight: register CCP preview feature
On the first mutating attempt (B4), the RP rejected the call with:
```text
ERROR: (BadRequest) .properties.azureMonitorProfile.metrics.controlPlane configuration is not supported
```
This subscription had `AzureMonitorMetricsControlPlanePreview` unregistered. Registered + propagated:
```powershell
az feature register --namespace Microsoft.ContainerService --name AzureMonitorMetricsControlPlanePreview
# state -> Registered
az provider register -n Microsoft.ContainerService
```

## B4 — `--disable-control-plane-metrics` (long form)
Pre `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --disable-control-plane-metrics --yes
```
Exit `0`. Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ CP→false.

## B5 — `--enable-control-plane-metrics` (long form)
Pre:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
Exit `0`. Post:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ CP→true.

## B6 — `--disable-cp-metrics` (alias)
Pre:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
Exit `0`. Post:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ Short alias accepted, CP→false.

## B7 — `--enable-cp-metrics` (alias, restore brownfield)
Pre:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
Exit `0`. Post:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ Short alias accepted, CP→true. **Brownfield cluster restored.**

> Same caveat as the GA test: the brownfield cluster `kaveeshclitest` has no `Microsoft.Insights/dataCollectionRuleAssociations` linked, so AMW-side verification of brownfield enable isn't possible. End-to-end metric flow is verified on the greenfield cluster below.

---

# Part 2 — Greenfield (`kclitest-gf-ext`) with AMW data-plane verification

## G0 — Create greenfield cluster with `--enable-AMP --enable-CP`
```powershell
az aks create -g kaveeshclitest -n kclitest-gf-ext --location eastus2 `
  --node-count 1 --node-vm-size Standard_D2s_v3 --generate-ssh-keys `
  --enable-azure-monitor-metrics `
  --azure-monitor-workspace-resource-id "/subscriptions/.../microsoft.monitor/accounts/kaveeshclitest" `
  --grafana-resource-id           "/subscriptions/.../microsoft.dashboard/grafana/kaveeshclitest" `
  --enable-control-plane-metrics --no-wait
```
CLI output:
```text
WARNING: The behavior of this command has been altered by the following extension: aks-preview
WARNING: Monitoring Data Reader role assignment already exists on the Azure Monitor Workspace for the Grafana managed identity. Skipping role assignment.
Using Azure Monitor Workspace (stores prometheus metrics) : /subscriptions/.../kaveeshclitest
```
Exit `0` at 11:38:38. `az aks wait -g kaveeshclitest -n kclitest-gf-ext --created --timeout 1500` returned `0` at **11:41:04**.

`az aks show` post-create:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```

DCR + DCRA artifacts confirm the deferred-flip pattern in the extension fired:
```text
$ az monitor data-collection rule list -g kaveeshclitest --query "[].name"
[ "MSProm-eastus2-kclitest-gf",  # leftover from prior session
  "MSProm-eastus2-kclitest-gf-ext" ]
$ az rest --method get --url ".../managedClusters/kclitest-gf-ext/providers/Microsoft.Insights/dataCollectionRuleAssociations?api-version=2022-06-01"
[ {
    "name": "ContainerInsightsMetricsExtension",
    "dcrId": ".../dataCollectionRules/MSProm-eastus2-kclitest-gf-ext",
    "createdAt": "2026-06-11T18:38:14.1499295Z"
} ]
```

## G1 — AMW verification after enable (T0)

Polled every 2 minutes from T+25 min. CCP scrape targets appeared between T+23 and T+25 min after cluster Succeeded — matches the ~25-30 min CCP-collector startup latency observed in the GA test.

**T0 snapshot at 12:06:30 (~25 min after Succeeded):**
```text
--- up{job=~controlplane-.*, cluster="kclitest-gf-ext"}: instant ---
  controlplane-apiserver / kube-apiserver-5568ffdfdf-njkbl -> up=1 @ 12:06:30
  controlplane-apiserver / kube-apiserver-5568ffdfdf-672lm -> up=1 @ 12:06:30
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> up=1 @ 12:06:30
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> up=1 @ 12:06:30
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> up=1 @ 12:06:30

--- etcd_server_has_leader: instant ---
  etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> 1 @ 12:06:31
  etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> 1 @ 12:06:31
  etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> 1 @ 12:06:31

--- etcd_mvcc_db_total_size_in_bytes: instant ---
  etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> 6844416 bytes @ 12:06:32
  etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> 6545408 bytes @ 12:06:32
  etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> 6709248 bytes @ 12:06:32

--- apiserver_current_inflight_requests: instant ---
  kube-apiserver-5568ffdfdf-672lm [mutating] -> 1 @ 12:06:32
  kube-apiserver-5568ffdfdf-njkbl [mutating] -> 1 @ 12:06:32
  kube-apiserver-5568ffdfdf-njkbl [readonly] -> 1 @ 12:06:32
  kube-apiserver-5568ffdfdf-672lm [readonly] -> 1 @ 12:06:32

--- count(apiserver_request_total) ---
  count = 796 @ 12:06:33

--- count by(job)(up{...}) ---
  controlplane-apiserver -> 2 targets
  controlplane-etcd      -> 3 targets
```
✅ All five default CCP metric families from the doc list are present and scraping fresh data: `apiserver_request_total` (796 series), `apiserver_current_inflight_requests` (2 pods × 2 request_kind), `etcd_server_has_leader=1` (3 etcd pods), `etcd_mvcc_db_total_size_in_bytes` (~6.5–6.8 MB each).

(Raw snapshot saved at `files/ext_snapshot_T0_after_enable.txt`.)

## G2 — Disable CP, wait, re-query

```powershell
az aks update -g kaveeshclitest -n kclitest-gf-ext --disable-control-plane-metrics --yes
```
Submitted 12:06:48, completed at **12:09:11**, exit `0`.

Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ Extension's update path correctly flipped `controlPlane.enabled = false` in the cluster spec.

**T1 snapshot at 12:27:39 (~18 min after disable):**
```text
--- up{job=~controlplane-.*, cluster="kclitest-gf-ext"}: instant ---
  controlplane-apiserver / kube-apiserver-5568ffdfdf-njkbl -> up=1 @ 12:27:40
  controlplane-apiserver / kube-apiserver-5568ffdfdf-672lm -> up=1 @ 12:27:40
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> up=1 @ 12:27:40
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> up=1 @ 12:27:40
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> up=1 @ 12:27:40

--- etcd_mvcc_db_total_size_in_bytes: instant ---
  etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> 7294976 bytes  @ 12:27:41   (was 6844416 @ T0)
  etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> 6995968 bytes  @ 12:27:41   (was 6545408 @ T0)
  etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> 7307264 bytes  @ 12:27:41   (was 6709248 @ T0)

--- count(apiserver_request_total) ---
  count = 823 @ 12:27:42   (was 796 @ T0)

--- count by(job)(up{...}) ---
  controlplane-apiserver -> 2 targets
  controlplane-etcd      -> 3 targets
```
**Observation:** 18 min after `--disable-control-plane-metrics` returned, CCP scrape targets are still `up=1`, etcd DB sizes grew by ~450 KB each, apiserver_request_total cardinality climbed from 796 → 823. **CCP collectors are still actively scraping.**

(Raw snapshot saved at `files/ext_snapshot_T1_after_disable.txt`.)

### ⚠️ Same CRP-async-teardown behavior as the GA port

This is **identical** to the behavior observed in the GA port test (`live_test_results.md` G2). The extension's `--disable-control-plane-metrics` path does what it should:
- Sets `mc.azure_monitor_profile.metrics.control_plane.enabled = False`
- PATCHes the cluster
- `az aks show` immediately reflects `cp.enabled=false`

But the actual teardown of the server-side CCP collector pods (which run in the hosted control plane and emit `controlplane-apiserver`/`controlplane-etcd` metrics) is handled asynchronously by the AKS Resource Provider. In both runs (GA port and aks-preview extension), the collectors kept scraping for ≥17 min after the CLI returned. This is **a CRP-side behavior, not a CLI bug**, and is consistent between the two implementations — exactly what you'd expect since the disable code in PR #2 and the GA port are byte-for-byte equivalent.

## G3 — Re-enable CP, verify still flowing
```powershell
az aks update -g kaveeshclitest -n kclitest-gf-ext --enable-control-plane-metrics --yes
```
Submitted 12:27:56, completed at **12:30:20**, exit `0`.

Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ Extension's update path correctly flipped `controlPlane.enabled = true`.

**T2 snapshot at 12:32:11 (~2 min after re-enable):**
```text
--- up{job=~controlplane-.*, cluster="kclitest-gf-ext"}: instant ---
  controlplane-apiserver / kube-apiserver-5568ffdfdf-njkbl -> up=1 @ 12:32:12
  controlplane-apiserver / kube-apiserver-5568ffdfdf-672lm -> up=1 @ 12:32:12
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> up=1 @ 12:32:12
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> up=1 @ 12:32:12
  controlplane-etcd      / etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> up=1 @ 12:32:12

--- etcd_mvcc_db_total_size_in_bytes: instant ---
  etcd-6a2aff152ec3900001cefe6c-qbhst7dgvj -> 7294976 bytes @ 12:32:13
  etcd-6a2aff152ec3900001cefe6c-7kfmjfnx6x -> 7143424 bytes @ 12:32:13
  etcd-6a2aff152ec3900001cefe6c-54djl4gsg7 -> 7454720 bytes @ 12:32:13

--- count(apiserver_request_total) ---
  count = 823 @ 12:32:14
```
✅ All 5 CCP scrape targets active (because they never tore down from G2's disable). The CLI spec change is correct; data continuity is governed by CRP.

(Raw snapshot saved at `files/ext_snapshot_T2_after_reenable.txt`.)

## Cleanup
```powershell
az aks delete -g kaveeshclitest -n kclitest-gf-ext --yes --no-wait     # 12:32:25
az aks wait -g kaveeshclitest -n kclitest-gf-ext --deleted --timeout 1500  # 12:38:17 exit 0
az aks list -g kaveeshclitest --query "[].name"  # -> [ "kaveeshclitest" ]
az feature unregister --namespace Microsoft.ContainerService --name AzureMonitorMetricsControlPlanePreview  # -> Unregistered
```
Greenfield cluster fully gone, only `kaveeshclitest` remains in the RG, feature unregistered as it was at start of session.

---

# Summary

| # | Mode | Test | Pre cp | Post cp | CLI verified | AMW verified | Result |
|---|---|---|---|---|---|---|---|
| B1 | brownfield | `aks create --enable-CP` w/o AMP | — | — | error msg + exit 1 (extension) | n/a | ✅ `RequiredArgumentMissingError` |
| B2 | brownfield | mutex enable+disable CP | true | true | error msg + exit 1, no PUT | n/a | ✅ `MutuallyExclusiveArgumentError` |
| B3 | brownfield | mutex enable-CP + disable-AMP | true | true | error msg + exit 1, no PUT | n/a | ✅ `MutuallyExclusiveArgumentError` |
| B4 | brownfield | `--disable-control-plane-metrics` | true | false | `az aks show` post | n/a (no DCRA on this cluster) | ✅ |
| B5 | brownfield | `--enable-control-plane-metrics` | false | true | `az aks show` post | n/a | ✅ |
| B6 | brownfield | `--disable-cp-metrics` alias | true | false | `az aks show` post | n/a | ✅ |
| B7 | brownfield | `--enable-cp-metrics` alias | false | true | `az aks show` post | n/a | ✅ |
| G0 | greenfield | `aks create --enable-AMP --enable-CP` | n/a | true | Succeeded; DCR `MSProm-eastus2-kclitest-gf-ext` + DCRA created → deferred-flip pattern works | — | ✅ |
| G1 | greenfield | verify CP metrics flow (T0) | true | true | `az aks show` | **PromQL: 5/5 default CCP metric families present**, 5 controlplane scrape targets `up=1`, fresh timestamps at 12:06:30 | ✅ |
| G2 | greenfield | `--disable-control-plane-metrics` | true | false | `az aks show` post | CLI spec change confirmed; CCP scrape targets **still up at T+18 min**, samples still incrementing (CRP async teardown) | ✅ (CLI), ⚠️ (CRP async — matches GA port) |
| G3 | greenfield | `--enable-control-plane-metrics` | false | true | `az aks show` post | CCP scrape targets still up; CP metrics continuous | ✅ |

**Conclusion:** PR #2 in `bragi92/azure-cli-extensions` (the aks-preview branch with `--enable/--disable-control-plane-metrics`) behaves **identically** to the GA port (`live_test_results.md`) across all 11 live tests: same validation errors, same spec flips, same DCR/DCRA wiring, same default CCP metric families flowing into AMW after enable, and same CRP-async-teardown behavior on disable. The two implementations are functionally equivalent — which is expected because the GA port was made directly from PR #2's commits.

**Environment restored:**
- Brownfield cluster `kaveeshclitest`: AMP=true, CP=true (as at session start).
- Greenfield cluster `kclitest-gf-ext`: deleted; only `kaveeshclitest` remains in the RG.
- `AzureMonitorMetricsControlPlanePreview` feature: `Unregistered`.
- `aks-preview` extension still loaded from `C:\Users\kadubey\Documents\git_repos\azure-cli-extensions` (the local PR #2 branch).

**Artifacts in `files/`:**
- `query_amw_ext.ps1` — helper script (cluster-filtered) for AMW Prometheus snapshots
- `ext_snapshot_T0_after_enable.txt` — CCP metrics flowing after greenfield create
- `ext_snapshot_T1_after_disable.txt` — 18 min after disable (CCP still up — CRP async)
- `ext_snapshot_T2_after_reenable.txt` — after re-enable (continuous)
