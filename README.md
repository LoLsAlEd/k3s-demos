---
shell: bash
terminalRows: 20
---

# Kubernetes automatic rollout demos

These small, runnable demos show what happens when configuration changes in a
Kubernetes Deployment. They start with the default behavior, then introduce a
few common ways to make configuration changes roll out safely.

To run a demo, open its `README.md` with the
[Runme extension](https://docs.runme.dev/installation/vscode/) and run the cells
from top to bottom.

If Kubernetes is new to you, these terms will help:

- A **Pod** runs one or more containers.
- A **Deployment** manages Pods and replaces them during a rollout.
- A **ConfigMap** stores non-secret configuration.
- A **Secret** stores sensitive configuration. The examples use fake values.

Start with demo 01. It shows the default Kubernetes behavior, and the later
demos build on what you learn there.

## Prerequisites

- A disposable Kubernetes cluster selected as your current `kubectl` context
- Kubernetes 1.21 or newer
- Helm 3 or 4
- Permission to create and delete namespaces
- Network access for pulling `busybox:1.36.1`
- The `helm-diff` plugin for demo 05

```sh { name=check-prerequisites }
set -eu
command -v kubectl
command -v helm
kubectl config current-context
kubectl version --client
helm version --short
```

## Demos

1. [A ConfigMap change does not trigger a rollout](01-configmap-change-does-not-rollout/README.md)
2. [Helm checksum annotations](02-helm-checksum-annotations/README.md)
3. [Versioned ConfigMaps and Secrets](03-versioned-configmaps-secrets/README.md)
4. [Gate an upgrade with a Helm hook Job](04-helm-pre-upgrade-job/README.md)
5. [Preview changes with kubectl diff and helm diff](05-kubectl-helm-diff/README.md)

Each demo uses its own namespace, so the examples do not interfere with one
another. They are independent and can be run in any order. Every README ends
with a cleanup cell that removes the resources it created.

> [!CAUTION]
> These examples create resources in the current Kubernetes context. Before you
> continue, check the context printed by the prerequisites cell so you know
> which cluster you are using.

> [!NOTE]
> The secret values in this repository are fake example values. A Kubernetes
> `Secret` is not automatically encrypted just because it uses the `Secret`
> resource kind. Use a proper secret-management solution for real credentials.

## What each demo teaches

| Demo | What you will see | Main lesson |
| --- | --- | --- |
| 01 | A ConfigMap changes but the Pod does not | Config changes do not automatically roll a Deployment |
| 02 | Helm changes a checksum and replaces the Pod | A checksum connects config changes to a rollout |
| 03 | The Deployment points to a new config name | Versioned names make each configuration distinct |
| 04 | A Job succeeds or fails before an upgrade | A Helm hook can allow or stop an upgrade |
| 05 | kubectl and Helm preview changes before applying them | A diff helps you review an update before changing the cluster |

## General documentation

- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Helm chart template guide](https://helm.sh/docs/chart_template_guide/)
