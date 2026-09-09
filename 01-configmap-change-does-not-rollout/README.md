---
shell: bash
terminalRows: 20
---

# 01 — A ConfigMap change does not trigger a rollout

## Concept

A rollout means that a Deployment replaces its Pods with new Pods. Kubernetes
starts a rollout when the Pod template inside the Deployment changes.

A ConfigMap is stored separately from the Deployment. Changing the ConfigMap
does not change the Pod template, so Kubernetes keeps the existing Pod running.

This Pod consumes the same ConfigMap in two ways:

- `DEMO_MESSAGE` is read when the container starts. It stays at v1 until the Pod
   is replaced.
- `/etc/demo/message` is a mounted file. Kubernetes updates this file in the
   existing Pod after a short delay.

## Relevant YAML

The Deployment refers to the ConfigMap, but the ConfigMap data is not copied
into the Pod template. Later in the demo, adding the annotation shown below
changes the template and causes a rollout.

```yaml { ignore=true }
spec:
  template:
    metadata:
      annotations:
        demo.kubernetes.io/config-revision: v2 # This template change rolls Pods
    spec:
      containers:
        - env:
            - name: DEMO_MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: demo-config
                  key: message
      volumes:
        - name: config
          configMap:
            name: demo-config
```

## Expected behavior

1. Applying ConfigMap v2 leaves the Pod UID and Deployment revision unchanged.
2. The environment variable stays at v1, while the mounted file becomes v2.
3. Changing a Pod-template annotation creates a new ReplicaSet and Pod.
4. The replacement Pod starts with v2 in both locations.

## Prerequisites

```sh { name=check-prerequisites }
set -eu
command -v kubectl
echo "Current context: $(kubectl config current-context)"
kubectl cluster-info
```

## Deploy v1

```sh { name=deploy-v1 }
set -eu
kubectl create namespace demo-01-configmap --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f configmap-v1.yaml -f deployment.yaml
kubectl rollout status deployment/configmap-rollout-demo -n demo-01-configmap --timeout=90s
```

## Observe v1

```sh { name=observe-v1 }
set -eu
pod=$(kubectl get pods -n demo-01-configmap -l app.kubernetes.io/name=configmap-rollout-demo --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid=$(kubectl get pod "$pod" -n demo-01-configmap -o jsonpath='{.metadata.uid}')
revision=$(kubectl get deployment configmap-rollout-demo -n demo-01-configmap -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}')

echo "Pod:      $pod"
echo "UID:      $uid"
echo "Revision: $revision"
kubectl exec -n demo-01-configmap "$pod" -- sh -c 'echo "Environment: $DEMO_MESSAGE"; echo "Volume:      $(cat /etc/demo/message)"'
```

## Modify only the ConfigMap

This cell waits for the mounted file to update, then checks that Kubernetes kept
the same Pod and Deployment revision.

```sh { name=update-configmap-without-rollout }
set -eu
selector='app.kubernetes.io/name=configmap-rollout-demo'
pod_before=$(kubectl get pods -n demo-01-configmap -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_before=$(kubectl get pod "$pod_before" -n demo-01-configmap -o jsonpath='{.metadata.uid}')
revision_before=$(kubectl get deployment configmap-rollout-demo -n demo-01-configmap -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}')

kubectl apply -f configmap-v2.yaml

attempt=0
until [ "$(kubectl exec -n demo-01-configmap "$pod_before" -- cat /etc/demo/message)" = "hello from v2" ]; do
  attempt=$((attempt + 1))
  if [ "$attempt" -ge 30 ]; then
    echo "Timed out waiting for the ConfigMap volume projection" >&2
    exit 1
  fi
  sleep 2
done

pod_after=$(kubectl get pods -n demo-01-configmap -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n demo-01-configmap -o jsonpath='{.metadata.uid}')
revision_after=$(kubectl get deployment configmap-rollout-demo -n demo-01-configmap -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}')
environment=$(kubectl exec -n demo-01-configmap "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')
volume=$(kubectl exec -n demo-01-configmap "$pod_after" -- cat /etc/demo/message)

test "$uid_before" = "$uid_after"
test "$revision_before" = "$revision_after"
test "$environment" = "hello from v1"
test "$volume" = "hello from v2"

echo "Same Pod UID:       $uid_after"
echo "Same revision:      $revision_after"
echo "Stale environment:  $environment"
echo "Updated volume:     $volume"
```

## Change the Pod template

The application does not use this annotation. It triggers a rollout simply
because it changes the Deployment's Pod template.

```sh { name=trigger-rollout-with-template-annotation }
set -eu
selector='app.kubernetes.io/name=configmap-rollout-demo'
uid_before=$(kubectl get pods -n demo-01-configmap -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.uid}')

kubectl patch deployment configmap-rollout-demo -n demo-01-configmap --type=merge \
  -p '{"spec":{"template":{"metadata":{"annotations":{"demo.kubernetes.io/config-revision":"v2"}}}}}'
kubectl rollout status deployment/configmap-rollout-demo -n demo-01-configmap --timeout=90s

pod_after=$(kubectl get pods -n demo-01-configmap -l "$selector" --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
uid_after=$(kubectl get pod "$pod_after" -n demo-01-configmap -o jsonpath='{.metadata.uid}')
revision_after=$(kubectl get deployment configmap-rollout-demo -n demo-01-configmap -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}')
environment=$(kubectl exec -n demo-01-configmap "$pod_after" -- printenv DEMO_MESSAGE | tr -d '\r')
volume=$(kubectl exec -n demo-01-configmap "$pod_after" -- cat /etc/demo/message)

test "$uid_before" != "$uid_after"
test "$environment" = "hello from v2"
test "$volume" = "hello from v2"

echo "New Pod:         $pod_after"
echo "New Pod UID:     $uid_after"
echo "New revision:    $revision_after"
echo "Environment:     $environment"
echo "Volume:          $volume"
```

## Good to know

- A mounted ConfigMap can take a short time to update.
- ConfigMaps mounted with `subPath` do not receive projected updates.
- An application may need its own reload feature before it notices a changed
   file.
- ConfigMap-backed environment variables update only when a new container
   starts.

## Learn more

- [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- [Updating configuration using a ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/)
- [Mounted ConfigMaps and update behavior](https://kubernetes.io/docs/concepts/configuration/configmap/#mounted-configmaps-are-updated-automatically)

## Cleanup

```sh { name=cleanup }
kubectl delete namespace demo-01-configmap --ignore-not-found --wait=true
```
