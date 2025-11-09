# **Guided Lab 04: Auto-Scaling**

## **Real-World Pain Running Containers**
1. Containers are tied to a **single host**
2. No built-in **self-healing**
3. No Production **Deployment/rollback** mechanism
> 4. No horizontal scaling (**Auto Scaling**)
5. No **load balancing, service discovery, stable networking, decouples frontend and backend, external access**
6. Lack of **Enterprise-Grade** Features
---

## **Why Auto Scaling Matters**

Imagine your app starts getting a spike in traffic — hundreds or thousands of users hitting it at once at 2:13 AM. You don’t want to scale it manually, right? 

Kubernetes should add it—**without you**. And when traffic is low, it should shrink back pods to save money. That’s **auto-scaling.*”

You also don’t want to keep checking if all Pods are running.

**Auto-healing:** replace failed Pods to **keep the desired count**.
**Auto-scaling:** **change the desired count** based on load (up and down).

> Together, they keep your app both reliable and right-sized.”

> Kubernetes can **automatically scale** your app up (or down) and **automatically heal** failed Pods — without you doing anything.

Let’s see it in action.

---

## **Step 1: Set Up Your Lab Directory**

```bash
cd ~/k8s-labs
mkdir -p autoscaling
cd autoscaling
```

---

## **Part 1: Auto-Healing – Built-in by Design**

### Step 1: Use a basic Deployment again

Create a file:

```bash
touch autohealing-deployment.yaml
vi autohealing-deployment.yaml
```

Paste this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-autohealing
  labels:
    app: autoheal
spec:
  replicas: 2
  selector:
    matchLabels:
      app: autoheal
  template:
    metadata:
      labels:
        app: autoheal
    spec:
      containers:
        - name: nginx
          image: nginx
```

Apply it:

```bash
kubectl apply -f autohealing-deployment.yaml
kubectl get pods
```

### Step 2: Simulate a failure

Delete a Pod:

```bash
kubectl delete pod <name-of-any-pod>
```

Watch what happens:

```bash
kubectl get pods
```

Kubernetes **automatically replaces the deleted Pod** to maintain the desired number of replicas — that’s **auto-healing**, and it’s enabled by default via Deployments and ReplicaSets.

---

## **Part 2: Auto-Scaling with Horizontal Pod Autoscaler (HPA)**

### What is Horizontal Pod Autoscaler (HPA)?

A Horizontal Pod Autoscaler (HPA) is a Kubernetes **feature** that automatically adjusts the number of pods in a deployment, replica set, or stateful set to match the current workload. 

This is where Kubernetes will **scale your Pods based on CPU usage**.

We’ll simulate it with a special image that generates CPU load.

### Step 1: Create the file

```bash
touch hpa-deployment.yaml
vi hpa-deployment.yaml
```

Paste this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
        - name: cpu-stress
          image: vish/stress
          args:
            - -cpus
            - "1"
          resources:
            limits:
              cpu: "200m"
            requests:
              cpu: "100m"
```

Apply it:

```bash
kubectl apply -f hpa-deployment.yaml
```

---

### Step 2: Enable the autoscaler

### Enable metrics-server for MiniKube

You’ll need the **metrics server** running in your cluster. If not installed, follow this (for Minikube):

```bash
minikube addons enable metrics-server
```

### Enable metrics-server for Kind

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system \
  --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--kubelet-insecure-tls"}]'
```

#### Verify Metrics Server is running

```bash
kubectl get deployment metrics-server -n kube-system
```

Make sure the READY column shows 1/1.

Now create the HPA:

## **Option 1: Imperative Command (Quick & Ad-Hoc)**

```bash
kubectl autoscale deployment cpu-demo --cpu-percent=50 --min=1 --max=5
```

### When to use:

* Quick testing or debugging
* Live demos or labs
* Ad-hoc changes in dev environments

---

## **Option 2: Declarative YAML (Production Best Practice)**

### Step 1: Create the YAML file

```bash
touch hpa-cpu-demo.yaml
vi hpa-cpu-demo.yaml
```

Paste this:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

Apply it:

```bash
kubectl apply -f hpa-cpu-demo.yaml
```

Check status:

```bash
kubectl get hpa
kubectl describe hpa cpu-demo-hpa

kubectl delete hpa cpu-demo
```

---

### Step 3: (Optional) Manually Generate CPU Load

To simulate high CPU load and trigger scaling:

```bash
kubectl exec -it <pod-name> -- /bin/sh
# Inside the container:
yes > /dev/null
```

Then check again:

```bash
kubectl get hpa
kubectl get pods
```

You’ll see Kubernetes **automatically increasing the number of Pods** based on CPU stress.

---

## **Wrap-Up: What Did You Learn?**

* **Auto-Healing** is built into Deployments via ReplicaSets
* Kubernetes replaces failed Pods without any manual action
* **Horizontal Pod Autoscaler (HPA)** lets you define CPU-based rules to auto-scale Pods
* You can scale between a minimum and maximum range — automatically
* All of this is done with **zero human intervention**
