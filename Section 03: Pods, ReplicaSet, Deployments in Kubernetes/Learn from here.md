# Section 03 — Pods, ReplicaSet, Deployments in Kubernetes

In this section you’ll go from a **single Pod** to **self-healing ReplicaSets** and **zero-downtime Deployments**.

## Prerequisites
- A local cluster (minikube or kind) from Section 02
- `kubectl` configured

## Outcomes
By the end, you can:
- Define and run a **Pod** from YAML
- Use a **ReplicaSet** to keep N copies alive (self-healing)
- Ship new versions safely with a **Deployment** (rolling updates + rollback)

## Videos (watch in order)
- **Pod in Kubernetes** — *13:06*  
  ▶️ [Day 10 - Pods in Kubernetes: What They Are, Why They Exist, and How to Create Them](https://youtu.be/RZyE_fKFOxA)
- **Creating Your First Pod in Kubernetes** — *13:11*  
  ▶️ [Day 11 - Create Your First Kubernetes Pod (Imperative vs Declarative, Step-by-Step)](https://youtu.be/dXDo1PCYCd0)
- **Guided Lab 02: ReplicaSets – Pod Auto-Heals** — *15:33*  
  ▶️ [Day 12 - ReplicaSet in Kubernetes: Self-Healing Pods & Easy Scaling (But Why Prod Uses Deployments)](https://youtu.be/PxORg73EHJg)
- **Guided Lab 03: Deployments – From Basics to Rolling Updates** — *17:29*  
  ▶️ [Day 13 - Kubernetes Deployments: Zero-Downtime Releases, Rollbacks, and Scaling (The Right Way)](https://youtu.be/d9C1O3OUEbw)

**Full playlist:** [Kubernetes Zero to Hero with Projects](https://youtube.com/playlist?list=PLVOdqXbCs7bUOYZcgEZYCbg_kolYzQN93&si=upgzcKckCxQEE75p)

## Self-Reflection (post as a YouTube comment)

In 4–6 lines:

1. What’s the key difference between **Pod**, **ReplicaSet**, and **Deployment** in your own words?
2. When would you use a ReplicaSet directly vs always a Deployment?
3. Describe one scenario where **rollback** saves you in production.
4. What’s still unclear that you want explained before **Services**?

## Self-Check

* [ ] I can create a Pod from YAML and see it run.
* [ ] I can prove a ReplicaSet **auto-heals** by deleting a pod.
* [ ] I can perform a Deployment **rolling update** and **rollback**.

**Next:** *Section 04 — Kubernetes Services* (expose your Pods cleanly).
