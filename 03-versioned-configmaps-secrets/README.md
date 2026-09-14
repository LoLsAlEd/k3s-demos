---
shell: bash
terminalRows: 24
---

# 03 — Give each ConfigMap and Secret its own version

## What this demo shows

In this pattern, every configuration version gets a different name. For
example, `app-config-v1` becomes `app-config-v2`. The Deployment then points to
the new name, and that change causes Kubernetes to replace the Pod.

The ConfigMaps and Secrets are marked immutable. That means Kubernetes rejects
attempts to edit them in place. To change the configuration, create a new
version instead.

This demo shows two ways to do that:

1. Helm uses an explicit `v1` or `v2` revision in each resource name.
2. Kustomize creates a name suffix from the generated content.

## The important YAML

The Helm chart puts the version in the resource name and in the Deployment's
reference to that resource:

```yaml { ignore=true }
# ConfigMap
metadata:
  name: {{ .Release.Name }}-config-{{ .Values.config.revision }}
immutable: true

# Deployment volume reference
configMap:
  name: {{ .Release.Name }}-config-{{ .Values.config.revision }}
```

Kustomize generates the version for you. By default, its name suffix is based
on the generated content:

```yaml { ignore=true }
configMapGenerator:
  - name: demo-config
    literals:
      - message=hello from Kustomize v2
secretGenerator:
  - name: demo-secret
    literals:
      - token=fake-kustomize-token-v2
generatorOptions:
  immutable: true
```

## What to expect

Both variants replace the Pod because the ConfigMap and Secret references in
the Pod template change. The new Pod reads the v2 configuration from newly
created, immutable resources.

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

Helm removes the v1 resources after the upgrade because the current release no
longer includes them. Remember to change the revision whenever you change the
content. Otherwise Helm tries to edit an immutable object and the upgrade
fails.

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

Kustomize creates the v2 objects but does not delete the v1 objects in this
demo. Keeping an old version can make rollback easier, but clean up old
versions once no Pods use them.

## A few useful details

- Immutable objects protect a version from accidental changes.
- To roll back, point the Deployment back to an earlier resource name.
- Do not delete an old version while a running Pod still refers to it.
- Do not put real secret material in values files or Kustomize literals that are
  committed to source control.

## Learn more

- [Kubernetes immutable ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/#immutable-configmaps)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kustomize generators](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#generating-resources)
- [Helm values files](https://helm.sh/docs/chart_template_guide/values_files/)

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-03-helm demo-03-kustomize --ignore-not-found --wait=true
```
