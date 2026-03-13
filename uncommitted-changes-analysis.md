# Uncommitted Changes Analysis

This document summarizes the current uncommitted changes in the workspace. Related files are grouped by the bug, error, or missing feature they address.

## Problems and Solutions

| # | Files | Problem | Solution |
| --- | --- | --- | --- |
| 1 | `deploy/001-iac/main.tf`; `deploy/001-iac/modules/sil/observability.tf`; `deploy/001-iac/modules/sil/variables.deps.tf` | AKS observability resources were gated by `null` checks on typed dependency objects instead of the actual feature flags. When the monitor workspace or data collection endpoint was intentionally disabled, the module could still try to create data collection rules and associations because the object shape was present. This pattern can also surface as Terraform `Invalid count argument` errors when `count` depends on module outputs backed by provider-computed attributes, because Terraform may not be able to prove the `null` check during planning. | Added explicit `should_deploy_monitor_workspace` and `should_deploy_dce` inputs to the SIL module and switched the DCR and DCRA `count` conditions to those booleans. This makes resource creation follow the intended observability feature flags instead of object presence and removes the need to derive `count` from dependency object nullness. |
| 2 | `deploy/001-iac/modules/vpn/main.tf`; `deploy/001-iac/modules/vpn/variables.tf`; `deploy/001-iac/vpn/variables.tf`; `deploy/001-iac/vpn/terraform.tfvars.example` | The VPN gateway module used an outdated BGP argument name and a non-AZ default SKU. That combination risks provider validation errors and produces defaults that do not match the recommended Azure-supported SKU family. | Replaced `enable_bgp` with `bgp_enabled` and changed the default VPN gateway SKU from `VpnGw1` to `VpnGw1AZ`. The example tfvars file now shows the AZ-backed SKU explicitly. |
| 3 | `deploy/001-iac/terraform.tfvars.example` | The example Spot GPU pool required `min_count = 1`, which keeps one node allocated even when the cluster is idle. That increases baseline cost and can also block deployment in low-capacity regions. | Changed the example Spot pool minimum size to `0` so autoscaling can scale the pool down completely when unused. |
| 4 | `deploy/002-setup/01-deploy-robotics-charts.sh` | `resolve_latest_gpu_operator_version` used Helm commands that wrote status text to stdout. When this function was used in command substitution, repo-management noise could pollute the captured chart version value. | Redirected `helm repo add` and `helm repo update` output to `/dev/null` so the function returns only the GPU Operator version string. Concrete example: before, command substitution could capture output like `"Hang tight while we grab the latest from your chart repositories...\nv25.3.0"` or other repo-status text instead of just a version. After the change, the function output is only `v25.3.0`. |
| 5 | `deploy/002-setup/03-deploy-osmo-control-plane.sh`; `deploy/002-setup/lib/common.sh`; `deploy/002-setup/manifests/aks-secret-provider-class-postgres-only.yaml` | The control-plane script did not fully support in-cluster Redis. It always assumed an external Redis secret flow, did not ensure a usable StorageClass for the in-cluster chart, and always synced both Postgres and Redis secrets from Key Vault even when Redis should be managed inside Kubernetes. | Added default StorageClass detection for `--use-incluster-redis`, enabled Redis chart values with `storageClassName`, created the in-cluster Redis secret placeholder in Kubernetes, and introduced a Postgres-only `SecretProviderClass` manifest so Key Vault sync excludes Redis when Redis is deployed in-cluster. |
| 6 | `deploy/002-setup/lib/common.sh` | `osmo_login_and_setup` assumed the connected OSMO service exposed the newer `user` management API. Against older OSMO service versions, the script failed with `404 Not Found` during user bootstrap even though the rest of the deployment could proceed. | Added unsupported-API detection for OSMO CLI output. When the `user` API is unavailable, the script now logs a warning and skips dev-user bootstrap instead of aborting the deployment. |
| 7 | `deploy/002-setup/04-deploy-osmo-backend.sh`; `deploy/002-setup/lib/common.sh` | The backend deployment assumed the OSMO service supported personal token APIs such as `osmo token list` and `osmo token set`. The current control plane runs an older `6.0.0` service that returns `404` or `422` for those newer endpoints. | Added token API probing and compatibility detection. When the token API is unavailable, the backend deployment switches into a compatibility path instead of failing immediately on token generation or token discovery. |
| 8 | `deploy/002-setup/04-deploy-osmo-backend.sh` | Reusing the local CLI login token as a backend operator token did not work. The backend operator expects a token type that the older service can refresh, and the dev-login token caused `Access Token is invalid` crashes in the listener and worker. | Added a dev-login compatibility mode for backend operator deployments. The script patches the listener and worker deployments to pass `--method dev --username guest` so they authenticate using the developer-mode header path that the older service supports. |
| 9 | `deploy/002-setup/04-deploy-osmo-backend.sh` | The dev-login compatibility path initially still crashed because the backend operator binaries call `load_kube_config()` when `--method dev` is active. The containers run as non-root, so mounting the generated kubeconfig under `/root/.kube` left the file inaccessible to the process. | Added generation of per-service-account kubeconfig secrets, mounted them under `/tmp/osmo-kubeconfig`, and set `HOME=/tmp` and `KUBECONFIG=/tmp/osmo-kubeconfig/config` on the backend operator deployments. This provides a readable kubeconfig for non-root containers while preserving in-cluster API access. |
| 10 | `deploy/002-setup/04-deploy-osmo-backend.sh` | After live `kubectl patch` changes to deployment args, later `helm upgrade` runs hit field-manager conflicts on the operator deployments. That blocked repeat runs of the backend deployment script. | Added a delete-and-recreate reconcile path for the backend listener and worker deployments when dev-login compatibility mode is active, then re-applied the compatibility patches and waited for rollout. This keeps the script rerunnable. |
| 11 | `deploy/002-setup/lib/common.sh` | Two smaller maintenance issues were present in shared shell helpers: the exported local `osmo` wrapper triggered a shellcheck false positive, and `tf_require` packed multiple locals onto one line, which reduced readability. | Added a targeted shellcheck suppression for the exported wrapper function and split `tf_require` local declarations across separate lines. These are maintainability fixes rather than behavior changes. |
| 12 | `package-lock.json` | The lockfile root package name was still `azure-nvidia-robotics-reference-architecture`, which no longer matched the repository package identity. That causes unnecessary lockfile churn and misleading package metadata during npm operations. | Refreshed the lockfile so the root package name matches `physical-ai-toolchain`. The remaining peer-flag adjustments are lockfile regeneration side effects, not functional runtime changes. |

