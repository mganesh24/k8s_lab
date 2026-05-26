# Healthy Pods — Solutions

Each answer has the command to run and a one or two line explanation. If you're following along without doing the tasks first, that's fine — read the question, run the command, read the explanation, move on.

> **Before you start:** make sure you applied the three manifests:
> ```bash
> kubectl apply -f manifests/alpha.yaml
> kubectl apply -f manifests/bravo.yaml
> kubectl apply -f manifests/charlie.yaml
> ```

---

## Pod: `alpha`

### 1. Imperative or declarative?

**Declarative.** You wrote a YAML file describing the desired state and applied it with `kubectl apply -f`. Imperative would be `kubectl run alpha --image=nginx` — no file, no source of truth.

### 2. Which node is the pod running on?

```bash
kubectl get pod alpha -o wide
```

Look at the `NODE` column. It will be one of your two workers — the control plane is normally tainted to keep workloads off it.

### 3. What image is the container using?

```bash
kubectl describe pod alpha | grep -i image:
```

Or more precisely:

```bash
kubectl get pod alpha -o jsonpath='{.spec.containers[*].image}'
```

`nginx:1.27-alpine`. The image is what the container runtime pulls and runs.

### 4. What is the pod IP address?

```bash
kubectl get pod alpha -o wide
```

The `IP` column. This is a **cluster-internal** IP from the pod CIDR — only reachable from inside the cluster, not from your laptop directly.

### 5. Can the pod reach the internet?

```bash
kubectl exec -it alpha -- sh -c "wget -qO- -T 3 https://www.google.com | head -c 100"
```

If you get HTML back, yes. Outbound internet works by default in most clusters as long as the nodes themselves have egress.

### 6. Is the pod accessible from the external world?

Yes — via a `NodePort` Service. Find any node's IP:

```bash
kubectl get nodes -o wide
```

Then from your laptop:

```bash
curl http://<any-node-ip>:30080
```

You should see the nginx welcome page. The pod itself doesn't have a public IP — the NodePort Service exposes it on every node at port `30080`.

### 7. When was the pod created?

```bash
kubectl get pod alpha
```

The `AGE` column. For a precise timestamp:

```bash
kubectl get pod alpha -o jsonpath='{.metadata.creationTimestamp}'
```

### 8. What phase is the pod currently in?

```bash
kubectl get pod alpha
```

The `STATUS` column. For a healthy pod it's `Running`. The five pod phases are `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`.

### 9. Which resource owns this pod?

```bash
kubectl get pod alpha -o jsonpath='{.metadata.ownerReferences}'
```

Empty. **No owner** — `alpha` is a bare/standalone pod. It is not managed by a Deployment, ReplicaSet, Job, or StatefulSet.

### 10. What happens if you delete the pod manually?

```bash
kubectl delete pod alpha
kubectl get pods
```

It's gone for good. With no controller to recreate it, deletion is permanent. Re-apply the manifest to bring it back.

> After testing, re-apply:
> ```bash
> kubectl apply -f manifests/alpha.yaml
> ```

### 11. What is the pod's restart policy?

```bash
kubectl get pod alpha -o jsonpath='{.spec.restartPolicy}'
```

`Always`. The restart policy applies to **containers inside the pod**, not to the pod itself. If the nginx process inside `alpha` crashes, kubelet will restart the container. But if the pod (the whole thing) is deleted, no one brings it back — that's the controller's job.

### 12. Has the container restarted before?

```bash
kubectl get pod alpha
```

The `RESTARTS` column. For a freshly applied healthy pod it's `0`.

### 13. CPU and memory requests and limits?

```bash
kubectl describe pod alpha | grep -A4 -i limits
```

- Requests: `cpu: 50m`, `memory: 64Mi` — what the scheduler reserves on the node.
- Limits: `cpu: 200m`, `memory: 128Mi` — the hard cap. Exceed memory and the container is OOM-killed; exceed CPU and it's throttled.

---

## Pod: `bravo`

### 1. How many containers are inside the pod?

```bash
kubectl get pod bravo
```

The `READY` column shows `2/2` — two app containers. Plus there was an init container that already finished:

