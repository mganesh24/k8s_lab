# Healthy Pods — Tasks

Apply each manifest and then answer the questions for that pod using only `kubectl`. Do not look at the YAML for the answers — practice getting them from the running cluster, the way you would in real life.

If you get stuck, the solutions file has every answer with the exact command.

---

## Pod: `alpha`

```bash
kubectl apply -f manifests/alpha.yaml
```

1. Was this pod created imperatively or declaratively?
2. Which node is the pod running on?
3. What image is the container using?
4. What is the pod IP address?
5. Can the pod reach the internet?
6. Is the pod accessible from the external world? If yes, how?
7. When was the pod created?
8. What phase is the pod currently in?
9. Which Kubernetes resource owns this pod? Is it managed by a Deployment, ReplicaSet, Job, or StatefulSet?
10. What happens if you delete the pod manually?
11. What is the pod's restart policy?
12. Has the container restarted before? How many times?
13. Does the pod have CPU and memory requests and limits? What are they?

---

## Pod: `bravo`

```bash
kubectl apply -f manifests/bravo.yaml
```

1. How many containers are inside the pod?
2. Which container started first?
3. What type of volume is mounted in the pod?
4. Is the volume ephemeral or persistent?
5. Which path inside the container is the volume mounted at?
6. Verify that both application containers are writing to / reading from the same volume.
7. Delete the pod. Is the data still there when the pod comes back? Why or why not?

---

## Pod: `charlie`

```bash
kubectl apply -f manifests/charlie.yaml
```

1. Can you see this pod using `kubectl get pods` with no flags? If not, why?
2. Which Kubernetes resource is the direct owner of this pod?
3. Is the pod managed by a Deployment, ReplicaSet, Job, or StatefulSet?
4. Delete the pod. What happens? Why?
5. Now scale the owning resource to zero. Does the pod come back this time?
