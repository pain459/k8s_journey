## Why this is a good practice (quick confirmation)

By doing this you get:

* ✅ Safety: no accidental work in `default` or `kube-system`
* ✅ Clean teardown: delete one namespace, everything is gone
* ✅ Mental separation: *system* vs *experiments*
* ✅ Muscle memory: all `kubectl` commands “just work”

This is especially important when you start doing:

* failure experiments
* resource pressure tests
* networking breakage
* controllers / CRDs

---

## Step 1 — Create an experiment namespace

Pick a name that reflects intent. Good options:

* `lab`
* `experiments`
* `sandbox`
* `playground`

I’ll assume **`lab`** (short, neutral).

```bash
kubectl create namespace lab
```

Verify:

```bash
kubectl get ns
```

---

## Step 2 — Make it your default namespace (this is the key step)

This modifies your **kubectl context**, not the cluster.

```bash
kubectl config set-context --current --namespace=lab
```

Verify:

```bash
kubectl config view --minify | grep namespace
```

Expected:

```
namespace: lab
```

From now on:

```bash
kubectl get pods
kubectl apply -f pod.yaml
```

→ all go to **`lab`** automatically.

---

## Step 3 — Sanity check (important)

Run:

```bash
kubectl get pods
kubectl get deployments
```

You should see **nothing**, and **no errors**.

Then confirm system namespaces are untouched:

```bash
kubectl get pods -n kube-system | head
```

---

## Step 4 — (Optional but recommended) Label the namespace

This helps later with policies, selectors, or automation.

```bash
kubectl label namespace lab purpose=experiments owner=ravik
```

Verify:

```bash
kubectl get ns lab --show-labels
```

---

## Step 5 — Test with a tiny pod (confidence check)

```bash
kubectl run hello \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sh -c "echo hello from lab; sleep 60"
```

Check:

```bash
kubectl get pods
kubectl logs hello
```

Cleanup:

```bash
kubectl delete pod hello
```

---

## Step 6 — Escape hatch (important to know)

If you ever want to run something **outside** the lab:

```bash
kubectl get pods -n kube-system
kubectl apply -n default -f something.yaml
```

Namespace scoping stays explicit when needed.

---

## Recommended habit going forward

* **All experiments** → `lab`
* **Never touch**:

  * `kube-system`
  * `kube-public`
* **One experiment = one namespace** (later, if needed)

Example future flow:

```bash
kubectl create ns cpu-tests
kubectl config set-context --current --namespace=cpu-tests
```
