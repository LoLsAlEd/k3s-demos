---
shell: bash
terminalRows: 24
---

# 03 — Versioned ConfigMaps and Secrets

## Concept

Instead of changing an existing object, create a new immutable ConfigMap or
Secret with a new name and update the Deployment to reference it. The reference
is part of `.spec.template`, so the name change triggers a rollout.

This demo shows two approaches:

1. Helm uses explicit `v1` and `v2` revision values in resource names.
2. Kustomize derives a name suffix from the generated content.

## Expected behavior

Both variants replace the Pod because the ConfigMap and Secret references in
the Pod template change. The new Pod reads v2 configuration from newly created,
immutable resources.

## Prerequisites

```sh { name=check-prerequisites }
set -eu
command -v kubectl
command -v helm
echo "Current context: $(kubectl config current-context)"
kubectl kustomize --help >/dev/null
```

## Variant A: explicit versions with Helm

### Deploy Helm v1

```sh { name=deploy-helm-v1 }
set -eu
helm upgrade --install versioned-demo ./helm/chart \
  --namespace demo-03-helm \
  --create-namespace \
  --values helm/values-v1.yaml \
  --wait \
  --timeout 90s

kubectl get configmaps,secrets -n demo-03-helm \
  -l app.kubernetes.io/instance=versioned-demo
```

### Upgrade Helm to v2

```sh { name=upgrade-helm-to-v2 }
set -eu
namespace=demo-03-helm
selector='app.kubernetes.io/instance=versioned-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')
config_before=$(kubectl get deployment versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.volumes[?(@.name=="config")].configMap.name}')
secret_before=$(kubectl get deployment versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEMO_TOKEN")].valueFrom.secretKeyRef.name}')

helm upgrade versioned-demo ./helm/chart -n "$namespace" \
  --values helm/values-v2.yaml \
  --wait \
  --timeout 90s

pod_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n "$namespace" -o jsonpath='{.metadata.uid}')
config_after=$(kubectl get deployment versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.volumes[?(@.name=="config")].configMap.name}')
secret_after=$(kubectl get deployment versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEMO_TOKEN")].valueFrom.secretKeyRef.name}')
message=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')
token=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_TOKEN | tr -d '\r')

test "$uid_before" != "$uid_after"
test "$config_before" != "$config_after"
test "$secret_before" != "$secret_after"
test "$message" = "hello from Helm v2"
test "$token" = "fake-helm-token-v2"

echo "ConfigMap: $config_before -> $config_after"
echo "Secret:    $secret_before -> $secret_after"
echo "New Pod:   $pod_after"
echo "Message:   $message"
echo "Token:     $token"
```

Helm removes the v1 resources because they are no longer in the current release
manifest. Changing the content while keeping `revision: v1` would instead try to
patch an immutable resource and fail. Treat the revision and content as one
change.

## Variant B: content hashes with Kustomize

### Deploy Kustomize v1

```sh { name=deploy-kustomize-v1 }
set -eu
kubectl create namespace demo-03-kustomize --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -k kustomize/overlays/v1
kubectl rollout status deployment/kustomize-versioned-demo -n demo-03-kustomize --timeout=90s

kubectl get configmaps,secrets -n demo-03-kustomize \
  -l app.kubernetes.io/part-of=versioned-config-demo
```

### Apply Kustomize v2

```sh { name=apply-kustomize-v2 }
set -eu
namespace=demo-03-kustomize
selector='app.kubernetes.io/name=kustomize-versioned-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')
config_before=$(kubectl get deployment kustomize-versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.volumes[?(@.name=="config")].configMap.name}')
secret_before=$(kubectl get deployment kustomize-versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEMO_TOKEN")].valueFrom.secretKeyRef.name}')

kubectl apply -k kustomize/overlays/v2
kubectl rollout status deployment/kustomize-versioned-demo -n "$namespace" --timeout=90s

pod_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n "$namespace" -o jsonpath='{.metadata.uid}')
config_after=$(kubectl get deployment kustomize-versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.volumes[?(@.name=="config")].configMap.name}')
secret_after=$(kubectl get deployment kustomize-versioned-demo -n "$namespace" -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="DEMO_TOKEN")].valueFrom.secretKeyRef.name}')
message=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')
token=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_TOKEN | tr -d '\r')

test "$uid_before" != "$uid_after"
test "$config_before" != "$config_after"
test "$secret_before" != "$secret_after"
test "$message" = "hello from Kustomize v2"
test "$token" = "fake-kustomize-token-v2"

echo "ConfigMap: $config_before -> $config_after"
echo "Secret:    $secret_before -> $secret_after"
echo "New Pod:   $pod_after"
echo "Message:   $message"
echo "Token:     $token"
```

Kustomize creates the v2 objects but plain `kubectl apply -k` does not delete
the old generated objects. Keeping the previous revision briefly makes rollback
possible, but production workflows need an explicit, rollout-aware retention or
pruning policy.

## Production notes

- Immutable objects prevent accidental in-place changes and reduce API-server
   watches at scale.
- Versioned names make rollbacks explicit: roll the Pod template back to an old
   resource name.
- Retain old versions until no running Pod references them, then prune according
   to a defined policy.
- Do not put real secret material in values files or Kustomize literals committed
   to source control.

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-03-helm demo-03-kustomize --ignore-not-found --wait=true
```
