# Broken Pods — Solutions

For each pod we'll walk the same triage loop:

1. `kubectl get pod <name>` — what status?
2. `kubectl describe pod <name>` — what do the events and conditions say?
3. `kubectl logs <name>` — what does the container itself say?
4. Identify the root cause.
5. Propose the one-line fix.

That loop solves the vast majority of real-world pod failures.

---

## Pod: `delta`

### Status

```bash
kubectl get pod delta
```

```
NAME    READY   STATUS             RESTARTS   AGE
delta   0/1     ImagePullBackOff   0          30s
```

You may also briefly see `ErrImagePull` before it backs off.

### Failure reason

```bash
kubectl describe pod delta
```

Scroll to the bottom — the `Events` section will say something like:

```
Failed to pull image "nginx:thisversiondoesnotexist": ... manifest unknown
Error: ErrImagePull
Back-off pulling image "nginx:thisversiondoesnotexist"
```

### Warnings or errors

All in the events section of `describe`. The `Type` column will say `Warning`.

### Application logs

```bash
kubectl logs delta
```

```
Error from server (BadRequest): container "app" in pod "delta" is waiting to start: image can't be pulled
```

No app logs — the container never started.

### What changes after restarting?

Nothing — kubelet keeps retrying with exponential backoff. The status alternates between `ErrImagePull` and `ImagePullBackOff`.

### Category and fix

| | |
|---|---|
| Category | **Image / registry** |
| Root cause | The image tag `nginx:thisversiondoesnotexist` does not exist on Docker Hub |
| Fix | Change the tag to a real one, e.g. `nginx:1.27-alpine` |

> **Lesson:** `ImagePullBackOff` is almost always a typo in the image name, a missing tag, or a private registry without imagePullSecrets. Look at the `Events` first.

---

## Pod: `echo`

### Status

```bash
kubectl get pod echo
```

After a minute or so:

```
NAME   READY   STATUS             RESTARTS      AGE
echo   0/1     CrashLoopBackOff   4 (30s ago)   2m
```

Restart count keeps climbing.

### Failure reason

```bash
kubectl describe pod echo
```

Look for `Last State`:

```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```

Exit code 1 = the application itself exited with an error.

### Warnings or errors

The `Events` section shows `Back-off restarting failed container`. The exit reason is in `Last State`, not in events.

### Application logs

```bash
kubectl logs echo
```

```
starting up...
fatal: configuration missing
```

There it is. The app is telling you exactly what's wrong.

If the pod has already restarted and you want the previous run's logs:

```bash
kubectl logs echo --previous
```

### What changes after restarting?

The restart count goes up, and the backoff delay between restarts increases (10s → 20s → 40s → … capped at 5 minutes). Status stays `CrashLoopBackOff`.

### Category and fix

| | |
|---|---|
| Category | **Application / runtime** |
| Root cause | The container's command intentionally exits 1 after 2 seconds |
| Fix | Fix the application, or provide whatever configuration it's looking for |

> **Lesson:** `CrashLoopBackOff` means the container started and then died. The fix is almost always in the logs (or in `--previous` if the current attempt hasn't finished yet).

---

## Pod: `foxtrot`

### Status

```bash
kubectl get pod foxtrot
```

You'll see `Running` briefly, then `OOMKilled`, then `CrashLoopBackOff` after a few restarts:

```
NAME      READY   STATUS             RESTARTS   AGE
foxtrot   0/1     CrashLoopBackOff   3          1m
```

### Failure reason

```bash
kubectl describe pod foxtrot
```

This is the key field:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Exit code 137 = killed by SIGKILL (128 + signal 9). Combined with `Reason: OOMKilled`, that's unambiguous — the kernel killed the process for exceeding its memory limit.

### Warnings or errors

`describe` events show `Back-off restarting failed container`. The OOMKill signal itself is in the container's `Last State`, not in pod events.

### Application logs

```bash
kubectl logs foxtrot --previous
```

The `stress` tool may have been killed mid-allocation; logs are usually unhelpful for OOMKills because the kernel kills the process before anything is flushed. **Look at `Last State`, not logs, for OOM diagnosis.**

### What changes after restarting?

Same outcome every time. Each restart hits the memory limit again and gets killed.

### Category and fix

| | |
|---|---|
| Category | **Resource limits** |
| Root cause | Container tries to allocate ~150 MB but the limit is 64 Mi |
| Fix | Either raise `resources.limits.memory` to e.g. `256Mi`, or reduce what the app allocates |

> **Lesson:** `Reason: OOMKilled` and exit code `137` together = memory limit too low (or memory leak). For CPU, exceeding the limit doesn't kill the container — it throttles it.

---

## Pod: `golf`

### Status

```bash
kubectl get pod golf
```

```
NAME   READY   STATUS    RESTARTS   AGE
golf   0/1     Pending   0          1m
```

`Pending` and never moving. This is a different family of failure entirely — the pod hasn't been scheduled to any node, so the container hasn't even been created.

### Failure reason

```bash
kubectl describe pod golf
```

Events section:

```
Warning  FailedScheduling   default-scheduler  0/3 nodes are available:
3 node(s) didn't match Pod's node affinity/selector.
```

The scheduler is telling you exactly why. No node satisfies the pod's requirements, so it stays pending.

### Warnings or errors

In `Events`, with `Type: Warning` and `Reason: FailedScheduling`. There are no container-level logs because there is no container yet.

### Application logs

```bash
kubectl logs golf
```

```
Error from server (BadRequest): container "app" in pod "golf" is waiting to start:
ContainerCreating
```

Nothing useful — the container was never created.

### What changes after restarting?

Deleting and reapplying gives the same `Pending` state forever. The pod will sit there until the cluster gains a node with `disktype=nvme-ultra` or you change the selector.

### Category and fix

| | |
|---|---|
| Category | **Scheduling** |
| Root cause | `nodeSelector: disktype=nvme-ultra` matches no node in the cluster |
| Fix | Remove the selector, change it to a label that exists, or label a node with `kubectl label node <node> disktype=nvme-ultra` |

You can check what labels your nodes actually have:

```bash
kubectl get nodes --show-labels
```

> **Lesson:** `Pending` for more than a few seconds is a scheduler problem, not an app problem. The scheduler always tells you why in `kubectl describe`. Other common causes: insufficient CPU/memory across the cluster, taints without matching tolerations, PVC binding failures.

---

## Triage cheat sheet

| Symptom | Most likely cause | First command to run |
|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Bad image name, missing tag, private registry | `kubectl describe pod` (events) |
| `CrashLoopBackOff` with non-zero exit code | App crash | `kubectl logs <pod> --previous` |
| `OOMKilled` (exit 137) | Memory limit too low or memory leak | `kubectl describe pod` (last state) |
| `Pending` forever | Scheduler can't place it (selector, resources, taints) | `kubectl describe pod` (events) |
| `ContainerCreating` forever | Volume mount issue, image still pulling, network plugin issue | `kubectl describe pod` (events) |
| `Running` but `0/1 Ready` | Failing readiness probe | `kubectl describe pod` (events + probe definition) |

The pattern is clear: **`kubectl describe pod` is your first stop for almost everything**, and `kubectl logs` (with or without `--previous`) is your second.

---

## Clean up

```bash
kubectl delete -f manifests/
kubectl delete namespace workshop
kubectl delete pvc bravo-pvc
```
