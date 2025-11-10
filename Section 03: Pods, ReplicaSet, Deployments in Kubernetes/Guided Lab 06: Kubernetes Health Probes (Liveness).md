# Guided Lab 06 — Kubernetes Health Probes (Liveness)

> “What happens **after** it’s been running for a while? Who makes sure it’s still *healthy*?”

That’s where **Liveness Probes** come in. Let’s understand it. 

---

## 1) Introduction

---

## **You’ve Built a Smart, Reliable Deployment**

So far, your Deployment does a lot right:

* **Replicas (2)** → Always keeps two Pods alive.
* **Resources** → Prevents CPU starvation.
* **Rolling updates** → No downtime during upgrades.
* **Readiness Probe** → Ensures new Pods only get traffic when they’re ready.

You’ve taught Kubernetes how to wait patiently before trusting a Pod.
But once traffic starts flowing, a new question arises…

---

## **The Hidden Problem: What If a Running Pod “Hangs”?**

Imagine this real-world story:

Your `nginx` Pods have been serving traffic smoothly for hours.
Then suddenly — a bug, a memory leak, or a network glitch makes one container freeze.

* The process is still running in memory.
* Kubernetes sees it as “healthy” (because the container didn’t crash).
* But in reality, it’s not responding to requests anymore.

From Kubernetes’ point of view:

> “Container process is alive — everything’s fine.”
> From a user’s point of view:
> “Half the site is timing out!”

So now, you have a Pod that’s **running, but broken**.
And no one restarts it — because Kubernetes doesn’t know it’s sick.

---

## **This Is Where the Liveness Probe Saves You**

A **Liveness Probe** teaches Kubernetes *how to detect and fix stuck containers*.

You tell it:

> “Check this endpoint or command regularly. If it stops responding, restart the container automatically.”

---

### **How It Works**

Let’s say you add this probe to your container:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 20
```

Here’s what Kubernetes does behind the scenes:

1. **Wait 10 seconds** after startup (to let the app initialize).
2. Then, every **20 seconds**, send an HTTP GET request to `/` on port 80.
3. If it fails several times in a row (default: 3), Kubernetes says:

   > “This container isn’t healthy — restart it.”

It’s a simple but powerful self-healing loop.

---

## **Liveness vs. Readiness — Subtle but Crucial Difference**

| Feature             | Purpose                                           | When It Triggers        |
| ------------------- | ------------------------------------------------- | ----------------------- |
| **Readiness Probe** | Checks *if the app is ready* to handle traffic    | Before routing traffic  |
| **Liveness Probe**  | Checks *if the app is still alive* and responsive | After it’s been running |

Think of it like this:

* **Readiness = “I’m ready to serve customers.”**
* **Liveness = “I’m still serving customers properly.”**

---

## 2) Step-by-Step Hands-On Setup

We’ll keep files tidy in a dedicated folder, working in the **default namespace**.

```bash
cd ~
mkdir -p ~/k8s-labs/health-probes-liveness
cd ~/k8s-labs/health-probes-liveness
```

### 2.1 Baseline Deployment (no probes)

```bash
touch 00-nginx-deploy-baseline.yaml
vi 00-nginx-deploy-baseline.yaml
```

Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21.1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
            limits:
              cpu: "200m"
```

Apply and verify:

```bash
kubectl apply -f 00-nginx-deploy-baseline.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -o wide
kubectl describe deploy nginx-deployment | sed -n '1,160p'
```

**Why this baseline?** It matches your earlier deployments with **no probes**—so the effect of liveness will be obvious.

**Let’s recap:** Two NGINX Pods are running cleanly. No health checks yet.

---

## 3) Add a Liveness Probe

Now we’ll teach Kubernetes when to **restart** the container.

```bash
touch 01-nginx-deploy-liveness.yaml
vi 01-nginx-deploy-liveness.yaml
```

Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21.1
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /
              port: 80
            # Give nginx a small grace period, then verify regularly
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
          resources:
            requests:
              cpu: "100m"
            limits:
              cpu: "200m"
```

Apply and watch:

```bash
kubectl apply -f 01-nginx-deploy-liveness.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

