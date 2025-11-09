# Guided Lab 07 — Kubernetes Services (NodePort)

## 1) Introduction — Why NodePort?

ClusterIP solved **service-to-service** communication **inside** the cluster. But sometimes you want a quick way to reach your app from **outside**—for demos, simple on-prem setups, or when a cloud LoadBalancer isn’t available yet.

That’s what **NodePort** provides.

### What is a NodePort Service?

* It’s a Service that **opens a high port (30000–32767)** on **every node**.
* Requests to `http://<nodeIP>:<nodePort>` are **routed** to the selected Pods.
* Internally, it’s still a Service with a stable ClusterIP and selector; NodePort is just an **additional door** that exposes it via each node’s IP.

### How does it work behind the scenes?

* You define `type: NodePort` and a port mapping (`port` → `targetPort`), plus an optional fixed `nodePort`.
* The Service controller and EndpointSlices track the matching Pods (and their readiness).
* **kube-proxy** programs node networking so traffic hitting `<nodeIP>:<nodePort>` is forwarded to the Service and load-balanced across **Ready** endpoints.

We’ll keep things practical: reuse your Deployment, create a NodePort Service, and hit it from outside (when possible) and from inside (always possible).

---

## 2) Step-by-Step Hands-On Setup (NodePort)

Organize files:

```bash
cd ~
mkdir -p ~/k8s-labs/services/nodeport
cd ~/k8s-labs/services/nodeport
```

### 2.1 Reuse your Deployment (with probes)

> If it’s already running from the previous lab, re-applying is harmless and keeps everyone in sync.

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

**Recap:** Two healthy NGINX Pods are running with the label `app=nginx`. These labels are how Services find the right backends.

---

### 2.2 Create a NodePort Service

We’ll choose a predictable external port: **30080**.

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
      port: 80         # Service's port (virtual, cluster-internal)
      targetPort: 80   # containerPort exposed by the Pod
      nodePort: 30080  # fixed external port opened on every node (30000-32767)
```

Apply and inspect:

```bash
kubectl apply -f 01-nginx-svc-nodeport.yaml
kubectl get svc nginx-svc-nodeport -o wide
kubectl get endpoints nginx-svc-nodeport -o wide
```

**What those fields do:**

* `type: NodePort` — exposes the Service via a port on each node.
* `nodePort: 30080` — the exact port you’ll curl on the node IP(s).
* `port: 80` — the Service’s own virtual port.
* `targetPort: 80` — the actual container port inside the Pod.

**Recap:** You now have three “layers”: Node’s 30080 → Service port 80 → Pod’s 80.

---

### 2.3 Find a reachable node IP

List nodes and note their IPs:

```bash
kubectl get nodes -o wide
```

* On cloud or real VMs: use the **EXTERNAL-IP** if present; otherwise try the **INTERNAL-IP** from a machine that can reach that network.
* On **Minikube**: prefer the helper command below.
* On **kind / Docker Desktop**: node IPs often aren’t directly reachable from your host OS; see the alternatives.

---

### 2.4 Test the NodePort

#### A) From your laptop (best when the node IP is reachable)

Replace `<NODE_IP>` with a node’s reachable IP:

```bash
curl -I http://<NODE_IP>:30080
curl http://<NODE_IP>:30080 | head -n 10
```

You should see the NGINX default page headers/body.

#### B) From inside the cluster (always works)

Start a quick client Pod:

```bash
kubectl run tester --image=busybox:1.36 --restart=Never --command -- sh -c "sleep 3600"
kubectl get pod tester
```

Exec and curl the node IP:

```bash
kubectl exec -it tester -- sh -c "wget -qO- http://<NODE_IP>:30080 | head -n 10"
```

**If your environment blocks direct host access:**

* **Minikube**:

  ```bash
  minikube service nginx-svc-nodeport --url
  ```

  Minikube sets up routing to the NodePort and prints a usable URL.

* **kind / Docker Desktop**: Node IPs may be in an internal Docker network. Use either:

  * Port-forward (not NodePort, but proves the app works):

    ```bash
    kubectl port-forward svc/nginx-svc-nodeport 8080:80
    curl http://localhost:8080
    ```
  * Or test from **inside** the cluster using the `tester` pod as shown.

**Recap:** You’ve confirmed external (or internal) reachability at `<nodeIP>:30080`. NodePort acts as a simple, infrastructure-agnostic way to expose a Service beyond the cluster.

---

### 2.5 Quick selector drill (optional)

Let’s briefly prove that the NodePort is just a door; without matching Pods, it has nowhere to send traffic.

Break the selector:

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

Apply and check endpoints:

```bash
kubectl apply -f 02-nginx-svc-selector-broken.yaml
kubectl get endpoints nginx-svc-nodeport -o wide
```

Try to curl again:

```bash
# Replace <NODE_IP> again
curl -I http://<NODE_IP>:30080 || true
```

Fix it:

```bash
kubectl apply -f 01-nginx-svc-nodeport.yaml
kubectl get endpoints nginx-svc-nodeport -o wide
```

**Recap:** NodePort exposes the Service, but **Endpoints** still depend on **labels** and **readiness**. No matching, Ready Pods → no real backend.

---

### 2.6 (Optional) Prove resilience during pod churn

Delete a Pod and keep curling:

```bash
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD"
kubectl get pods -l app=nginx -w
# In a separate terminal:
curl -I http://<NODE_IP>:30080
```

You’ll see one Pod terminate and another appear, yet the NodePort endpoint stays usable as long as at least one Pod is Ready.

**Recap:** Services mask Pod churn. That’s the abstraction your users and other services rely on.

---

## 3) Verification & Troubleshooting

* **`curl <NODE_IP>:30080` times out from your laptop**

  * Firewalls or NAT may block access. Try from inside the cluster (`tester` pod) or use `minikube service … --url`.
  * Confirm the Service and endpoints:

    ```bash
    kubectl get svc nginx-svc-nodeport -o wide
    kubectl get endpoints nginx-svc-nodeport -o wide
    ```
* **`endpoints` is empty**

  * Label mismatch or Pods not Ready. Check:

    ```bash
    kubectl get pods -l app=nginx -o wide
    kubectl describe svc nginx-svc-nodeport | sed -n '1,160p'
    kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,200p'
    ```
* **Head requests succeed but body fails intermittently**

  * One Pod may be failing readiness. Watch Pod conditions and restart counts; ensure `targetPort` matches the containerPort actually in use.

---

## 4) Clean Up (optional)

```bash
kubectl delete svc nginx-svc-nodeport
kubectl delete pod tester
# Keep the Deployment for future labs, or remove it:
# kubectl delete deploy nginx-deployment
```

---

## Wrap-Up — What Did You Learn?

* **NodePort** exposes a Service externally at `<nodeIP>:<nodePort>` on **every node**.
* It layers on top of the same fundamentals:

  * **Selectors** connect Services to Pods.
  * **EndpointSlices** list backend Pod IPs (and readiness).
  * **kube-proxy** steers traffic from NodePort → Service → Ready Pods.
* You verified reachability from both **outside** (where possible) and **inside** the cluster.
* You saw how the NodePort “door” is useless without **matching, Ready** endpoints—and how the Service abstracts Pod churn during restarts.

**What’s next:** When you move to cloud (e.g., AKS), a `type: LoadBalancer` Service provisions an actual cloud load balancer and public IP—same Deployment, same selectors, just a different exposure method. That will be our short, cloud-variation lab after Services.
