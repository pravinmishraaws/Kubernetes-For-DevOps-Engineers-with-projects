# Guided Lab 07 — Kubernetes Services (ClusterIP)

> Scope: **ClusterIP** only. Default namespace. We’ll connect the dots between “healthy Pods” and “how anything actually reaches them,” then prove it end-to-end with hands-on drills.

---

## 1) Introduction — You’ve Built a Smart, Reliable Deployment… Now Who Finds It?

You’ve already done a lot right:

* **Replicas (2):** Kubernetes keeps two Pods alive.
* **Resources:** Requests/limits prevent CPU starvation.
* **Rolling updates:** No downtime during upgrades.
* **Readiness:** New Pods don’t receive traffic until they’re ready.
* **Liveness:** Wedged processes self-heal via restarts.

That’s an excellent foundation. But there’s a practical snag that appears the moment **another Pod** tries to talk to your app:

### The Real-World Problem

* **Pod IPs are ephemeral.** Scale up? Roll out? Crash/restart? Pod IPs **change**.
* A teammate hard-codes a Pod IP to test something… ten minutes later it breaks.
* Your system works in isolation, but **service-to-service communication** is flaky or manual.

We need a stable way to **find** and **reach** the right Pods—only those that are **Ready**—no matter how often they churn.

### The Concept — What a Service Is (Practically)

A **Service** is a *stable front door* for a set of Pods:

* It owns a **stable virtual IP** (the *ClusterIP*) and a **stable DNS name**.
* It uses a **label selector** (e.g., `app: nginx`) to decide which Pods sit behind the door.
* It **load-balances** across *Ready* endpoints, so callers don’t need to care which specific Pod serves them.

You’ll create one object—**a Service**—and every Pod in your namespace can reach your app at a single, unchanging name, e.g. `http://nginx-svc`.

### How It Works (Just Enough Detail)

* You define the Service with a **selector**.
* Kubernetes maintains **EndpointSlices** (lists of Pod IP:Port) that match the selector and reflect each Pod’s **Ready** state.
* **kube-proxy** on each node programs the data path (iptables/ipvs) so traffic to the Service’s **ClusterIP** is distributed to the backend Pod IPs.
* **CoreDNS** gives your Service a DNS name:
  `nginx-svc.default.svc.cluster.local` (and the short `nginx-svc` from the same namespace).

### Why Start with ClusterIP?

It’s the default type—**internal-only**. Exactly what your microservices use to talk to each other inside the cluster. We’ll prove discovery, DNS, and readiness-aware routing in a tight loop.

---

## 2) Step-by-Step Hands-On (ClusterIP)

We’ll keep files tidy but stay in the **default namespace**.

```bash
cd ~
mkdir -p ~/k8s-labs/services/clusterip
cd ~/k8s-labs/services/clusterip
```

### 2.1 Reuse Your Deployment (probes included)

> If it’s already applied from previous labs, re-applying won’t hurt—this keeps everyone aligned.

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

Apply and check:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

**Why this matters:** Labels (`app: nginx`) on the Pod template are the “hook” your Service will use. Readiness ensures only good Pods are considered *Ready*.

**Let’s recap:** Two healthy Pods with stable labels, listening on port 80. Perfect backends for a Service.

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
    app: nginx
  ports:
    - name: http
      port: 80         # Service's stable, virtual port
      targetPort: 80   # Pod containerPort
  type: ClusterIP      # internal-only
```

Apply and inspect:

```bash
kubectl apply -f 01-nginx-svc-clusterip.yaml
kubectl get svc nginx-svc -o wide
kubectl get endpoints nginx-svc -o wide
kubectl describe endpoints nginx-svc | sed -n '1,200p'
kubectl get endpointslice -l kubernetes.io/service-name=nginx-svc
```

**What you’re seeing:**

* `CLUSTER-IP` — the Service’s virtual IP.
* `Endpoints` / `EndpointSlices` — the current list of matching Pod IPs and ports (and their **Ready** state).

**Let’s recap:** You now have a *single* front door (`nginx-svc`) that Kubernetes keeps pointing at the right Pods as they come and go.

---

### 2.3 Call the Service from Inside the Cluster

Spin up a tiny client Pod and make requests by **name** and by **ClusterIP**.

```bash
kubectl run tester --image=busybox:1.36 --restart=Never --command -- sh -c "sleep 3600"
kubectl get pod tester
```

Now test:

```bash
# By DNS name (short form, same namespace)
kubectl exec -it tester -- sh -c "wget -qO- http://nginx-svc | head -n 5"

