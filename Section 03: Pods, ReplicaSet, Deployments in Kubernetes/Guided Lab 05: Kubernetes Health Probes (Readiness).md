# Guided Lab 05 — Kubernetes Health Probes (Readiness)

> Scope: Only **readiness**. Default namespace. No Services yet. We’ll prove why readiness matters by watching how it **gates rollouts** and **Pod availability**—the practical “why” before we learn how traffic is routed.

---

## 1) Introduction

### **You already have a running Deployment**

You’ve defined:

* **Replicas:** 2 — Kubernetes keeps two Pods running.
* **Resources:** `requests`/`limits` — each Pod gets fair CPU.
* **RollingUpdate strategy** — safe, graceful upgrades.

So far, so good. The Deployment looks clean.
But here’s the big gap: **how does Kubernetes know when your container is actually *ready* to serve traffic?**

---

### **The Hidden Problem**

“Running” is not the same as “Ready.”

Imagine this:

* The container process starts.
* But NGINX is still booting, parsing config, warming cache, or waiting on a dependency.
* If Kubernetes assumes “Pod is running ⇒ good to go,” users may hit **timeouts** or **errors**.

We need a way to say: **“Don’t send me work until I’m truly ready.”**

> Readiness Probe: Determines if a container is ready to accept traffic.
> Liveness Probe: Checks if a container is still running correctly. If it fails, the container is restarted. 
> Startup Probe: Verifies that the application within a container has started successfully.

---

### **What Readiness Probes Do**

A **readinessProbe** tells Kubernetes exactly when a Pod can reliably take requests.

* Until the probe **succeeds**, **Ready = False**.
* During that time, the Pod **does not count** as available for rollouts and (later when we add Services) is **excluded** from endpoints.
* When the probe **passes**, **Ready = True** and Kubernetes treats it as available.

> In rolling updates, this prevents the platform from swapping out old, healthy Pods for new Pods that are still warming up.

---

### **A Quick Example**

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

* Wait 5s after container start.
* Then hit `/` on port 80 every 10s.
* Only after a 200 OK will Kubernetes mark the Pod **Ready**.

---

## 2) Step-by-Step Hands-On Setup

We’ll keep files tidy in a dedicated folder, working in the **default namespace**.

```bash
cd ~
mkdir -p ~/k8s-labs/health-probes-readiness
cd ~/k8s-labs/health-probes-readiness
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
kubectl describe deploy nginx-deployment | sed -n '1,120p'
```

**Why this baseline?** It mirrors what you already have—**no probes yet**—so you can feel the difference once readiness is added.

**Let’s recap:** Two NGINX Pods running, clean rollout config, no health checks beyond “process is running.”

---

## 3) Add a Readiness Probe

Now we’ll teach Kubernetes when the Pod is **actually** ready.

```bash
touch 01-nginx-deploy-readiness.yaml
vi 01-nginx-deploy-readiness.yaml
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
          readinessProbe:
            httpGet:
              path: /
              port: 80
            # Tuning for quick feedback but avoiding false alarms
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
          resources:
            requests:
              cpu: "100m"
            limits:
              cpu: "200m"
```

Apply and watch:

```bash
kubectl apply -f 01-nginx-deploy-readiness.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

Inspect probe & conditions:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" | sed -n '1,200p'
```

**What changed:**

* The Pod won’t be considered **Ready** until the probe passes.
* With `maxUnavailable: 0`, rollouts require **a new Pod to be Ready** before an old Pod is removed—your “zero-downtime” safety net.

**Let’s recap:** Readiness is now a first-class signal. Kubernetes won’t count a Pod as available until `/` responds successfully.

---

## 4) Feel Readiness by Breaking It (and Fixing It)

We’ll intentionally fail the probe by pointing it to a path that returns 404.

```bash
cp 01-nginx-deploy-readiness.yaml 02-nginx-readiness-broken.yaml
vi 02-nginx-readiness-broken.yaml
```

