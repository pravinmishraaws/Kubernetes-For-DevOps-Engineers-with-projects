# Guided Lab 08 — Kubernetes Services (NodePort)

> Scope: **NodePort** only. Default namespace. We’ll take the same NGINX Deployment (with readiness & liveness) and open a door to it **from outside the cluster**. No LoadBalancer/Ingress yet—that’s next.

---

## 1) Introduction — You Have a Stable Internal Service… But Can Users Reach It?

You’ve already solved the **inside-the-cluster** story:

* **Replicas (2):** your app survives pod churn.
* **Resources:** no CPU starvation.
* **Rolling updates:** safe upgrades.
* **Readiness & Liveness:** availability and self-healing.
* **ClusterIP Service:** stable name + internal load balancing.

But now a practical, real-world question shows up:

> “I want to show this to someone outside the cluster. How do I reach it from my laptop or another network?”

You could port-forward for demos, but that’s a developer convenience, not a deployment model. You need a **supported, simple way** to open an external door without involving a cloud load balancer yet.

### What NodePort Is (Practically)

A **NodePort Service** opens a high port (default range **30000–32767**) on **every node** in your cluster. Traffic to
`http://<nodeIP>:<nodePort>` is forwarded to your Service, which then load-balances to the matching, **Ready** Pods.

* It’s still a Kubernetes **Service** (with ClusterIP, selector, endpoints).
* **NodePort** adds an extra **external entry point** on each node.

### When to Use NodePort

* Quick external access in labs or on-prem clusters **without** a cloud LoadBalancer.
* Simple, predictable exposure when you control the node network and firewall.
* A stepping stone before **LoadBalancer** (AKS/EKS/GKE) or **Ingress**.

We’ll create a NodePort Service, discover a node IP, and test from outside (when reachable) and from inside (always reachable).

---

## 2) Step-by-Step Hands-On (NodePort)

Keep things tidy:

```bash
cd ~
mkdir -p ~/k8s-labs/services/nodeport
cd ~/k8s-labs/services/nodeport
```

### 2.1 Reuse Your Deployment (probes included)

> If it’s already running, re-apply anyway so everyone’s in sync.

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

**Let’s recap:** Healthy Pods with stable labels (`app=nginx`) are ready to sit behind a NodePort door.

---

### 2.2 Create the NodePort Service

We’ll pick a fixed, easy-to-remember **nodePort: 30080**.

```bash
touch 01-nginx-svc-nodeport.yaml
vi 01-nginx-svc-nodeport.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      port: 80         # Service's virtual port (ClusterIP)
      targetPort: 80   # Pod's containerPort
      nodePort: 30080  # External port open on every node (30000-32767)
```

Apply and inspect:

```bash
kubectl apply -f 01-nginx-svc-nodeport.yaml
kubectl get svc nginx-svc-nodeport -o wide
kubectl get endpoints nginx-svc-nodeport -o wide
```

**What each field does**

* `type: NodePort` → exposes via `<nodeIP>:<nodePort>`.
* `nodePort: 30080` → predictable external port on **every** node.
* `port: 80` → Service’s own virtual port.
* `targetPort: 80` → where the container actually listens.

**Let’s recap:** You’ve added an **external door** to the existing Service mechanics.

---

### 2.3 Find a Reachable Node IP

```bash
kubectl get nodes -o wide
```

* Use **EXTERNAL-IP** if available; otherwise **INTERNAL-IP** from a machine that can reach the node network.
* **Minikube:** prefer the helper command shown below.
* **kind / Docker Desktop:** node IPs may not be directly reachable from your host; see alternatives below.

---

### 2.4 Test the NodePort

#### A) From your laptop (preferred if node is reachable)

```bash
# Replace <NODE_IP> with a reachable node IP
curl -I http://<NODE_IP>:30080
curl     http://<NODE_IP>:30080 | head -n 10
```

You should see NGINX headers and the default HTML body.

#### B) From inside the cluster (always works)

