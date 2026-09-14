---
shell: bash
terminalRows: 30
---

# 05 — Preview changes with `kubectl diff` and `helm diff`

## Concept

A diff is a preview of what will change. Removed lines normally begin with `-`
and added lines begin with `+`.

- `kubectl diff` compares YAML files with the resources currently running in
   the cluster.
- `helm diff upgrade` compares an installed Helm release with a chart and values
   you are considering for the next upgrade.

Neither diff command applies the proposed changes. You still need to run
`kubectl apply` or `helm upgrade` after reviewing the preview.

## Relevant commands

```sh { ignore=true }
kubectl diff -f kubectl/app-v2.yaml

helm diff upgrade diff-demo ./helm/chart \
  --namespace demo-05-diff \
  --values helm/values-v2.yaml
```

The runnable cells use an additional exit-code option so they can check the
result:

- `kubectl diff` returns `0` when there are no changes and `1` when it finds
   changes. A value greater than `1` means an error occurred.
- `helm diff --detailed-exitcode` returns `0` for no changes and `2` when it
   finds changes.

## Expected behavior

1. Deploy the v1 Kubernetes YAML and Helm values.
2. Preview v2 and see the proposed ConfigMap, replica, and Pod-template changes.
3. Apply v2.
4. Run the same diff again and see that nothing remains to change.

## Prerequisites

`helm diff` is an external Helm plugin. If it is missing, follow the plugin's
[current installation instructions](https://github.com/databus23/helm-diff#install).

```sh { name=check-prerequisites }
set -eu
command -v kubectl
command -v helm
command -v diff
echo "Current context: $(kubectl config current-context)"

if ! helm plugin list | awk 'NR > 1 { print $1 }' | grep -qx diff; then
  echo "The helm-diff plugin is not installed." >&2
  echo "See: https://github.com/databus23/helm-diff#install" >&2
  exit 1
fi

echo "helm-diff version: $(helm diff version)"
```

## Part A: `kubectl diff`

### Deploy the v1 YAML

```sh { name=deploy-kubectl-v1 }
set -eu
kubectl create namespace demo-05-diff --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f kubectl/app-v1.yaml
kubectl rollout status deployment/kubectl-diff-demo -n demo-05-diff --timeout=90s
```

### Preview the v2 YAML

Finding differences is the expected result, but `kubectl diff` reports that as
exit code `1`. This cell handles the expected code so Runme displays success.

```sh { name=preview-with-kubectl-diff }
set -eu
set +e
output=$(kubectl diff -f kubectl/app-v2.yaml 2>&1)
diff_exit=$?
set -e

printf '%s\n' "$output"
if [ "$diff_exit" -ne 1 ]; then
  echo "Expected kubectl diff exit code 1, received $diff_exit" >&2
  exit 1
fi

echo "kubectl diff found the expected changes; nothing was applied."
```

### Apply v2 and confirm the diff is empty

```sh { name=apply-kubectl-v2 }
set -eu
kubectl apply -f kubectl/app-v2.yaml
kubectl rollout status deployment/kubectl-diff-demo -n demo-05-diff --timeout=90s

set +e
output=$(kubectl diff -f kubectl/app-v2.yaml 2>&1)
diff_exit=$?
set -e

test "$diff_exit" -eq 0
test -z "$output"
echo "kubectl diff is now empty because v2 matches the cluster."
```

## Part B: `helm diff`

### Install the Helm v1 release

```sh { name=deploy-helm-v1 }
set -eu
helm upgrade --install diff-demo ./helm/chart \
  --namespace demo-05-diff \
  --values helm/values-v1.yaml \
  --wait \
  --timeout 90s
```

### Preview the Helm v2 values

The plugin reads the installed release and renders the local chart with the v2
values. `--no-color` keeps the saved notebook output easy to read.

```sh { name=preview-with-helm-diff }
set -eu
set +e
output=$(helm diff upgrade diff-demo ./helm/chart \
  --namespace demo-05-diff \
  --values helm/values-v2.yaml \
  --detailed-exitcode \
  --no-color \
  --context 3 2>&1)
diff_exit=$?
set -e

printf '%s\n' "$output"
if [ "$diff_exit" -ne 2 ]; then
  echo "Expected helm diff exit code 2, received $diff_exit" >&2
  exit 1
fi

echo "helm diff found the expected changes; nothing was upgraded."
```

### Apply Helm v2 and confirm the diff is empty

```sh { name=apply-helm-v2 }
set -eu
helm upgrade diff-demo ./helm/chart \
  --namespace demo-05-diff \
  --values helm/values-v2.yaml \
  --wait \
  --timeout 90s

set +e
output=$(helm diff upgrade diff-demo ./helm/chart \
  --namespace demo-05-diff \
  --values helm/values-v2.yaml \
  --detailed-exitcode \
  --no-color 2>&1)
diff_exit=$?
set -e

test "$diff_exit" -eq 0
test -z "$output"
echo "helm diff is now empty because the release uses the v2 values."
```

## Good to know

- Always check which Kubernetes context and namespace you are comparing.
- A clean diff means the rendered input matches the live resource; it does not
   prove that the application is healthy.
- Helm charts can render differently based on values, chart versions, and
   cluster capabilities. Diff the same inputs you plan to upgrade with.
- Diffs may include defaulted or generated fields. Read the whole preview before
   deciding whether a change is safe.
- Helm diff hides Secret contents by default. Avoid options that reveal secrets
   in terminals, logs, or CI output.

## Learn more

- [`kubectl diff` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_diff/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [`helm-diff` documentation and examples](https://github.com/databus23/helm-diff)
- [`helm upgrade` reference](https://helm.sh/docs/helm/helm_upgrade/)

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-05-diff --ignore-not-found --wait=true
```
