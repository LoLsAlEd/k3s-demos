---
shell: bash
terminalRows: 24
---

# 04 — Check an upgrade with a Helm pre-upgrade Job

## What this demo shows

A Kubernetes Job runs a task until it succeeds or fails. Helm hooks let a chart
run a Job at a particular point during an install or upgrade.

This demo uses a `pre-upgrade` hook as a checkpoint. Helm waits for the Job
before it updates the application. If the Job succeeds, the upgrade continues.
If it fails, Helm stops the upgrade.

This is similar to running a database migration, but it does not need a
database. After the checkpoint succeeds, a checksum annotation tells
Kubernetes to roll out the new application configuration.

## The hook annotations

These annotations tell Helm to treat the Kubernetes Job as a hook:

```yaml { ignore=true }
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    helm.sh/hook: pre-upgrade
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation
spec:
  backoffLimit: 0
```

- `pre-upgrade` runs the Job before Helm changes the application.
- `before-hook-creation` removes the previous hook Job before the next attempt.
- `backoffLimit: 0` makes this demo report a failure without retrying.

## What to expect

1. The initial install creates v1. The `pre-upgrade` hook does not run during
   install.
2. The v2 migration Job succeeds, and then the v2 configuration rolls out.
3. The v3 migration Job fails, so the ConfigMap, Deployment, and Pod stay at
   v2.

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

## Try a successful upgrade

The completed Job is kept long enough for you to read its logs. The next
upgrade removes it before creating another Job with the same name.

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

## Try an upgrade that fails

The Helm command is expected to fail. The cell captures that expected error so
Runme can still report the demonstration as successful.

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

## A few useful details

- Hook Jobs are not managed like ordinary chart resources. Give them a cleanup
  policy or a Job TTL.
- `before-hook-creation` keeps the latest Job available for debugging, then
  removes it before the next attempt. This demo also uses a ten-minute Job TTL.
- A real migration should be safe to run again if someone retries an upgrade.
- Helm can stop an upgrade, but it cannot undo changes that a Job already made
  to an external database or service.

## Learn more

- [Helm chart hooks](https://helm.sh/docs/topics/charts_hooks/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Deployment rolling updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-04-hook-job --ignore-not-found --wait=true
```