Create a tiny client pod:

```bash
kubectl run tester --image=busybox:1.36 --restart=Never --command -- sh -c "sleep 3600"
kubectl get pod tester
```

Call via node IP:

```bash
kubectl exec -it tester -- sh -c "wget -qO- http://<NODE_IP>:30080 | head -n 10"
```

#### Environment notes

* **Minikube**:

  ```bash
  minikube service nginx-svc-nodeport --url
  ```

  This prints a usable URL that routes to the NodePort.
* **kind / Docker Desktop**:

  * Often can’t reach node IPs directly from the host. Use the `tester` pod (above), or for local demos:

    ```bash
    kubectl port-forward svc/nginx-svc-nodeport 8080:80
    curl http://localhost:8080
    ```

    (Port-forward uses the Service but is not NodePort; helpful to validate the app.)

**Let’s recap:** NodePort exposes your Service externally. Whether you hit it from your laptop depends on your environment’s networking, but it always works from inside the cluster.

---

### 2.5 Prove the Selector Contract (Optional but Valuable)

Break the selector so the Service has no backends.

```bash
cp 01-nginx-svc-nodeport.yaml 02-nginx-svc-selector-broken.yaml
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
kubectl get endpoints nginx-svc-nodeport -o wide
```

Try to call again:

```bash
curl -I http://<NODE_IP>:30080 || true
```

Fix it:

```bash
kubectl apply -f 01-nginx-svc-nodeport.yaml
kubectl get endpoints nginx-svc-nodeport -o wide
```

**Let’s recap:** NodePort is just a door. Without matching, **Ready** Pods, there’s nowhere to send traffic.

---

### 2.6 See Resilience During Pod Churn (Quick Drill)

Delete a Pod and keep calling the NodePort:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD"
kubectl get pods -l app=nginx -w
# In another terminal:
curl -I http://<NODE_IP>:30080
```

**What you’ll notice:** One Pod terminates, the Deployment recreates it, and the Service keeps serving as long as at least one Pod is Ready.

**Let’s recap:** Services abstract churn. NodePort doesn’t change that—it simply gives you an external entry point.

---

## 3) Verification & Troubleshooting

* **`curl <NODE_IP>:30080` times out from your laptop**

  * Firewall/NAT may block node access. Try from inside the cluster (`tester` pod).
  * On Minikube, use `minikube service … --url`.
  * Confirm Service and endpoints:

    ```bash
    kubectl get svc nginx-svc-nodeport -o wide
    kubectl get endpoints nginx-svc-nodeport -o wide
    ```

* **Endpoints are empty**

  * Label mismatch or Pods not Ready:

    ```bash
    kubectl get pods -l app=nginx -o wide
    kubectl describe svc nginx-svc-nodeport | sed -n '1,160p'
    kubectl describe endpoints nginx-svc-nodeport | sed -n '1,200p'
    ```

* **Intermittent failures**

  * One Pod may be NotReady or crash-looping. Check conditions and restarts:

    ```bash
    kubectl get pods -l app=nginx
    kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
    ```

* **Wrong ports**

  * Ensure `targetPort` matches `containerPort: 80`, and that your app listens on 80.

---

## 4) Clean Up (optional)

```bash
kubectl delete pod tester
kubectl delete svc nginx-svc-nodeport
# Keep the Deployment for future labs (LoadBalancer/Ingress), or remove it:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **NodePort** adds an external entry point: `<nodeIP>:<nodePort>` on **every node**.
* It builds on the same fundamentals: **labels → selector → endpoints → Ready Pods**.
* You verified reachability from both **outside** (when networking allows) and **inside** the cluster.
* You saw that NodePort is **just a door**—it’s only useful if it leads to healthy, matching backends.

**Next step:** Move to cloud exposure with **`type: LoadBalancer`** (e.g., AKS). Same Deployment and labels, but Kubernetes will provision a cloud load balancer and public IP for true internet-facing access.
