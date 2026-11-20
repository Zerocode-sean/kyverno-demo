# Kyverno Demo

A small Kubernetes configuration repository demonstrating how [Kyverno](https://kyverno.io/) policy-as-code can validate workload definitions and enforce organizational standards.

## What is Kyverno?

Kyverno is a Kubernetes-native policy engine that uses Kubernetes custom resources rather than a separate policy language. It evaluates resources against policies during admission, allowing clusters to:

- **Validate** that resources meet required configuration standards.
- **Mutate** resources by changing fields before they are stored.
- **Generate** supporting resources when a resource is created.
- **Verify** or report compliance with existing cluster resources.

Kyverno integrates with the Kubernetes admission controller. Policies can block non-compliant resources or report violations depending on the configured action.

## Repository contents

| File | Resource | Purpose |
| --- | --- | --- |
| `deployment.yaml` | `Deployment` | Defines a sample nginx deployment with one replica. |
| `label-policy.yaml` | `ValidatingPolicy` | Requires deployments to include a `team` label. |
| `replica-policy.yaml` | `ValidatingPolicy` | Requires deployments to specify at least three replicas. |
| `no-latest-tag-policy.yaml` | `ValidatingPolicy` | Rejects Pods that use images with the `latest` tag. |
| `private-registry.yaml` | `ValidatingPolicy` | Restricts Pod images to registries matching `kodekloud.io/*`. |

## How Kyverno is used here

This repository uses CEL-based `ValidatingPolicy` resources under `policies.kyverno.io/*`.

The policies use deny mode (`validationActions: [Deny]`). A resource that violates a policy is rejected by the Kubernetes API server.

### Team label

`label-policy.yaml` requires every deployment to contain a `team` label with a non-empty value.

```yaml
metadata:
  labels:
    team: platform
```

### Minimum replicas

`replica-policy.yaml` requires deployments to set `spec.replicas` to `3` or greater.

### Container image tags

`no-latest-tag-policy.yaml` rejects Pods whose container image uses the `latest` tag. Using an explicit, immutable version or digest is recommended:

```yaml
image: nginx:1.27.4
```

### Private registry

`private-registry.yaml` permits Pod images only from the `kodekloud.io` registry. Replace `kodekloud.io/*` with an organization-specific registry when adapting this policy for a real environment.

## Prerequisites

- A Kubernetes cluster
- `kubectl` configured for the cluster
- Kyverno installed in the cluster

Verify Kyverno is running before applying the policies:

```bash
kubectl get pods -n kyverno
```

## Apply the policies

Apply all policies from this repository:

```bash
kubectl apply -f label-policy.yaml
kubectl apply -f replica-policy.yaml
kubectl apply -f no-latest-tag-policy.yaml
kubectl apply -f private-registry.yaml
```

Confirm that the policies are registered:

```bash
kubectl get validatingpolicy
```

## KodeKloud lab Git sync workflow

If you edit locally and run the cluster from a KodeKloud control node clone, sync the lab checkout before reapplying manifests.

Push local changes:

```bash
git add .
git commit -m "Migrate remaining policies to ValidatingPolicy"
git push origin main
```

On the KodeKloud control node:

```bash
cd ~/kyverno-demo
git fetch origin
git pull --ff-only origin main
kubectl apply -f .
```

If `git pull` fails due to local lab edits, either stash and pull, or reclone to match remote exactly.

## Test the sample deployment

Apply the sample workload:

```bash
kubectl apply -f deployment.yaml
```

With these deny-mode policies, the sample deployment may be rejected because it:

- Uses a deployment with only one replica.
- Does not include the required `team` label.
- Uses a public `nginx` image instead of an image under `kodekloud.io/*`.

To test policy enforcement, update the deployment to satisfy the example policies. For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
    team: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: container1
          image: kodekloud.io/nginx:1.19.0
```

> Image availability: the example private-registry policy permits a registry pattern, but the named image must exist and be accessible to the cluster. Replace it with an image available in your organization’s registry.

## Quick negative policy tests

Use `policy-negative-tests.yaml` to intentionally trigger policy denials.

```bash
kubectl apply -f policy-negative-tests.yaml
```

Expected behavior:

- `Deployment/fail-team-and-replicas` is denied by the team-label and minimum-replicas policies.
- `Pod/fail-latest-and-registry` is denied by the no-latest-tag and private-registry policies.

If resources already exist from a previous run, delete and retry:

```bash
kubectl delete -f policy-negative-tests.yaml --ignore-not-found
kubectl apply -f policy-negative-tests.yaml
```

## Inspect policy results

List installed policies:

```bash
kubectl get validatingpolicy
```

Describe a policy for rule and status details:

```bash
kubectl describe validatingpolicy require-team-label
kubectl describe validatingpolicy minimum-replicas
kubectl describe validatingpolicy no-latest-tag-policy
kubectl describe validatingpolicy deny-Public-Registry-Images
```

View recent PolicyReports if reporting resources are available:

```bash
kubectl get policyreport -A
kubectl get clusterpolicyreport
```

## YAML manifest notes

Kyverno policies should be validated against the version installed in the cluster. For a production environment:

- Use an immutable image digest or a pinned image tag.
- Prefer specific private registry names instead of a broad pattern.
- Scope policies with `namespaces` or `match.exclude` when rules must not apply cluster-wide.
- Run policies in `Audit` mode while developing or testing, then switch to `Deny`.
- Keep policy tests and exception documentation alongside the manifests.

## Cleanup

Remove the demo policies:

```bash
kubectl delete -f private-registry.yaml
kubectl delete -f no-latest-tag-policy.yaml
kubectl delete -f replica-policy.yaml
kubectl delete -f label-policy.yaml
```

Remove the sample deployment:

```bash
kubectl delete -f deployment.yaml
```
