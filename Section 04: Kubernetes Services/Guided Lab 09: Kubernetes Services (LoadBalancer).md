# Guided Lab 09 — Kubernetes Services (LoadBalancer)

> Scope: **type: LoadBalancer** on a real cloud cluster (AKS). Default namespace. We’ll provision a small AKS cluster, deploy the same NGINX app (with readiness & liveness), expose it with a **public IP**, and verify end-to-end from the internet. No Ingress yet—that comes later.

---

## 1) Introduction — You Have Internal & Node-Level Access… But What About the Internet?

You’ve already covered:

* **ClusterIP** — stable, in-cluster communication by name.
* **NodePort** — quick external door via `<nodeIP>:<nodePort>` on every node.

Those are great—but product teams usually ask a different question:

> “Can our users on the **public internet** reach this service at a **stable IP/DNS** without knowing node IPs or odd ports?”

That’s where a **LoadBalancer Service** shines.

### What a LoadBalancer Service Is (Practically)

* You declare `type: LoadBalancer` on your Service.
* Kubernetes talks to the **cloud provider** (here: Azure) to **provision a cloud load balancer + public IP**.
* Traffic to that public IP is sent to your cluster and **load-balanced across Ready Pods** (through NodePorts/ClusterIP behind the scenes).

### Why It Matters

* **Stable, internet-facing endpoint** without managing external LBs yourself.
* **First production-like exposure**: real public IP, real cloud primitives.
* Same Deployment and labels. Only the Service type changes.

We’ll stand up AKS, deploy the app, create a LoadBalancer Service, wait for an **EXTERNAL-IP**, and curl it from your machine.

---

## 2) Prepare a Small AKS Cluster (One-Time for this Lab)

> Requires Azure CLI logged into an Azure subscription with permissions to create resources.

```bash
# 2.1 Create a resource group
az group create -n rg-aks-lb-lab -l westeurope

# 2.2 Create a minimal AKS cluster (1 node is fine for the lab)
az aks create -g rg-aks-lb-lab -n aks-lb-lab --node-count 1 --generate-ssh-keys --node-vm-size Standard_B2s

# 2.3 Fetch kubeconfig
az aks get-credentials -g rg-aks-lb-lab -n aks-lb-lab

# 2.4 Verify connectivity
kubectl get nodes
```

**What this does (at a glance):**

* Creates a managed Kubernetes control plane and a node pool.
* Configures your `kubectl` to talk to AKS.

**Let’s recap:** You now have a real cloud cluster where `type: LoadBalancer` can provision an Azure public IP automatically.

---

## 3) Reuse Your Deployment (Probes Included)

Keep files tidy:

```bash
cd ~
mkdir -p ~/k8s-labs/services/loadbalancer
cd ~/k8s-labs/services/loadbalancer
```

Create the Deployment:

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

Apply and confirm:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

**Why this matters:** The Deployment sets **labels** (`app: nginx`) and exposes container `port: 80`. Readiness will control which Pods are eligible for traffic.

**Recap:** Two healthy Pods, ready to sit behind a cloud load balancer.

---

## 4) Create a LoadBalancer Service

```bash
touch 01-nginx-svc-loadbalancer.yaml
vi 01-nginx-svc-loadbalancer.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - name: http
      port: 80         # Service port (public-facing)
      targetPort: 80   # Pod's containerPort
```

Apply and watch for a public IP:

```bash
kubectl apply -f 01-nginx-svc-loadbalancer.yaml

# Watch until EXTERNAL-IP is allocated (may take 30-120s)
kubectl get svc nginx-svc-lb -w
```

When `EXTERNAL-IP` shows a value (e.g., `20.101.x.y`), open a new terminal and test:

```bash
EXTERNAL_IP=$(kubectl get svc nginx-svc-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I http://$EXTERNAL_IP
curl    http://$EXTERNAL_IP | head -n 10
```

**What just happened behind the scenes:**

