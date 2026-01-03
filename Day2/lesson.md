# 📅 Day 2 – Kubernetes Under Pressure

**Theme:**

> *What actually happens when pods hit CPU/memory limits and when nodes are stressed?*

**Timebox:** ~45–60 minutes
**Outcome:** You’ll understand **requests vs limits**, **CPU throttling**, **OOMKills**, and **who takes action (kubelet vs kernel vs scheduler)**.

---

## 0️⃣ Pre-check (2 minutes)

Just confirm you’re scoped correctly:

```bash
kubectl config view --minify | grep namespace
```

Expected:

```
namespace: lab
```

---

## 1️⃣ Experiment 1 — CPU requests vs limits (throttling)

### Create a CPU-hungry pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-pressure
spec:
  containers:
  - name: stress
    image: polinux/stress
    args: ["--cpu", "2"]
    resources:
      requests:
        cpu: "200m"
      limits:
        cpu: "500m"
EOF
```

Watch it:

```bash
kubectl get pod cpu-pressure -w
```

Once running:

```bash
kubectl top pod cpu-pressure
```

### Observe (important)

* CPU usage **won’t exceed ~500m**
* Pod stays Running
* No restarts

🔍 **Key insight**
CPU limits cause **throttling**, not restarts.

---

## 2️⃣ See throttling at the node level (real internals)

Find the container ID:

```bash
sudo crictl ps | grep cpu-pressure
```

Inspect cgroups:

```bash
sudo crictl inspect <container-id> | grep -A5 cpu
```

You’re now seeing **kernel-enforced limits**, not Kubernetes magic.

---

## 3️⃣ Experiment 2 — Memory limits (OOMKill)

This is where Kubernetes behaves very differently.

### Create a memory-hungry pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mem-pressure
spec:
  containers:
  - name: stress
    image: polinux/stress
    args: ["--vm", "1", "--vm-bytes", "300M"]
    resources:
      requests:
        memory: "128Mi"
      limits:
        memory: "200Mi"
EOF
```

Watch closely:

```bash
kubectl get pod mem-pressure -w
```

Then:

```bash
kubectl describe pod mem-pressure
```

### What you should see

* Pod enters `CrashLoopBackOff`
* Reason: **OOMKilled**
* Restart count increases

🔍 **Key insight**
Memory limits are **hard** → kernel kills the container.

---

## 4️⃣ Who killed it? (critical SRE understanding)

Check kubelet logs:

```bash
sudo journalctl -u kubelet -n 50 --no-pager
```

You’ll see:

* kubelet reporting container termination
* kernel OOM signal underneath

🧠 **Mental model**

* Kernel kills container
* kubelet observes
* Kubernetes reacts (restart policy)

---

## 5️⃣ Scheduler reality check (why requests matter)

Delete both pods:

```bash
kubectl delete pod cpu-pressure mem-pressure
```

Now create a pod with **huge requests**:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: unschedulable
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "8"
        memory: "16Gi"
EOF
```

Check:

```bash
kubectl get pod unschedulable
kubectl describe pod unschedulable
```

### Observe

* Pod stays `Pending`
* Scheduler event explains **why**

🔍 **Key insight**
Scheduler uses **requests**, not limits.

---

## 6️⃣ Cleanup (always clean)

```bash
kubectl delete pod unschedulable
```

---

## ✅ Day 2 Success Criteria

You should now be able to answer confidently:

* Why CPU limits don’t restart pods
* Why memory limits do
* Who actually enforces resource limits
* Why bad requests can block scheduling
* Where to look during “pod keeps restarting” incidents

If you can explain this without looking at notes — Day 2 is done.