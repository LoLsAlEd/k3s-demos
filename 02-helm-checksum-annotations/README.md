---
shell: bash
terminalRows: 20
---

# 02 — Helm checksum annotations

## Concept

A checksum is a fingerprint of some content. Helm calculates one fingerprint
for the ConfigMap and another for the Secret, then puts them on the Deployment's
Pod template.

When configuration changes, its fingerprint changes. That changes the Pod
template, so Kubernetes replaces the Pod. When the configuration is unchanged,
the fingerprints stay the same and the Pod is left alone.

## Relevant Helm template

These two lines in `chart/templates/deployment.yaml` connect configuration
changes to the Pod template:

```yaml { ignore=true }
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum | quote }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum | quote }}
```

`include` renders the file and `sha256sum` creates the fingerprint.

## Expected behavior

1. Installing v1 creates one Pod with v1 configuration.
2. Upgrading with the same values changes the Helm release revision, but not the
   Pod template or Pod UID.
3. Upgrading to v2 changes both checksums and replaces the Pod.

## Prerequisites

```sh { name=check-prerequisites }
set -eu
command -v kubectl
command -v helm
echo "Current context: $(kubectl config current-context)"
helm version --short
```

## Deploy v1

```sh { name=deploy-v1 }
set -eu
helm upgrade --install checksum-demo ./chart \
  --namespace demo-02-checksum \
  --create-namespace \
  --values values-v1.yaml \
  --wait \
  --timeout 90s
```

## Observe v1

```sh { name=observe-v1 }
set -eu
namespace=demo-02-checksum
selector='app.kubernetes.io/instance=checksum-demo'
pod=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')

echo "Pod: $pod"
kubectl get deployment checksum-demo -n "$namespace" \
  -o jsonpath='Config checksum: {.spec.template.metadata.annotations.checksum/config}{"\n"}Secret checksum: {.spec.template.metadata.annotations.checksum/secret}{"\n"}'
kubectl exec -n "$namespace" "$pod" -- sh -c 'echo "Message: $DEMO_MESSAGE"; echo "Token:   $DEMO_TOKEN"; echo "Volume:  $(cat /etc/demo/message)"'
```

## Perform a no-op upgrade

This cell checks that running Helm again with the same values keeps the same
Pod.

```sh { name=no-op-upgrade-does-not-roll }
set -eu
namespace=demo-02-checksum
selector='app.kubernetes.io/instance=checksum-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')
checksums_before=$(kubectl get deployment checksum-demo -n "$namespace" -o jsonpath='{.spec.template.metadata.annotations.checksum/config}:{.spec.template.metadata.annotations.checksum/secret}')

helm upgrade checksum-demo ./chart -n "$namespace" --values values-v1.yaml --wait --timeout 90s

uid_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')
checksums_after=$(kubectl get deployment checksum-demo -n "$namespace" -o jsonpath='{.spec.template.metadata.annotations.checksum/config}:{.spec.template.metadata.annotations.checksum/secret}')

test "$uid_before" = "$uid_after"
test "$checksums_before" = "$checksums_after"
echo "Pod UID stayed the same: $uid_after"
echo "Checksums stayed the same: $checksums_after"
helm list -n "$namespace"
```

## Upgrade to v2

```sh { name=upgrade-config-and-roll }
set -eu
namespace=demo-02-checksum
selector='app.kubernetes.io/instance=checksum-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')
checksums_before=$(kubectl get deployment checksum-demo -n "$namespace" -o jsonpath='{.spec.template.metadata.annotations.checksum/config}:{.spec.template.metadata.annotations.checksum/secret}')

helm upgrade checksum-demo ./chart -n "$namespace" --values values-v2.yaml --wait --timeout 90s
kubectl rollout status deployment/checksum-demo -n "$namespace" --timeout=90s

pod_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n "$namespace" -o jsonpath='{.metadata.uid}')
checksums_after=$(kubectl get deployment checksum-demo -n "$namespace" -o jsonpath='{.spec.template.metadata.annotations.checksum/config}:{.spec.template.metadata.annotations.checksum/secret}')
message=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')
token=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_TOKEN | tr -d '\r')

test "$uid_before" != "$uid_after"
test "$checksums_before" != "$checksums_after"
test "$message" = "hello from v2"
test "$token" = "fake-demo-token-v2"

echo "New Pod:       $pod_after"
echo "New Pod UID:   $uid_after"
echo "New checksums: $checksums_after"
echo "Message:       $message"
echo "Token:         $token"
```

## Good to know

- Hash the rendered resource so every relevant change affects the checksum.
- Use separate annotations for configuration and secrets to make changes easier
  to see.
- Kubernetes still uses the Deployment's normal rolling-update settings.
- Never print real secret values as this demo does for illustration.

## Learn more

- [Helm: automatically roll Deployments](https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments)
- [Helm template functions](https://helm.sh/docs/chart_template_guide/function_list/)
- [Kubernetes Deployment updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-02-checksum --ignore-not-found --wait=true
```
