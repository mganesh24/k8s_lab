# Broken Pods — Tasks

Each of the following pods is broken for a different reason. Apply them one at a time and triage. The questions are the same for each pod — your goal is to get fluent at the triage loop:

1. **What status is the pod currently in?**
2. **How can you identify the failure reason?** (Which command(s) gave you the answer?)
3. **Can you find any warnings or errors?** Where exactly did you find them?
4. **What does the application log say?** (If there is one — sometimes the failure happens before the app even starts.)
5. **What changes after restarting the pod?** (Delete and re-apply, or watch automatic restarts.)
6. **What single change would fix this pod?** Don't actually fix it yet — write down your hypothesis first.

---

## Pod: `delta`

```bash
kubectl apply -f manifests/delta.yaml
```

Answer the six questions above.

---

## Pod: `echo`

```bash
kubectl apply -f manifests/echo.yaml
```

Answer the six questions above. Pay attention to the restart count over time.

---

## Pod: `foxtrot`

```bash
kubectl apply -f manifests/foxtrot.yaml
```

Answer the six questions above. The interesting field for this one is `Last State` in `kubectl describe`.

---

## Pod: `golf`

```bash
kubectl apply -f manifests/golf.yaml
```

Answer the six questions above. Hint: this pod's problem happens before any container is even pulled.

---

## After you're done

For each pod, write down (in one sentence each):
- The failure category (image, runtime, resource, scheduling, config, networking, …)
- The single line in the YAML that caused it
- The command that gave you the clearest signal

That table is the actual deliverable from this lab.
