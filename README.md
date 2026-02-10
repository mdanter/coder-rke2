# Deploying Coder Workspaces to a Separate Namespace on RKE2

This guide walks through configuring Coder to deploy Kubernetes-based workspaces in a `coder-workspaces` namespace, separate from the `coder` control plane namespace, on an RKE2 cluster.

## Version requirements

The `workspaceNamespaces` Helm value was introduced in Coder v2.27.0 ([PR #19517](https://github.com/coder/coder/pull/19517)). A bugfix for RBAC rendering when `workspacePerms=false` was included in v2.27.3 ([PR #20569](https://github.com/coder/coder/pull/20569)).

**Minimum recommended version: v2.27.3 or later.**

At the time of writing, the latest releases are:
- Mainline: v2.30.1
- Stable: v2.29.6

## Prerequisites

- Coder installed via the `coder/coder` Helm chart in the `coder` namespace
- `kubectl` access to the RKE2 cluster
- `helm` 3.5+

## Step 1: Create the workspace namespace

```bash
kubectl create namespace coder-workspaces
```

## Step 2: Label the namespace for Pod Security Admission

RKE2 enforces Pod Security Admission (PSA) by default. Label the workspace namespace to allow workspace pods to run:

```bash
kubectl label namespace coder-workspaces \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline
```

If your workspaces require elevated privileges (e.g., Docker-in-Docker via `envbox`), use `privileged` instead of `baseline`:

```bash
kubectl label namespace coder-workspaces \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged
```

## Step 3: Update Coder Helm values to grant RBAC in the workspace namespace

Add `workspaceNamespaces` to your existing `values.yaml`:

```yaml
coder:
  serviceAccount:
    workspacePerms: true
    enableDeployments: true
    workspaceNamespaces:
      - name: coder-workspaces
```

This tells the Helm chart to create a Role and RoleBinding in `coder-workspaces` that grant the `coder` service account (in the `coder` namespace) permission to manage pods, PVCs, and deployments there.

Apply the change:

```bash
helm repo update
helm upgrade coder coder-v2/coder \
  --namespace coder \
  -f values.yaml
```

## Step 4: Verify the RBAC resources were created

```bash
kubectl get role -n coder-workspaces
kubectl get rolebinding -n coder-workspaces
```

You should see a Role named `coder-workspace-perms` and a RoleBinding named `coder` in the `coder-workspaces` namespace.

Verify the role has the expected permissions:

```bash
kubectl describe role coder-workspace-perms -n coder-workspaces
```

The output should show rules for `pods`, `persistentvolumeclaims`, and `apps/deployments` with create, delete, deletecollection, get, list, patch, update, and watch verbs.

Verify the RoleBinding references the correct service account:

```bash
kubectl describe rolebinding coder -n coder-workspaces
```

The subjects should list the `coder` ServiceAccount in the `coder` namespace.

## Step 5: (Optional) Check Rancher Project constraints

If the `coder-workspaces` namespace is assigned to a Rancher Project, verify the following:

**Resource Quotas** — Ensure project-level quotas allow the CPU, memory, and storage your workspaces will request.

**Network Policies** — If the project applies default-deny NetworkPolicies, add a policy allowing workspace pods to reach the Coder control plane. At minimum, workspace pods need egress to the Coder server for agent connectivity. Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-workspace-to-coder
  namespace: coder-workspaces
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: coder
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 53
        - protocol: UDP
          port: 53
```

Adjust this based on your network requirements.

## Step 6: Configure your Coder template

In your Kubernetes Terraform template, set the `namespace` variable to `coder-workspaces`. When pushing the template:

```bash
coder templates push my-kubernetes-template \
  --var namespace=coder-workspaces
```

Or set the default in your template's `variables.tf`:

```hcl
variable "namespace" {
  type        = string
  description = "The Kubernetes namespace to create workspaces in."
  default     = "coder-workspaces"
}
```

All resources in the template (`kubernetes_deployment_v1`, `kubernetes_persistent_volume_claim_v1`) reference `namespace = var.namespace`, so this is the only value that needs to change.

## Step 7: Test a workspace build

1. Create a workspace using the updated template.
2. Verify the workspace resources are running in the correct namespace:

```bash
kubectl get pods -n coder-workspaces
kubectl get deployments -n coder-workspaces
kubectl get pvc -n coder-workspaces
```

3. Confirm the workspace agent connects successfully in the Coder dashboard.

## Troubleshooting

**`pods is forbidden` or `deployments.apps is forbidden` in build logs** — The RBAC was not applied correctly. Re-run Step 3 and verify Step 4.

**Pod rejected by admission controller** — The PSA label on the namespace is too restrictive for your workspace pod spec. Re-run Step 2 with the appropriate level.

**Workspace agent never connects** — A NetworkPolicy may be blocking egress from the workspace namespace. Check Step 5.

**PVC stuck in `Pending`** — Either no StorageClass is available, or a project-level resource quota has been exceeded. Verify with `kubectl get sc` and check Rancher project quotas.

## References

- Helm chart `values.yaml`: https://github.com/coder/coder/blob/main/helm/coder/values.yaml
- Feature PR: https://github.com/coder/coder/pull/19517
- RBAC bugfix PR: https://github.com/coder/coder/pull/20569
- Coder Rancher install docs: https://coder.com/docs/install/rancher
- Coder Kubernetes install docs: https://coder.com/docs/install/kubernetes