```bash
kubectl get pod bravo -o jsonpath='{.spec.initContainers[*].name} | {.spec.containers[*].name}'
```

So: **2 application containers + 1 init container** = 3 containers defined in total.

### 2. Which container started first?

The **init container** (`prep`). Init containers always run to completion before any app container starts. Verify:

```bash
kubectl describe pod bravo | grep -A2 "Init Containers" 
```

Among the app containers, both start in parallel — there is no ordering guarantee for `writer` vs `reader`.

### 3. What type of volume is mounted?

```bash
kubectl get pod bravo -o jsonpath='{.spec.volumes}'
```

A **PersistentVolumeClaim** named `bravo-pvc`. The PVC binds to a PersistentVolume that your cluster's default StorageClass provisions.

### 4. Is the volume ephemeral or persistent?

**Persistent.** Data survives pod restarts and pod deletion (as long as the PVC isn't deleted). Compare to an `emptyDir` volume, which dies with the pod.

### 5. Which path is the volume mounted at?

```bash
kubectl describe pod bravo | grep -A1 "Mounts:"
```

`/data` — in both `writer` and `reader` containers. That's how they share the file.

### 6. Verify both containers see the same volume

```bash
# Writer appends timestamps
kubectl exec bravo -c writer -- tail -3 /data/heartbeat.log

# Reader sees the same lines
kubectl logs bravo -c reader | tail -3
```

Same content. The volume is genuinely shared.

### 7. Delete the pod — does data survive?

```bash
kubectl delete pod bravo
# wait — but note: a bare pod won't be recreated
kubectl apply -f manifests/bravo.yaml
kubectl exec bravo -c writer -- wc -l /data/heartbeat.log
```

The heartbeat log still has the older lines. Persistent storage outlived the pod because the PVC was untouched.

---

## Pod: `charlie`

### 1. Can you see this pod with plain `kubectl get pods`?

```bash
kubectl get pods
```

**No** — because it lives in a different namespace (`workshop`). `kubectl` defaults to the `default` namespace. To see it:

```bash
kubectl get pods -n workshop
# or, across all namespaces:
kubectl get pods -A
```

This is one of the most common confusions for newcomers — pods exist, but in another namespace.

### 2. Which resource directly owns the pod?

```bash
kubectl get pod -n workshop -l app=charlie -o jsonpath='{.items[0].metadata.ownerReferences}'
```

A **ReplicaSet**. Note: a Deployment doesn't own pods directly — it owns a ReplicaSet, and the ReplicaSet owns the pods. The chain is `Deployment → ReplicaSet → Pod`.

### 3. Deployment, ReplicaSet, Job, or StatefulSet?

```bash
kubectl get all -n workshop
```

There's a `Deployment` and a `ReplicaSet`. The Deployment is the top-level owner, and the ReplicaSet is the direct owner.

### 4. Delete the pod — what happens?

```bash
POD=$(kubectl get pod -n workshop -l app=charlie -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n workshop "$POD"
kubectl get pods -n workshop -w
```

A new pod with a different name appears almost immediately. The ReplicaSet noticed its desired replica count (1) was not met and created a new pod to fix it. This is the core reconciliation loop of Kubernetes.

### 5. Scale the Deployment to zero

```bash
kubectl scale deployment charlie -n workshop --replicas=0
kubectl get pods -n workshop
```

Now the pod stays gone. Desired state is zero, so no recreation. Scale back up:

```bash
kubectl scale deployment charlie -n workshop --replicas=1
```

---

## Concept recap

| Concept | Key idea |
|---|---|
| Imperative vs declarative | YAML + `apply` is declarative; `kubectl run` is imperative |
| Pod IP | Cluster-internal; not directly reachable from outside |
| External access | Needs a Service (NodePort, LoadBalancer, or Ingress) |
| Owner references | Bare pod = no owner; managed pod has a ReplicaSet/Job/etc. above it |
| Restart policy | Restarts **containers** inside a pod, never recreates the pod itself |
| Persistent volume | Survives pod deletion; `emptyDir` does not |
| Init container | Runs to completion **before** any app container |
| Namespace | Logical scope; `kubectl get pods` defaults to `default` |
| Deployment → ReplicaSet → Pod | Each layer reconciles toward desired state |
