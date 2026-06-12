# Bug bash — `--enable-control-plane-metrics` / `--disable-control-plane-metrics`

Pre-release build of the `aks-preview` extension for bug-bashing the new
`--enable-control-plane-metrics` / `--disable-control-plane-metrics` flags on
`az aks create` and `az aks update` before they merge upstream.

- Upstream PR: <https://github.com/Azure/azure-cli-extensions/pull/9931>
- In-box CLI mirror PR: <https://github.com/Azure/azure-cli/pull/33537>
- Source branch (this fork): `kadubey/aks-control-plane-metrics-upstream`

The flags surface the first-class API property
`azureMonitorProfile.metrics.controlPlane.enabled`, which replaces the previous
AFEC-gated preview. Enabling control-plane metrics turns on the Azure Monitor
managed Prometheus collection for `kube-apiserver`, `etcd`, `kube-scheduler`,
and `kube-controller-manager`.

> **Greenfield race fix:** on `az aks create`, the CP flip is intentionally
> deferred to a postprocessing PUT that runs **after** the DCRA is created.
> Otherwise the RP would schedule the control-plane-metrics collection (CCP)
> pod before its DCRA exists and it would crash-loop with "DCRA not found"
> until reconciliation. The brownfield `az aks update` path is unchanged —
> the DCRA already exists, so there is no race.

## Install

> Replace `<URL>` with the GitHub Release asset URL once you publish the wheel
> (see the **Publishing** section below). After publish, the URL pattern is:
>
> ```
> https://github.com/bragi92/azure-cli-extensions/releases/download/aks-preview-21.0.0b4-cpmetrics-bugbash/aks_preview-21.0.0b4-py2.py3-none-any.whl
> ```

```bash
# 1. Remove any installed aks-preview
az extension remove --name aks-preview 2>/dev/null

# 2. Install the bug-bash build
az extension add --yes --source <URL>

# 3. Confirm
az extension show --name aks-preview --query "{name:name, version:version}" -o table
# expected: aks-preview  21.0.0b4
```

## What to test

| # | Command | Expected result |
|---|---|---|
| 1 | `az aks create --enable-azure-monitor-metrics --enable-control-plane-metrics …` | Cluster created. After ~5 min, `az aks show` shows `azureMonitorProfile.metrics.controlPlane.enabled == true`. Default CCP metrics flow into AMW. |
| 2 | `az aks create --enable-control-plane-metrics …` (no `--enable-azure-monitor-metrics`) | **Rejected** client-side (`RequiredArgumentMissingError`). |
| 3 | `az aks update --enable-control-plane-metrics` on cluster with AMW already on | CP metrics enabled; metrics flow within ~5 min. |
| 4 | `az aks update --enable-control-plane-metrics` on cluster with AMW **off** | **Rejected** (`RequiredArgumentMissingError`). |
| 5 | `az aks update --enable-azure-monitor-metrics --enable-control-plane-metrics …` (AMW off → on) | AMW + CP enabled in one call. |
| 6 | `az aks update --disable-control-plane-metrics` | CP off, AMW left intact. |
| 7 | `az aks update --enable-control-plane-metrics --disable-control-plane-metrics` | **Rejected** (`MutuallyExclusiveArgumentError`). |
| 8 | `az aks update --enable-control-plane-metrics --disable-azure-monitor-metrics` | **Rejected** (`MutuallyExclusiveArgumentError`). |

After **disable**, wait ~15 minutes before re-asserting metrics in AMW — the
previous CCP pod's series will continue showing until they age out.

## Verifying control-plane metrics in Azure Monitor Workspace

```bash
AMW_QUERY_ENDPOINT=$(az monitor account show -g <rg> -n <amw> \
  --query metrics.prometheusQueryEndpoint -o tsv)

az rest --method get \
  --url "$AMW_QUERY_ENDPOINT/api/v1/query?query=apiserver_request_total" \
  --resource "https://prometheus.monitor.azure.com"
```

Look for these default CP metric families (per
[Default Prometheus metrics configuration in Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/containers/prometheus-metrics-scrape-default)):

- `apiserver_request_total`
- `apiserver_request_duration_seconds_*`
- `etcd_server_has_leader`
- `etcd_mvcc_db_total_size_in_bytes`
- `process_start_time_seconds`

## Reporting bugs

Drop a comment on <https://github.com/Azure/azure-cli-extensions/pull/9931>
with:

- `az --version` output
- the exact command run
- `az aks show -g <rg> -n <name> --query azureMonitorProfile -o jsonc`
- error message (if any)

## Publishing the wheel (extension maintainers)

The `aks-preview` wheel is built from the PR branch with:

```bash
azdev extension repo add /path/to/azure-cli-extensions
azdev extension build aks-preview
# → dist/aks_preview-21.0.0b4-py2.py3-none-any.whl
```

Then publish a GitHub Release on the fork so testers have a public download URL:

1. Open <https://github.com/bragi92/azure-cli-extensions/releases/new>
2. **Tag:** `aks-preview-21.0.0b4-cpmetrics-bugbash` → "Create new tag on publish"
3. **Target:** `kadubey/aks-control-plane-metrics-upstream`
4. **Title:** `aks-preview 21.0.0b4 (control-plane-metrics bug bash)`
5. **Description:** point at this file (or paste the **Install** + **What to test** sections above)
6. **Attach binaries:** drag `dist/aks_preview-21.0.0b4-py2.py3-none-any.whl`
7. Tick **"Set as a pre-release"**
8. Click **Publish release**

The asset URL GitHub generates is the `<URL>` value testers paste into the
install command at the top of this file.

### Alternative distribution channels

- **Azure Storage blob** (public or SAS): `az storage blob upload …` then share
  the URL. Same `az extension add --source <url>` install flow.
- **Direct file share** (Teams / email): recipient runs
  `az extension add --source ./aks_preview-21.0.0b4-py2.py3-none-any.whl --yes`.
- **Official internal dev pipeline:** the `azure-cli-extensions` Azure DevOps
  pipeline produces signed wheels; ask in the AKS CLI Teams channel for the
  current artifact path if you'd prefer that route.