Change only the probe path:

```yaml
          readinessProbe:
            httpGet:
              path: /does-not-exist   # break readiness on purpose
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
```

Apply and observe:

```bash
kubectl apply -f 02-nginx-readiness-broken.yaml
kubectl rollout status deployment/nginx-deployment --timeout=60s || true
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
```

**What you should see:**

* Pod **Conditions** show **Ready: False**.
* The Deployment can struggle to complete a rollout (since new Pods never become Ready).

Now **fix** readiness:

```bash
cp 01-nginx-deploy-readiness.yaml 03-nginx-readiness-fixed.yaml
kubectl apply -f 03-nginx-readiness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

**Let’s recap:** You saw Pods stay **NotReady** when the probe fails, and the rollout won’t fully complete. Restoring a good probe returns Pods to **Ready**, and the rollout finishes cleanly.

---

## 5) See How Readiness Gates Rolling Updates

This is the real “why”: readiness prevents bad versions from replacing good ones.

1. **Start from the fixed readiness spec**:

```bash
kubectl apply -f 03-nginx-readiness-fixed.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

2. **Trigger an upgrade** to a new image (good case):

```bash
kubectl set image deploy/nginx-deployment nginx=nginx:1.21.2
kubectl rollout status deploy/nginx-deployment
kubectl get rs -l app=nginx
kubectl get pods -l app=nginx -o wide
```

You’ll see a new ReplicaSet created; new Pods become **Ready**, then old Pods are scaled down—**no downtime**.

3. **Rollback the image and break readiness** (to see rollout protection):

```bash
kubectl set image deploy/nginx-deployment nginx=nginx:1.21.3
# Now make readiness fail on the template so new pods never become Ready
cat > 04-readiness-broken-patch.yaml <<'EOF'
spec:
  template:
    spec:
      containers:
        - name: nginx
          readinessProbe:
            httpGet:
              path: /does-not-exist
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
EOF

kubectl patch deploy nginx-deployment --type merge --patch-file 04-readiness-broken-patch.yaml
kubectl rollout status deploy/nginx-deployment --timeout=60s || true
kubectl get pods -l app=nginx
kubectl describe deploy nginx-deployment | sed -n '1,200p'
```

The rollout should **hang** or stay **in progress**, because new Pods never reach **Ready**.
Fix it:

```bash
kubectl apply -f 03-nginx-readiness-fixed.yaml
kubectl rollout status deploy/nginx-deployment
```

**Let’s recap:** Readiness + `maxUnavailable: 0` = safe rollouts. New Pods must prove they’re Ready before old Pods are retired.

---

## 6) Practical Tuning Notes (just enough for now)

* **HTTP GET to a lightweight path** (`/healthz` or `/`) is perfect for web apps.
* **`initialDelaySeconds`** covers startup warmup; **don’t** make it so long that you hide real issues.
* **`periodSeconds`/`timeoutSeconds`**: keep checks cheap and frequent; `1s` timeouts are common.
* **`failureThreshold`**: small numbers fail fast; larger values give apps a chance to recover during brief hiccups.
* **Rollouts**: with `maxUnavailable: 0`, readiness directly enforces zero-downtime behavior.

---

## 7) Clean Up (optional)

```bash
# Keep it for the next lab, or remove it now:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **Readiness** answers: *“Can this Pod reliably take requests?”*
  Until it’s true, **Ready = False**.

* With readiness in place, Kubernetes:

  * **Excludes** NotReady Pods from availability (and later from Service endpoints).
  * **Protects** rolling updates—new Pods must become Ready before old Pods are removed (thanks to `maxUnavailable: 0`).

* You:

  * Added a readiness probe.
  * Broke it to see **NotReady** behavior and rollout stalling.
  * Fixed it to complete the rollout cleanly.
  * Learned sane defaults and what each knob does.

**Next:** when we understand health probes **Liveness**. 
