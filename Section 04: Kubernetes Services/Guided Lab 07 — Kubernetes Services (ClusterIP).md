# Guided Lab 06 — Kubernetes Services (ClusterIP)

## 1) Introduction — Why Services?

Your Deployments keep Pods alive and roll out new versions safely. Health probes tell Kubernetes whether a Pod can serve and whether it should be restarted. **But how do other Pods actually find and talk to your app when Pod IPs keep changing?**

That’s the problem Services solve.

### What is a Service?

A **Service** is a stable front door for a set of Pods:

* It has a **stable virtual IP (ClusterIP)** and a **stable DNS name**.
* It uses a **label selector** to pick which Pods are “behind” it.
* It **load-balances** traffic across those selected Pods.

### How does it work (behind the scenes)?

* **You** create a Service with a selector (e.g., `app: nginx`).
* The **Service controller** creates/maintains **EndpointSlices** that list the matching Pods’ IPs and ports, including their **Ready** condition (from readiness probes).
* **kube-proxy** on each node programs the OS (iptables/ipvs) so that packets to the Service’s virtual IP get **distributed** to the backing Pod IPs.
* **CoreDNS** gives you a stable **DNS name** for the Service:
  `nginx-svc.default.svc.cluster.local` (and the short form `nginx-svc` from the same namespace).

### Why start with ClusterIP?

**ClusterIP** is the default Service type—reachable **only inside** the cluster. It’s what your microservices use to talk to each other. We’ll prove it end-to-end, and we’ll also show how selectors, readiness, and endpoints tie together.

---

## 2) Step-by-Step Hands-On Setup (ClusterIP)

We’ll keep files tidy in a new folder but stay in the **default namespace**.

```bash
cd ~
mkdir -p ~/k8s-labs/services/clusterip
cd ~/k8s-labs/services/clusterip
```

### 2.1 Reuse your Deployment (with probes)

> If your Deployment is already running from the previous lab, you can still re-apply for consistency.

```bash
touch 00-nginx-deploy.yaml
vi 00-nginx-deploy.yaml
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
            initialDelaySeconds: 2
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 80
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

Apply and verify:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

**What we did & why:** We stood up two healthy NGINX Pods. Readiness ensures only serving Pods are considered “Ready”—that becomes important when Services decide where to send traffic.

**Quick recap:** Two Pods are up, labeled `app=nginx`, and listening on port 80. Perfect candidates for a Service.

---

### 2.2 Create the ClusterIP Service

```bash
touch 01-nginx-svc-clusterip.yaml
vi 01-nginx-svc-clusterip.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx            # <- matches the Pods from the Deployment
  ports:
    - name: http
      port: 80            # <- Service's stable virtual port
      targetPort: 80      # <- containerPort on the Pod
  type: ClusterIP         # <- default; internal-only access
```

Apply and inspect:

```bash
kubectl apply -f 01-nginx-svc-clusterip.yaml
kubectl get svc nginx-svc -o wide
kubectl get endpoints nginx-svc -o wide
kubectl describe endpoints nginx-svc | sed -n '1,200p'
kubectl get endpointslice -l kubernetes.io/service-name=nginx-svc
```

**What each field means:**

* `selector`: This is the glue—only Pods with `app: nginx` are candidates.
* `ports.port`: The port your **clients** use when they talk to the Service.
* `ports.targetPort`: The port on the **container** that actually handles the traffic.
* `type: ClusterIP`: Kubernetes allocates a **virtual IP** inside the cluster (see `CLUSTER-IP` in `kubectl get svc`).

**How readiness shows up here:**
In `Endpoints`/`EndpointSlices`, each Pod IP has a **ready** condition. Kube-proxy will not route traffic to endpoints that aren’t Ready—so your readiness probe directly affects traffic.

**Let’s recap:** The Service now has a stable IP and DNS name, and it knows which Pods to send traffic to via the selector. You can see the backend Pod IPs in `endpoints`.

---

### 2.3 Call the Service from inside the cluster

We’ll use a tiny one-off BusyBox Pod as a client.

```bash
kubectl run tester --image=busybox:1.36 --restart=Never --command -- sh -c "sleep 3600"
kubectl get pod tester
```

Call the Service **by name** and **by ClusterIP**:

```bash
# By DNS name (short form works inside the same namespace)
kubectl exec -it tester -- sh -c "wget -qO- http://nginx-svc | head -n 5"

# By ClusterIP (replace the IP you saw in 'kubectl get svc')
kubectl exec -it tester -- sh -c "wget -qO- http://<CLUSTER_IP> | head -n 5"