## Current Outcome

The uncommitted setup-script changes address two deployment failures that were reproducible in this workspace:

| Area | Outcome |
| --- | --- |
| OSMO control plane | Supports in-cluster Redis and no longer fails when the older service lacks the `user` API. |
| OSMO backend | Deploys successfully against the current older OSMO service by switching the operator to dev-login compatibility mode and mounting a non-root-readable kubeconfig. |

## Suggested PR Plan

Use one PR per commit when you want the narrowest review surface. Each commit message already matches a single problem statement.

| PR | Commit | Scope |
| --- | --- | --- |
| 1 | `c63f909` | Gate SIL observability resources with explicit feature flags. |
| 2 | `600f01c` | Update VPN gateway defaults and provider argument naming. |
| 3 | `476bc2f` | Let the example Spot pool scale to zero. |
| 4 | `2a8341c` | Silence Helm repo noise during GPU Operator version lookup. |
| 5 | `5b84418` | Harden shared OSMO setup helpers for older control-plane APIs. |
| 6 | `88b3475` | Add in-cluster Redis support for OSMO control-plane deployment. |
| 7 | `a056608` | Add legacy-login compatibility for OSMO backend deployment. |
| 8 | `6d692f3` | Refresh lockfile metadata to match the repository package name. |

If you want fewer PRs, merge PRs 5 through 7 into a single OSMO compatibility PR. Those three commits are still independently reviewable, but they all address deployment compatibility with the current OSMO service in this workspace.
