# Guided Lab — Kubernetes Health Probes (Readiness & Liveness)

## 1) Introduction

Deployments ensure Pods exist and roll out new versions safely. But Kubernetes still needs a way to answer two questions:

1. **Can this Pod accept traffic right now?**
   That’s **readiness**. If the readiness probe fails, the Pod is marked **not Ready**. It keeps running, but the control plane treats it as unavailable for work.

2. **Is this container stuck and should it be restarted?**
   That’s **liveness**. If the liveness probe fails repeatedly, the kubelet **restarts** the container.

In this lab, you’ll add both probes to your existing NGINX Deployment, then trigger failures on purpose. You’ll watch how Kubernetes reflects those states and recovers.

**Let’s recap the goal before we begin:**

* Start from your last working Deployment (the one you shared).
* Add a **readinessProbe** (HTTP GET `/`).
* Add a **livenessProbe** (HTTP GET `/`).
* Break each probe once to see what happens.
* Fix it and confirm recovery.

---

## 2) Step-by-Step Hands-On Setup

We’ll keep your labs tidy in a dedicated folder, but we’ll work in the **default namespace**.

```bash
cd ~
mkdir -p ~/k8s-labs/health-probes
cd ~/k8s-labs/health-probes
```

### 2.1 Save your last known good Deployment (baseline, no probes)

```bash
touch 00-nginx-deploy-baseline.yaml
vi 00-nginx-deploy-baseline.yaml
```

Paste the YAML you’re already using:

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
kubectl get pods -l app=nginx
kubectl describe deployment nginx-deployment | sed -n '1,120p'
```

**What you did:** You re-applied the exact Deployment you’ve been using, so we have a consistent starting point without any probes.

**Let’s recap:** We now have two replicas of NGINX running as before. Next, we’ll add probes to teach Kubernetes how to decide “ready” and “alive.”

---

## 3) Add Readiness & Liveness Probes

We’ll add both probes in a single change. For NGINX, an HTTP GET to `/` is simple and effective.

```bash
touch 01-nginx-deploy-with-probes.yaml
vi 01-nginx-deploy-with-probes.yaml
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
          # --- Probes we are adding now
          readinessProbe:
            httpGet:
              path: /
              port: 80
            # Quick but not too aggressive: check every 5s, 1s timeout
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
          livenessProbe:
            httpGet:
              path: /
              port: 80
            # Give it a short grace period before first liveness check
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

Apply and observe:

```bash
kubectl apply -f 01-nginx-deploy-with-probes.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

Describe one Pod to see probe sections and conditions:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" | sed -n '1,200p'
```

**What those fields mean (at a glance):**

* `readinessProbe`: Decides if a Pod is **Ready**. Failures mark it **not Ready** but do not restart the container.
* `livenessProbe`: Decides if the container should be **restarted** after repeated failures.
* `initialDelaySeconds`: Wait before the first check; useful to avoid false alarms right after start.
* `periodSeconds`: How often to check.
* `failureThreshold`: How many consecutive failures before taking action.

**Let’s recap:** You added both probes. Kubernetes will now withhold “Ready” until `/` answers successfully and will restart containers if `/` keeps failing liveness.

---

## 4) Verification — Make Readiness Fail, Then Fix It

To truly “feel” readiness, we’ll break it by pointing to a non-existent path. HTTP probes consider only **2xx/3xx** responses successful; a `404` fails.

```bash
cp 01-nginx-deploy-with-probes.yaml 02-nginx-readiness-broken.yaml
vi 02-nginx-readiness-broken.yaml
```

Change only the readiness `path`:

```yaml
          readinessProbe:
            httpGet:
              path: /does-not-exist
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
```

Apply and watch:

```bash
kubectl apply -f 02-nginx-readiness-broken.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
```

What you’ll notice:

* Pod(s) show **Ready: False** in the `Conditions`.
* **Restart count stays the same**. Readiness does not restart containers.

Now fix readiness:

```bash
cp 01-nginx-deploy-with-probes.yaml 03-nginx-readiness-fixed.yaml
kubectl apply -f 03-nginx-readiness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
```

**Let’s recap what you saw:**

* Breaking readiness made the Pod **not Ready** but didn’t kill it.
* Fixing the probe returned the Pod to **Ready**.
* This is exactly what you want during slow inits or dependencies not yet available: keep the Pod running, but don’t consider it ready.

---

## 5) Verification — Make Liveness Fail, Then Fix It

Now we’ll prove that liveness actually **restarts** containers on persistent failure. An easy way is to point the liveness probe at the **wrong port**.

```bash
cp 03-nginx-readiness-fixed.yaml 04-nginx-liveness-broken.yaml
vi 04-nginx-liveness-broken.yaml
```

Change only the liveness `port` from `80` to `81`:

```yaml
          livenessProbe:
            httpGet:
              path: /
              port: 81            # <-- wrong on purpose
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
```

Apply and watch:

```bash
kubectl apply -f 04-nginx-liveness-broken.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

After ~30–40 seconds (initial delay + failures), you should see **RESTARTS** start to increment:

```bash
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
```

Fix liveness:

```bash
cp 03-nginx-readiness-fixed.yaml 05-nginx-liveness-fixed.yaml
kubectl apply -f 05-nginx-liveness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

**What just happened and why:**

* Liveness failures tell the kubelet, “this container is unhealthy; restart it.”
* By pointing to an invalid port, the probe failed consistently. After the threshold, Kubernetes restarted the container.
* Restoring the correct port stopped the restarts.

**Let’s recap:** Readiness affects **availability** (Ready/NotReady). Liveness affects **survivability** (Restart on failure). You’ve seen both in action.

---

## 6) Quick Tuning Guidance (just enough for now)

* **Start simple**: For web apps, check `/` or a lightweight `/health` endpoint.
* **Make readiness strict, liveness conservative**: Prefer catching transient issues in readiness. Let liveness act only when the container is truly in a bad state.
* **Use small timeouts**: Probes run often; keep them cheap (e.g., `timeoutSeconds: 1`).
* **Give liveness a short initial delay**: Avoid restarts during the first few seconds of startup.

---

## 7) Clean Up (optional)

If you want to remove the resources created in this lab:

```bash
kubectl delete deployment nginx-deployment
```

(We stayed in the default namespace; nothing else to clean.)

---

## Wrap-Up: What Did You Learn?

* **Readiness Probe** answers: *Can this Pod serve?*
  Failure → Pod marked **not Ready**, but it **keeps running**.

* **Liveness Probe** answers: *Should this container be restarted?*
  Failure (over time) → **kubelet restarts** the container.

* You applied both probes to your existing Deployment, **broke each deliberately**, and observed the expected behaviors—**not Ready** without restarts for readiness failures, and **restarts** for liveness failures.

* You tuned only a few core fields (`initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`) and stayed within concepts already covered.
