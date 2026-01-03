
# 🌅 Morning Experiment #1

## **“How Kubernetes Actually Schedules and Runs a Pod”**

This is the **foundation experiment**. Everything else (autoscaling, failures, AI, observability) builds on this.

---

## 🎯 What you’ll learn (by the end of this experiment)

You will *see* and *prove*:

1. How a Pod request flows:

   * API Server → Scheduler → Kubelet → containerd
2. Where **truth actually lives**:

   * API vs kubelet vs CRI
3. How scheduling constraints really work
4. What happens when you *intentionally* break things

This is the experiment most people **never do**, even after years of Kubernetes.

---

## ⏱️ Timebox

**30–45 minutes**
Perfect “morning warm-up” before heavier experiments later.

---

## Step 1 — Create a pod with intent (not a Deployment)

Create a pod that:

* requests CPU & memory
* pins to your node
* runs long enough to inspect

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: internals-pod
spec:
  nodeName: sting
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo Pod started; sleep 3600"]
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
EOF
```

Watch carefully:

```bash
kubectl get pod internals-pod -o wide -w
```

---

## Step 2 — Observe the scheduler’s decision

Describe the pod **before and after** it runs:

```bash
kubectl describe pod internals-pod
```

Focus on:

* **Events**
* Node assignment
* Resource checks

🔍 Ask yourself:

> What would stop this pod from scheduling?

---

## Step 3 — Compare three “truths” (this is the key insight)

### A) Kubernetes API view

```bash
kubectl get pod internals-pod -o wide
```

### B) Kubelet view (node-level reality)

```bash
sudo crictl pods
sudo crictl ps
```

### C) containerd view (runtime truth)

```bash
sudo ctr -n k8s.io containers list
sudo ctr -n k8s.io tasks list
```

🧠 **Important realization**

* Kubernetes does **not** run containers
* kubelet does
* Kubernetes only *describes desired state*

---

## Step 4 — Exec into the container (runtime boundary)

```bash
kubectl exec -it internals-pod -- sh
```

Inside:

```sh
ps
cat /proc/1/cgroup
```

Then exit.

Now from the host:

```bash
sudo crictl inspect $(sudo crictl ps -q | head -n1)
```

🔍 Notice:

* cgroup paths
* namespaces
* runtime metadata

---

## Step 5 — Controlled failure: kill the container (not the pod)

From the host:

```bash
sudo crictl stop $(sudo crictl ps -q)
```

Immediately watch:

```bash
kubectl get pod internals-pod -w
```

🧠 Observe:

* Who restarts the container?
* Did Kubernetes notice?
* Did kubelet act first?

This answers:

> “Who is actually in charge?”

---

## Step 6 — Read kubelet’s side of the story

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

Look for:

* Pod sync
* Container restart
* Runtime calls

This is **where real incidents are debugged**.

---

## Cleanup

```bash
kubectl delete pod internals-pod
```

---

## ✅ Success criteria (self-check)

You should be able to answer **confidently**:

* Who restarts a crashed container?
* What happens if API server is down but kubelet is running?
* Where does scheduling *actually* fail?
* Why `kubectl` sometimes lies during incidents

If you can answer those, you’ve done the experiment right.