* AKS created an **Azure Public IP** and an **Azure Load Balancer**.
* The LB forwards to NodePorts in your cluster (wired by Kubernetes), then to **Ready** endpoints selected by your Service’s labels.

**Let’s recap:** With one YAML change (`type: LoadBalancer`), you exposed the app to the internet—no custom infra work.

---

## 5) Quick Drills to Build Intuition

### 5.1 Readiness affects real users

Break readiness briefly and observe accessibility.

```bash
# Create a patch that makes readiness fail
cat > 02-readiness-broken.yaml <<'EOF'
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

kubectl patch deploy nginx-deployment --type merge --patch-file 02-readiness-broken.yaml
kubectl rollout status deploy/nginx-deployment

# Endpoints should shrink / exclude NotReady Pods
kubectl describe endpoints nginx-svc-lb | sed -n '1,200p'

# From your machine — retry a few times
curl -I http://$EXTERNAL_IP
```

Now fix it:

```bash
kubectl apply -f 00-nginx-deploy.yaml
kubectl rollout status deploy/nginx-deployment
kubectl describe endpoints nginx-svc-lb | sed -n '1,200p'
curl -I http://$EXTERNAL_IP
```

**What you learned:** The public LoadBalancer **respects readiness** all the way down; NotReady Pods drop out of rotation.

---

### 5.2 Scale and observe load balancing

Scale up and watch endpoints grow:

```bash
kubectl scale deploy/nginx-deployment --replicas=4
kubectl rollout status deploy/nginx-deployment
kubectl get endpoints nginx-svc-lb -o wide
```

Call the external IP multiple times (you may see different `Server` headers/connection details across requests depending on client behavior and LB config, but the key is more endpoints are eligible).

**Recap:** The cloud LB fronts your Service, which fronts multiple Ready Pods—scaling feels natural to the caller.

---

## 6) Troubleshooting

* **`EXTERNAL-IP` stays `<pending>`**

  * Ensure you’re on a cloud cluster (AKS) with the Azure cloud provider enabled (the default for AKS).
  * Check Service events:

    ```bash
    kubectl describe svc nginx-svc-lb | sed -n '1,200p'
    ```
  * Verify the AKS cluster is in a ready state:

    ```bash
    az aks show -g rg-aks-lb-lab -n aks-lb-lab --query "provisioningState"
    ```

* **Public IP works intermittently**

  * Check Pod readiness and restarts:

    ```bash
    kubectl get pods -l app=nginx
    kubectl describe pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}') | sed -n '1,240p'
    ```
  * Ensure `targetPort` matches the containerPort (80).

* **Need HTTPS / custom ports**

  * For this lab, we stick to port 80. TLS/host routing belongs to **Ingress** (up next).

* **You’re not using AKS**

  * On **minikube**, you can simulate with:

    ```bash
    # Start a tunnel to allocate an external IP locally
    minikube tunnel
    kubectl apply -f 01-nginx-svc-loadbalancer.yaml
    kubectl get svc nginx-svc-lb -w
    ```
  * On bare metal, consider **MetalLB** (out of scope for this lab).

---

## 7) Clean Up (optional)

```bash
# Kubernetes objects
kubectl delete svc nginx-svc-lb
kubectl delete deploy nginx-deployment

# Azure resources
az group delete -n rg-aks-lb-lab --yes --no-wait
```

*(If you plan to do the Ingress lab next on the same cluster, keep the AKS resources and only delete the Service/Deployment.)*

---

## Wrap-Up — What Did You Learn?

* **type: LoadBalancer** gives you a **public IP** fronting your Service—no manual LB setup.
* Azure (AKS) automatically provisions the **cloud load balancer** and wires it to your cluster.
* The whole path remains **label/selector → EndpointSlices (Ready only) → kube-proxy → NodePorts → Pods**.
* You verified end-to-end from the public internet, saw **readiness** affect traffic, and experienced **scaling** behind a stable public endpoint.