# Resolve DNS for the Service (BusyBox has nslookup)
kubectl exec -it tester -- nslookup nginx-svc
kubectl exec -it tester -- nslookup nginx-svc.default.svc.cluster.local
```

**What’s happening:**

* DNS `nginx-svc` resolves to the Service’s ClusterIP.
* Requests to that IP/port are distributed to the Ready Pod IPs listed in the Endpoints/EndpointSlices.

**Let’s recap:** You reached your app using the **Service name**—no Pod IPs required. That’s the day-to-day developer experience inside Kubernetes.

---

### 2.4 Prove selectors matter (break, observe, fix)

Now, temporarily break the selector so it matches **no** Pods.

```bash
cp 01-nginx-svc-clusterip.yaml 02-nginx-svc-selector-broken.yaml
vi 02-nginx-svc-selector-broken.yaml
```

Change the selector:

```yaml
spec:
  selector:
    app: does-not-match
```

Apply and check endpoints:

```bash
kubectl apply -f 02-nginx-svc-selector-broken.yaml
kubectl get endpoints nginx-svc -o wide
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

You should see **no addresses**. Try to call the Service:

```bash
kubectl exec -it tester -- sh -c "wget -S -O- http://nginx-svc 2>&1 | head -n 20"
```

Now fix it:

```bash
kubectl apply -f 01-nginx-svc-clusterip.yaml
kubectl get endpoints nginx-svc -o wide
kubectl exec -it tester -- sh -c "wget -qO- http://nginx-svc | head -n 5"
```

**Why we did this:** Services are dumb without selectors. No label match → no endpoints → nowhere to send traffic.

**Let’s recap:** The **label/selector contract** is fundamental. Your Deployment sets labels on Pods; your Service selects by those labels.

---

### 2.5 See readiness influence (quick drill)

Make one Pod temporarily **Not Ready** by changing readiness to a bad path and rolling only one replica—then restore it. This shows how “Not Ready” removes a Pod from traffic.

1. Patch Deployment to 1 replica (keeps output compact):

```bash
kubectl scale deploy/nginx-deployment --replicas=1
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

2. Break readiness **briefly** (bad path), apply, observe `Ready=False`, then fix:

```bash
# Save current spec
kubectl get deploy nginx-deployment -o yaml > 03-deploy-backup.yaml

# Create a quick patch file
cat > 03-readiness-broken.yaml <<'EOF'
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
EOF

kubectl patch deploy nginx-deployment --type merge --patch-file 03-readiness-broken.yaml
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx
kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,160p'

# Check endpoints (pod should be marked not ready and excluded from routing)
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

3. Restore the original deployment:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

**What you observed:** When readiness failed, the Pod stayed running but stopped being considered Ready for the Service. Restoring readiness put it back into rotation.

**Let’s recap:** Readiness gates **traffic**; Services use that signal to avoid sending requests to Pods that aren’t ready.

---

## 3) Verification & Troubleshooting

* **`kubectl get endpoints nginx-svc` is empty**
  Likely a selector/label mismatch. Check:

  ```bash
  kubectl get pods -l app=nginx -o wide
  kubectl describe svc nginx-svc | sed -n '1,160p'
  ```
* **Service exists, endpoints exist, but calls fail**

  * The Pod might not be Ready yet (check `kubectl describe pod …` conditions).
  * The container might not be listening on `targetPort` (ensure `containerPort: 80` and `targetPort: 80` align).
* **DNS name doesn’t resolve inside cluster**
  Ensure CoreDNS is healthy:

  ```bash
  kubectl -n kube-system get pods -l k8s-app=kube-dns
  ```
* **BusyBox lacks a tool**
  Use `wget -qO-` for HTTP, and `nslookup` for DNS. If you need `curl` or `dig`, run a different image (e.g., `curlimages/curl:8.7.1`).

---

## 4) Clean Up (optional)

```bash
kubectl delete svc nginx-svc
kubectl delete pod tester
# leave the Deployment for the next lab, or delete it too:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **ClusterIP** gives your app a **stable, internal-only** virtual IP and DNS name.
* A Service **selects** Pods via labels and **load-balances** to **Ready** endpoints.
* **EndpointSlices** track which Pod IPs are behind the Service (and their readiness); **kube-proxy** handles the actual traffic steering.
* You proved the full path:

  * Deployed Pods with probes
  * Created a ClusterIP Service
  * Resolved and called it by name/IP from a client Pod
  * Broke and fixed the selector
  * Saw readiness control which Pods receive traffic

Next, we’ll create a **NodePort** Service to make the same app reachable from **outside** the cluster—still using the exact Deployment you already understand.
