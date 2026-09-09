---
shell: bash
terminalRows: 20
---

# Kubernetes automatic rolling deployment demos

Small, executable demonstrations of how configuration changes interact with
Kubernetes Deployments. Open any demo `README.md` with the
[Runme extension](https://docs.runme.dev/installation/vscode/) and run its cells
from top to bottom.

## Prerequisites

- A disposable Kubernetes cluster selected as the current `kubectl` context
- Kubernetes 1.21 or newer
- Helm 3 or 4
- Permission to create and delete namespaces
- Network access for pulling `busybox:1.36.1`

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

The demos are independent and use separate namespaces. Every README ends with
a cleanup cell, so you can run the examples in any order.

> [!CAUTION]
> These examples create resources in the current Kubernetes context. Check the
> context printed by the prerequisites cell before continuing.

> [!NOTE]
> Secret values in this repository are deliberately fake, committed fixtures.
> Kubernetes Secrets are not encrypted merely because they use the `Secret`
> resource kind. Use a real secret-management workflow in production.

## What the sequence teaches

| Demo | Rollout trigger | Main lesson |
| --- | --- | --- |
| 01 | Manual change under `.spec.template` | A referenced ConfigMap is outside the Pod template |
| 02 | A rendered checksum annotation changes | Helm can connect config content to the Pod template |
| 03 | A referenced resource name changes | Immutable/versioned config gives every revision a distinct identity |
| 04 | Config checksum after a successful hook | A pre-upgrade Job can gate, but not transactionally wrap, a rollout |