Inspect one Pod to see the probe and events:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" | sed -n '1,240p'
```

**What changed:**

* Kubernetes will **periodically** verify NGINX responds on `/`.
* Sustained failures will **restart** the container in-place (Pod remains).

**Let’s recap:** Liveness is now active. It won’t affect traffic directly; it impacts **process survival** by restarting wedged containers.

---

## 4) Verification — Intentionally Trigger a Liveness Restart

To see liveness in action, we’ll make the probe **consistently fail** by pointing it to an invalid path (NGINX returns 404 → failure).

```bash
cp 01-nginx-deploy-liveness.yaml 02-nginx-liveness-broken.yaml
vi 02-nginx-liveness-broken.yaml
```

Change only the liveness path:

```yaml
          livenessProbe:
            httpGet:
              path: /does-not-exist   # force failures (404)
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
```

Apply and watch restarts:

```bash
kubectl apply -f 02-nginx-liveness-broken.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

After roughly **initialDelay + period × failureThreshold** (≈ 5 + 5×3 = 20s), you should see **RESTARTS** increment:

```bash
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,240p'
```

Fix the probe:

```bash
cp 01-nginx-deploy-liveness.yaml 03-nginx-liveness-fixed.yaml
kubectl apply -f 03-nginx-liveness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

**What you observed:**

* The kubelet **restarted** the container after sustained probe failures.
* Once fixed, restarts stop climbing and the Pod returns to steady state.

**Let’s recap:** You created a controlled failure and watched Kubernetes self-heal by restarting the container—exactly what liveness is for.

---

## 5) Explore Real-World Impact & Safe Tuning

Liveness is powerful, but aggressive settings can cause **restart loops**. Use these guidelines:

* **Start conservative:**
  `initialDelaySeconds: 10–15`, `periodSeconds: 10`, `timeoutSeconds: 1–2`, `failureThreshold: 3–5`.
  This avoids killing containers during brief hiccups.

* **Probe something cheap:**
  Keep the liveness endpoint lightweight (no heavy DB calls). If the DB is flaky, prefer failing **readiness** rather than liveness. Liveness is about *process health*, not *dependency health*.

* **Separate concerns:**

  * Use **readiness** to protect users from temporary issues.
  * Use **liveness** to recover from deadlocks and wedged states.

* **Watch `RESTARTS`:**
  `kubectl get pods` shows the restart count. If it steadily climbs, either the app is truly broken or your liveness thresholds are too strict.

**Quick drill: observe restart loops (optional)**

```bash
# Keep the broken liveness file applied for 1–2 minutes and watch:
kubectl get pods -l app=nginx -w
# Then fix it again with the good file:
kubectl apply -f 03-nginx-liveness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
```

**Let’s recap:** Liveness should be a **last resort** safety net—tuned to act on persistent unhealth, not transient slowness.

---

## 6) Troubleshooting

* **No restarts happen after breaking liveness**

  * Ensure you waited long enough: `initialDelaySeconds + periodSeconds × failureThreshold`.
  * Check events:

    ```bash
    kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,240p'
    ```
  * Confirm the probe is actually failing (non-2xx/3xx or timeout).

* **Restarts keep climbing even after fixing the probe**

  * Verify the applied YAML (`kubectl get deploy nginx-deployment -o yaml | less`).
  * Make sure the container actually serves on `targetPort` (80) and isn’t crash-looping for other reasons.

* **Rollouts feel slow**

  * That’s expected if Pods need time to stabilize. Consider larger `initialDelaySeconds` or a slightly bigger `failureThreshold`.

---

## 7) Clean Up (optional)

```bash
# Keep for future labs, or remove:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **Liveness** answers: *“Should Kubernetes restart this container?”*
* It’s different from readiness:

  * **Readiness** → *traffic gating* (Ready/NotReady), no restart.
  * **Liveness** → *self-healing* via **restart** after repeated failures.
* You:

  * Added a liveness probe.
  * Broke it intentionally (404 path) to trigger restarts.
  * Observed `RESTARTS` increase and events explaining why.
  * Restored a healthy probe and saw the Pod stabilize.

With readiness (availability) and liveness (survivability) mastered, your Deployments don’t just *run*—they **tell Kubernetes how to protect users and self-heal the process**.
