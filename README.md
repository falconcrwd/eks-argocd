# eks-argocd — Falcon Operator on EKS via ArgoCD

This repository is a GitOps source of truth for deploying CrowdStrike's **Falcon Operator** and its `FalconDeployment` custom resource into an EKS cluster (`jwong08-eks-cluster`, "cluster B") from another EKS cluster running ArgoCD (`jwong08-argo-cluster`, "cluster A"). Two `workflow_dispatch` GitHub Actions workflows automate the bootstrap and teardown; day-to-day changes to sensor config are pure GitOps — commit to `main`, walk away.

## Table of contents

- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
  - [1. GitHub repository secrets](#1-github-repository-secrets)
  - [2. AWS — OIDC identity provider](#2-aws--oidc-identity-provider-for-github-actions)
  - [3. AWS — IAM role for the runner](#3-aws--iam-role-assumed-by-the-workflow)
  - [4. AWS — EKS access entries on both clusters](#4-aws--eks-access-entries-on-both-clusters)
  - [5. AWS — VPC / security group for cluster B](#5-aws--security-group-ingress-from-cluster-a-to-cluster-b)
  - [6. Cluster A (ArgoCD) — installed and repo registered](#6-cluster-a-argocd-installed-and-this-repo-registered)
  - [7. Cluster B — public endpoint reachable](#7-cluster-b--api-endpoint-must-be-reachable-by-the-runner)
  - [8. CrowdStrike Falcon — API client + sensor update policy](#8-crowdstrike-falcon--api-client-scopes-and-sensor-update-policy)
- [Running the workflows](#running-the-workflows)
- [Day-to-day GitOps flow](#day-to-day-gitops-flow)
- [Teardown](#teardown)
- [Troubleshooting](#troubleshooting)

## Architecture

```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│  GitHub Actions runner          │         │  jwong08-argo-cluster (cluster A)│
│  (ubuntu-latest, ephemeral)     │         │                                 │
│  - assumes AWS role via OIDC    │         │  ArgoCD server + controllers    │
│  - port-forwards argocd-server  │◀───────▶│  - repo:  this GitHub repo      │
│  - runs `argocd` CLI            │  8080→  │  - target: cluster B (added by  │
│                                 │  443    │            the workflow)       │
└─────────────────────────────────┘         └──────────────┬──────────────────┘
                                                           │  ArgoCD applies the
                                                           │  FalconDeployment CR
                                                           ▼
                                            ┌─────────────────────────────────┐
                                            │  jwong08-eks-cluster (cluster B)│
                                            │                                 │
                                            │  Falcon Operator (installed by  │
                                            │  the workflow, imperatively)    │
                                            │  ↳ reconciles FalconDeployment  │
                                            │    ↳ Node Sensor (DaemonSet)    │
                                            │    ↳ Container Sensor (inject.) │
                                            │    ↳ KAC (admission controller) │
                                            │    ↳ IAR (image analyzer)       │
                                            └─────────────────────────────────┘
```

**Cluster A** hosts ArgoCD. It watches this repo and syncs the `FalconDeployment` CR into cluster B. **Cluster B** is the workload cluster where the Falcon Operator and the sensors actually run.

Two workflows live in `.github/workflows/`:

- **`deploy-falcon.yaml`** — bootstraps and re-syncs: installs the Falcon Operator on cluster B, creates the `falcon-secrets` Secret on first run, registers cluster B with ArgoCD, and creates/updates the ArgoCD Application.
- **`teardown-falcon.yaml`** — reverses the above: removes the ArgoCD app, deletes the `FalconDeployment` CR (letting the operator reap child workloads via finalizers), deletes the operator manifest, and unregisters cluster B from ArgoCD.

## Repository layout

```
.github/workflows/
├── deploy-falcon.yaml            # Manual bootstrap + re-sync
└── teardown-falcon.yaml          # Manual teardown

clusters/
└── jwong08-eks-cluster/
    └── falcon-operator/
        ├── kustomization.yaml    # Kustomize base ArgoCD renders
        └── manifests/
            └── falcondeployment.yaml  # The one resource that is GitOps-managed

.gitignore
README.md                         # (this file)
```

Only `falcondeployment.yaml` is managed by ArgoCD. The operator itself, the `falcon-secrets` Secret, and the ArgoCD cluster registration are managed by the workflow (imperatively, once) because they are prerequisites of the GitOps loop, not part of it.

## Prerequisites

The 8 subsections below are one-time setup. Once all of them are done, running `deploy-falcon.yaml` should succeed end-to-end without any manual intervention.

### 1. GitHub repository secrets

**Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value | Used by |
| --- | --- | --- |
| `AWS_ROLE_TO_ASSUME` | ARN of the IAM role from §3 (e.g. `arn:aws:iam::123456789012:role/jwong08-argocd-role`) | AWS OIDC auth step in both workflows |
| `FALCON_CLIENT_ID` | Falcon API client ID | `create_secret: true` step (first run only) |
| `FALCON_CLIENT_SECRET` | Falcon API client secret | `create_secret: true` step (first run only) |

The two `FALCON_*` secrets are only required the first time you run the workflow with `create_secret: true`.

### 2. AWS — OIDC identity provider for GitHub Actions

Adds GitHub as a trusted OIDC issuer for your AWS account. Skip if you've already added it for another repo in the same account.

**IAM → Identity providers → Add provider**

| Field | Value |
| --- | --- |
| Provider type | **OpenID Connect** |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

Click **Get thumbprint** so AWS fills it in, then **Add provider**.

### 3. AWS — IAM role assumed by the workflow

**IAM → Roles → Create role**

- **Trusted entity type:** Web identity
- **Identity provider:** `token.actions.githubusercontent.com`
- **Audience:** `sts.amazonaws.com`
- **GitHub organization:** _`<your-org>`_
- **GitHub repository:** _`<this-repo>`_

The generated trust policy looks like this (scope the `sub` to your org/repo, optionally to a branch):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":  { "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:*" }
    }
  }]
}
```

Attach an inline permissions policy that allows `eks:DescribeCluster` on both clusters (this is what `aws eks update-kubeconfig` calls):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["eks:DescribeCluster", "eks:ListClusters"],
    "Resource": [
      "arn:aws:eks:<REGION>:<ACCOUNT_ID>:cluster/jwong08-argo-cluster",
      "arn:aws:eks:<REGION>:<ACCOUNT_ID>:cluster/jwong08-eks-cluster"
    ]
  }]
}
```

Copy the role's ARN into GitHub secret `AWS_ROLE_TO_ASSUME` (§1).

> **Common failure mode:** `AccessDeniedException` when calling `DescribeCluster` even though "the policy exists" almost always means the policy is attached to a **different** role than the one being assumed. Check `aws iam list-attached-role-policies --role-name <role>` against the ARN in the error.

### 4. AWS — EKS access entries on both clusters

`eks:DescribeCluster` only opens the AWS API layer. Kubernetes RBAC is a separate authorization plane; you must grant the IAM role Kubernetes permissions on **both** clusters.

For **each** cluster (`jwong08-argo-cluster` and `jwong08-eks-cluster`):

**EKS → Clusters → `<cluster-name>` → Access → IAM access entries → Create access entry**

- **IAM principal:** the role ARN from §3
- **Type:** Standard
- **Kubernetes groups:** _(leave blank)_
- Next → **Access policy:** `AmazonEKSClusterAdminPolicy`, scope **Cluster**

Cluster-admin is the simplest scope; tighten later once you're comfortable. The workflow needs to read secrets in the `argocd` namespace on cluster A, install the Falcon Operator's release manifest on cluster B, and `argocd cluster add` (which creates a `argocd-manager` ServiceAccount + ClusterRoleBinding on the target).

### 5. AWS — security group ingress from cluster A to cluster B

**This is the piece most people miss.** `argocd cluster add` succeeds on the runner, but the ArgoCD server pod running inside cluster A must then be able to reach cluster B's Kubernetes API to verify the connection and later to sync. If ArgoCD's pod can't reach it, `argocd cluster add` fails with:

```
error getting server version: ... dial tcp 10.x.x.x:443: i/o timeout
```

Because both clusters share a VPC (or the VPCs are peered with DNS resolution enabled), the ArgoCD pod resolves cluster B's endpoint to its **private** IP. That IP is only reachable if security-group rules permit it.

**Fix:** on cluster B's control-plane security group, add ingress **TCP 443** from cluster A's node security group.

CLI:

```bash
# Find the SG IDs:
aws eks describe-cluster --name jwong08-eks-cluster --region <REGION> \
  --query 'cluster.resourcesVpcConfig.[clusterSecurityGroupId, securityGroupIds]'
aws eks describe-cluster --name jwong08-argo-cluster --region <REGION> \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId'

# Add the rule (repeat for any additional cluster-B SGs):
aws ec2 authorize-security-group-ingress \
  --group-id <cluster-B-control-plane-SG> \
  --protocol tcp --port 443 \
  --source-group <cluster-A-node-SG>
```

Or via Console: **VPC → Security Groups → cluster B's control-plane SG → Inbound rules → Edit → Add rule:** TCP / 443 / Source = cluster A's node SG.

### 6. Cluster A (ArgoCD installed and this repo registered)

Assumed already in place:

- ArgoCD installed in the `argocd` namespace on `jwong08-argo-cluster`.
- The `argocd-initial-admin-secret` still exists in the `argocd` namespace. The workflow authenticates as `admin` using this password. If you've followed ArgoCD's post-install recommendation to delete it, either recreate the secret or switch the workflow to use a long-lived project token instead.
- This repository (`https://github.com/<org>/<repo>`) is in ArgoCD's repository list (Settings → Repositories). Public repo → no credentials needed; private → add a deploy key or PAT.

Quick sanity check:

```bash
kubectl --context jwong08-argo-cluster -n argocd get secret argocd-initial-admin-secret
```

### 7. Cluster B — API endpoint must be reachable by the runner

The GitHub Actions runner needs to reach cluster B's Kubernetes API directly (in addition to via ArgoCD, for the operator install / Secret create steps). Cluster B must therefore have **public endpoint access enabled**, or you must switch to a self-hosted runner inside the VPC.

**EKS → Cluster → Networking → Manage endpoint access**

Recommended for a lab setup:
- **Public access:** enabled
- **Private access:** enabled (keeps intra-cluster / operator traffic private)
- **Public access source allowlist:** `0.0.0.0/0` (still IAM- and RBAC-gated; safe)

Or CLI:

```bash
aws eks update-cluster-config \
  --region <REGION> \
  --name jwong08-eks-cluster \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=0.0.0.0/0
```

Cluster A needs public endpoint access too (the workflow reaches its API to port-forward `argocd-server`). Confirm with `aws eks describe-cluster --name jwong08-argo-cluster --query 'cluster.resourcesVpcConfig.endpointPublicAccess'`.

> Endpoint access changes take 5–15 minutes to fully propagate through Route 53 even after the cluster status returns to `ACTIVE`. If the first workflow run fails with `dial tcp 10.x.x.x:443` immediately after flipping the setting, wait a few minutes and retry.

### 8. CrowdStrike Falcon — API client scopes and sensor update policy

**Falcon Console → Support and resources → API clients and keys → Create API client.** The workflow's `falcon-secrets` Secret carries this client's ID + secret and hands them to the operator.

Required scopes:

| Scope | Level | Why |
| --- | --- | --- |
| **Sensor Download** | Read | Pulling the sensor image bundle from `registry.crowdstrike.com` |
| **Sensor update policies** | Read | Resolving the `linux-prod` policy referenced by the CR |
| **Falcon Images Download** | Read | Pulling Container Sensor, KAC, IAR images |
| **Hosts** | Read | Sensor grouping tags surfaced in the console |
| **Kubernetes Protection** | Read/Write | KAC (admission controller) |

Copy the generated **client ID** into GitHub secret `FALCON_CLIENT_ID` and the **client secret** into `FALCON_CLIENT_SECRET`.

**Falcon Console → Host setup and management → Sensor update policies → Linux.** The CR at `clusters/jwong08-eks-cluster/falcon-operator/manifests/falcondeployment.yaml` references `updatePolicy: linux-prod` on both the Node Sensor and Container Sensor. That policy must:

- Exist with the exact name `linux-prod` (case-sensitive)
- Be **enabled**
- Have a Linux sensor build assigned (both a general Linux sensor build for the Node Sensor and a Container Sensor build for the Container Sensor)

If the policy is missing or disabled, the child sensor reconcilers stall at `RequirementsNotMet` and the DaemonSet/injector never render.

## Running the workflows

### First-time deploy

**GitHub → Actions → "Deploy Falcon to EKS via ArgoCD" → Run workflow**

| Input | First run | Subsequent runs |
| --- | --- | --- |
| `revision` | `main` | `main` |
| `force_sync` | `true` | `true` (or `false` if you don't want to wait) |
| `install_operator` | `true` | `true` (idempotent) or `false` after operator is stable |
| `falcon_operator_version` | `latest` (or pin to a tag like `v1.5.0`) | Same as first run |
| **`create_secret`** | **`true`** | **`false`** |

The run takes ~3–5 minutes. Watch it under **Actions**; the `argocd app wait` step should end in `Synced / Healthy`.

### Verify after the run

```bash
kubectl --context jwong08-eks-cluster get falcondeployment falcon-deployment
kubectl --context jwong08-eks-cluster -n falcon-system   get ds
kubectl --context jwong08-eks-cluster -n falcon-injector get deploy
kubectl --context jwong08-eks-cluster -n falcon-kac      get deploy
kubectl --context jwong08-eks-cluster -n falcon-iar      get deploy
```

Then check the Falcon console: **Host setup and management → Host management** — filter by cluster name to see the new sensor-managed hosts appear.

## Day-to-day GitOps flow

Once bootstrap is done, changes to `manifests/falcondeployment.yaml` are automatic:

1. Edit the CR (change tags, toggle a `deploy*` flag, adjust update policy).
2. `git commit && git push` to `main`.
3. Within ~3 minutes ArgoCD polls the repo, sees the change, and applies the updated CR to cluster B.
4. The operator propagates the change to the relevant child CRs; the affected workloads roll.
5. New env vars / sensor config land in the pods within a couple of minutes; sensor grouping tags show up in the Falcon console after the next sensor check-in.

To skip the 3-minute poll for a specific change:

```bash
argocd app get falcon-operator --refresh
```

The `--sync-policy automated --auto-prune --self-heal` flags mean ArgoCD will also revert any drift introduced by `kubectl edit` on the cluster within one reconciliation cycle. **Git is the source of truth for the CR.**

Things that **do not** auto-update and still need a workflow run:

- Bumping the Falcon Operator version (`falcon_operator_version` input)
- Rotating Falcon API credentials (`create_secret: true` with new secret values in GitHub)
- Changing the ArgoCD Application spec itself (repo URL, path, destination namespace)
- Adding another target cluster

## Teardown

**GitHub → Actions → "Teardown Falcon from EKS via ArgoCD" → Run workflow**

| Input | Value |
| --- | --- |
| `confirm` | **`DELETE`** (exact case, hard guard) |
| `falcon_operator_version` | Same tag as installed (`latest` if you installed via latest) |
| `remove_secret` | `false` (or `true` to also delete `falcon-secret` namespace) |
| `unregister_cluster` | `true` (removes cluster B from ArgoCD's cluster list) |

The workflow enforces safe ordering:

1. Delete the ArgoCD Application with `--cascade=false` → immediately stops auto-sync so nothing races.
2. `kubectl delete falcondeployment --all` across all namespaces on cluster B → operator's finalizers reap child DaemonSets, Deployments, webhooks.
3. Wait up to 5 minutes for finalizers to finish.
4. `kubectl delete -f https://github.com/crowdstrike/falcon-operator/releases/download/<version>/falcon-operator.yaml` → operator itself and its CRDs are removed.
5. Optionally delete `falcon-secret` namespace.
6. `argocd cluster rm` → cluster B unregistered from ArgoCD.

Note: the SG ingress rule from §5 is **not** removed by teardown — leave it in place so redeploys don't need a Console click.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Workflow fails at `aws eks update-kubeconfig` with `AccessDeniedException` on `eks:DescribeCluster` | IAM role missing the inline policy from §3, or attached to a different role than the one being assumed | Verify with `aws iam list-role-policies --role-name <role>` and check the ARN against the error message |
| Runner errors `dial tcp 10.x.x.x:443` when talking to the target cluster's API | Cluster B endpoint is private-only, or DNS propagation is in-flight after enabling public access | Enable public access on cluster B (§7), wait 5–15 min |
| `argocd cluster add` finishes the SA/CRB steps but errors `dial tcp 10.x.x.x:443: i/o timeout` on the version-check step | Cluster A's ArgoCD pod can't reach cluster B's private endpoint | Add the SG ingress rule from §5 |
| KAC and IAR deploy but no Node Sensor DaemonSet / no Container Sensor injector | The `linux-prod` sensor update policy is missing, disabled, or the API client is missing `Sensor update policies: Read` | Fix in Falcon console (§8), then `kubectl -n falcon-operator rollout restart deploy/falcon-operator-controller-manager` |
| `FalconNodeSensor` CR shows `Pending: RequirementsNotMet` and operator logs are silent for its controller | Stuck controller work-queue after a scope change | `kubectl -n falcon-operator rollout restart deploy/falcon-operator-controller-manager` |
| `kubectl get falconnodesensor` shows empty `FALCON SENSOR` column | Sensor image / build hasn't been resolved from the policy — often missing `Sensor Download: Read` scope | Add scope in Falcon console, then restart the operator |
| Git push doesn't seem to reconcile | Change not on the branch ArgoCD tracks, or automated-sync backed off after a previous failure | `argocd app get falcon-operator --refresh && argocd app diff falcon-operator`; then `argocd app sync falcon-operator` if `OutOfSync` |
| ArgoCD sync succeeds but tags don't appear in pods | Operator hasn't propagated to the child CR yet, or the pods haven't rolled | Wait ~1 min; if stuck, restart the operator |

For any Falcon-operator-side issue the definitive log is:

```bash
kubectl --context jwong08-eks-cluster -n falcon-operator \
  logs -l control-plane=controller-manager --tail=500
```
