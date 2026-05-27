# Kubernetes Pods — Hands-on Lab

A guided lab for Versa MS engineers who are new to Kubernetes. You will deploy a handful of pods to your lab cluster (one control plane, two workers) and then answer questions about them using `kubectl`.


---

## What's in this folder

```
pods/
├── README.md                 ← you are here
├── manifests/                ← apply these to your cluster
│   ├── alpha.yaml
│   ├── bravo.yaml
│   ├── charlie.yaml
│   ├── delta.yaml
│   ├── echo.yaml
│   ├── foxtrot.yaml
│   └── golf.yaml
├── tasks/
│   ├── healthy-pods.md       ← questions to answer
│   └── broken-pods.md
└── solutions/
    ├── healthy-pods-solutions.md   ← commands + short explanations
    └── broken-pods-solutions.md
```

> If you feel stuck or intimidated, jump straight to the `solutions/` files. Each answer has the exact command plus a one or two line explanation so you can follow along and still pick up the concept.

---

## Prerequisites

You need:

1. If you've not set the lab yet, please follow the README.md file in https://github.com/mganesh24/k8s_lab/tree/main/lab_setup_steps to set the lab. 

```
sudo apt update
sudo apt install -y git
git --version
```
Quick sanity check:

```bash
kubectl get nodes
```

You should see the `STATUS` of all the nodes in `Ready` state.

2. For PersistentVolumeClaim(PVC) the k8s cluster should have a default StorageClass.

We fix this once per cluster by installing local-path-provisioner — a tiny add-on that watches for PVCs and carves out storage from a folder on whichever worker node the pod lands on.

Run these four commands once. You won't need to repeat them.
  Install the provisioner
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml
```
**What it does:** creates a namespace called `local-path-storage`, an RBAC bundle (ServiceAccount, Role, ClusterRole, bindings), a Deployment that runs the provisioner pod, a ConfigMap with its settings, and — most importantly — a StorageClass named `local-path`.

Confirm the provisioner is running

```
kubectl -n local-path-storage get pods -w
```
**What it does:** watches (`-w`) the pods in the `local-path-storage` namespace until the provisioner pod reports `1/1 Running`. Press `Ctrl+C` to exit the watch.

Mark it as the default StorageClass
```
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
**What it does:** adds an annotation to the `local-path` StorageClass that tells Kubernetes "use this one when a PVC doesn't ask for a specific class."

Verify
```
kubectl get storageclass
```

**What it does:** lists all StorageClasses in the cluster.

---

## Getting the lab repo onto your machine

Clone the training repo
```bash
git clone https://github.com/mganesh24/k8s_lab
cd k8s_lab/pods
```

---

## Suggested workflow

1. **Healthy pods first.** Apply `alpha.yaml`, `bravo.yaml`, `charlie.yaml` one by one. Open `tasks/healthy-pods.md` and answer the questions for each pod. Use `solutions/healthy-pods-solutions.md` when you get stuck.
2. **Then the broken ones.** Apply `delta.yaml`, `echo.yaml`, `foxtrot.yaml`, `golf.yaml`. Open `tasks/broken-pods.md`. Each pod fails for a different reason — your job is to find out why using only `kubectl`.

The broken pods are the most valuable part of this lab. The skill you're building is reading `describe` output, events, and logs — that's 80% of real-world triage.

---

## Commands you'll lean on

| Command | What it shows |
|---|---|
| `kubectl get pods` | List pods in the current namespace |
| `kubectl get pods -A` | List pods across all namespaces |
| `kubectl get pods -o wide` | Adds node, IP, and more |
| `kubectl describe pod <name>` | Status, events, restart count, last state |
| `kubectl logs <pod>` | Container logs (current run) |
| `kubectl logs <pod> --previous` | Logs from the previous crash |
| `kubectl logs <pod> -c <container>` | Logs from a specific container in a multi-container pod |
| `kubectl get pod <name> -o yaml` | Full live spec and status |
| `kubectl exec -it <pod> -- sh` | Shell into the container |
| `kubectl delete -f manifests/<file>.yaml` | Tear down what you applied |

---

## Cleaning up

When you're done:

```bash
kubectl delete -f manifests/
kubectl delete namespace workshop      # one of the manifests creates this
kubectl delete pvc bravo-pvc           # PVCs aren't always cleaned up by `delete -f`
```

Verify nothing's left:

```bash
kubectl get pods -A
kubectl get pvc -A
```

---

## Where to next

In the upcoming sessions, we'll explore Services (beyond NodePort), ConfigMaps and Secrets, Deployments and rolling updates, and then Ingress, etc.
