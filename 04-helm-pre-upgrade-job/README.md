---
shell: bash
terminalRows: 24
---

# 04 — Gate an upgrade with a Helm hook Job

## Concept

A `pre-upgrade` hook runs after Helm renders and validates the chart but before
Helm updates the release resources. Helm waits for a hook Job to complete. If the
Job fails, the upgrade fails and Helm does not apply the new Deployment or
ConfigMap.

This models a database migration without requiring a database. The application
still uses a checksum annotation to trigger its actual rollout after the gate
succeeds.

See Helm's [chart hook lifecycle](https://helm.sh/docs/topics/charts_hooks/).

## Expected behavior

1. The initial install creates v1; `pre-upgrade` does not run during install.
2. The v2 migration Job succeeds, then the v2 configuration rolls out.
3. The v3 migration Job fails, so the running ConfigMap, Deployment, and Pod
   remain at v2.

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
helm upgrade --install hook-demo ./chart \
  --namespace demo-04-hook-job \
  --create-namespace \
  --values values-v1.yaml \
  --wait \
  --timeout 90s

pod=$(kubectl get pods -n demo-04-hook-job \
  -l app.kubernetes.io/name=hook-job-demo \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')
kubectl exec -n demo-04-hook-job "$pod" -- printenv DEMO_MESSAGE
```

## Run a successful upgrade

The completed hook is intentionally retained long enough to inspect its logs.
The next upgrade removes it before creating a new hook with the same name.

```sh { name=successful-pre-upgrade-hook }
set -eu
namespace=demo-04-hook-job
selector='app.kubernetes.io/name=hook-job-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')

helm upgrade hook-demo ./chart -n "$namespace" \
  --values values-v2.yaml \
  --wait \
  --timeout 90s

kubectl logs job/hook-demo-migrate -n "$namespace"
pod_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n "$namespace" -o jsonpath='{.metadata.uid}')
message=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')

test "$uid_before" != "$uid_after"
test "$message" = "hello from v2"

echo "The hook succeeded before Helm rolled the application."
echo "New Pod: $pod_after"
echo "Message: $message"
```

## Run an intentionally failing upgrade

The Helm command must fail for this assertion to pass. The cell captures that
expected error so Runme reports the demonstration itself as successful.

```sh { name=failing-hook-blocks-upgrade }
set -eu
namespace=demo-04-hook-job
selector='app.kubernetes.io/name=hook-job-demo'
uid_before=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')

set +e
output=$(helm upgrade hook-demo ./chart -n "$namespace" \
  --values values-v3-fail.yaml \
  --wait \
  --timeout 45s 2>&1)
status=$?
set -e

echo "$output"
if [ "$status" -eq 0 ]; then
  echo "Expected the v3 upgrade to fail, but it succeeded" >&2
  exit 1
fi

kubectl logs job/hook-demo-migrate -n "$namespace"
pod_after=$(kubectl get pods -n "$namespace" -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n "$namespace" -o jsonpath='{.metadata.uid}')
message_in_configmap=$(kubectl get configmap hook-demo-config -n "$namespace" -o jsonpath='{.data.message}')
message_in_pod=$(kubectl exec -n "$namespace" "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')

test "$uid_before" = "$uid_after"
test "$message_in_configmap" = "hello from v2"
test "$message_in_pod" = "hello from v2"

echo "Helm status:        $(helm status hook-demo -n "$namespace" -o json | sed -n 's/.*"status":"\([^"]*\)".*/\1/p')"
echo "Unchanged Pod UID:  $uid_after"
echo "ConfigMap message:  $message_in_configmap"
echo "Pod message:        $message_in_pod"
echo "The failed pre-upgrade hook blocked v3 as expected."
```

## Production notes

- Hook Jobs are separate lifecycle resources, not ordinary release resources.
   Define an explicit deletion or TTL policy.
- `before-hook-creation` preserves the latest Job for debugging and removes it
   before the next attempt. This demo also uses a ten-minute Job TTL.
- Keep migrations backward-compatible with both the old and new application
   versions during a rolling update.
- Helm does not transactionally undo external side effects performed by a hook.
   Design migrations to be idempotent and recoverable.
- Use `--atomic` only with a clear understanding of what Helm can roll back; it
   cannot reverse an arbitrary database migration.

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-04-hook-job --ignore-not-found --wait=true
```