# By ClusterIP (replace with the IP shown in 'kubectl get svc')
kubectl exec -it tester -- sh -c "wget -qO- http://<CLUSTER_IP> | head -n 5"

# DNS lookups via CoreDNS
kubectl exec -it tester -- nslookup nginx-svc
kubectl exec -it tester -- nslookup nginx-svc.default.svc.cluster.local
```

**Why this proves the point:** Your client doesn’t care which Pod serves the request—only that `nginx-svc` is stable. Kubernetes does the rest.

**Let’s recap:** You’ve consumed your app exactly how other services will—by name, not by Pod IP.

---

### 2.4 Show the Label/Selector Contract (Break → Observe → Fix)

If the selector doesn’t match labels, the Service has nowhere to send traffic.

```bash
cp 01-nginx-svc-clusterip.yaml 02-nginx-svc-selector-broken.yaml
vi 02-nginx-svc-selector-broken.yaml
```

Change:

```yaml
spec:
  selector:
    app: does-not-match
```

Apply and check:

```bash
kubectl apply -f 02-nginx-svc-selector-broken.yaml
kubectl get endpoints nginx-svc -o wide
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

Try to call it:

```bash
kubectl exec -it tester -- sh -c "wget -S -O- http://nginx-svc 2>&1 | head -n 20"
```

Now fix it:

```bash
kubectl apply -f 01-nginx-svc-clusterip.yaml
kubectl get endpoints nginx-svc -o wide
kubectl exec -it tester -- sh -c "wget -qO- http://nginx-svc | head -n 5"
```

**Let’s recap:** Services are simple but strict—**no label match, no endpoints**. Your Deployment owns labels; your Service selects them.

---

### 2.5 See Readiness Influence on Routing (Quick Drill)

We’ll momentarily break readiness so a Pod is **NotReady**, and then observe how the Service reflects that.

1. Keep things concise by scaling to one replica:

```bash
kubectl scale deploy/nginx-deployment --replicas=1
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

2. Patch readiness to a bad path → watch it fall out of rotation:

```bash
# Save current spec (for easy restore)
kubectl get deploy nginx-deployment -o yaml > 03-deploy-backup.yaml

# Create a patch that breaks readiness
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

# Endpoints should now exclude this Pod (NotReady)
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

3. Restore readiness:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl describe endpoints nginx-svc | sed -n '1,200p'
```

**What you learned:** Readiness isn’t academic—Services **respect** it. NotReady Pods don’t receive traffic.

**Let’s recap:** Readiness gates traffic; the Service’s endpoint list mirrors that truth in real time.

---

## 3) Verification & Troubleshooting

* **Endpoints empty?**
  Selector mismatch or no Ready Pods.

  ```bash
  kubectl get pods -l app=nginx -o wide
  kubectl describe svc nginx-svc | sed -n '1,160p'
  kubectl describe endpoints nginx-svc | sed -n '1,200p'
  ```

* **Service exists, but requests fail intermittently**

  * Pod might be flapping on readiness—check `Conditions` and `RESTARTS`:

    ```bash
    kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
    kubectl get pods -l app=nginx
    ```
  * Ensure `targetPort` matches `containerPort`.

* **DNS doesn’t resolve**
  Verify CoreDNS:

  ```bash
  kubectl -n kube-system get pods -l k8s-app=kube-dns
  ```

* **BusyBox limitations**
  Use `wget` and `nslookup`. If you need `curl`/`dig`, run a different toolbox image (e.g., `curlimages/curl`).

---

## 4) Clean Up (optional)

```bash
kubectl delete pod tester
kubectl delete svc nginx-svc
# Keep the Deployment for the next lab (NodePort), or remove it:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **ClusterIP** gives your app a **stable, internal** IP and DNS name—no more chasing Pod IPs.
* **Selectors** connect Services to Pods; **labels** make the contract explicit.
* **EndpointSlices + kube-proxy** steer traffic only to **Ready** Pods.
* You proved the full path:

  * Reused a healthy Deployment with probes
  * Created a ClusterIP Service
  * Reached it by DNS and IP from a client Pod
  * Broke and fixed the selector
  * Saw readiness immediately reflected in Service endpoints

**Next up:** we’ll expose the same app **outside** the cluster with **NodePort**—same labels, same Pods, just a different door.
