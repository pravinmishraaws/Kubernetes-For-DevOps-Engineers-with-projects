# Section 01 — Introduction

Welcome! This section warms you up for Kubernetes by setting the context:
- Why plain Docker isn’t enough for real production systems
- A quick timeline of containers (cgroups → Docker → orchestrators)
- What problems Kubernetes actually solves

## Who this is for
DevOps/Cloud newcomers who know basic Linux and Docker commands.

## Prerequisites
- Docker installed (any OS)
- Basic terminal skills

## Outcomes
By the end of this section you can:
- Explain the limits of “single-host Docker”
- Describe, at a high level, how containers evolved
- Name the core problems Kubernetes solves (scheduling, self-healing, scaling, service discovery)

## How to use this section
1) Watch the three short theory videos below (in order).  
2) Skim the notes and complete the reflection.  
3) Mark the self-check and move to Section 02.

## Videos (watch in order)
- **The Limitations of Docker Containers** — *14:13*  
  ▶️ YouTube: [Day 01 -  Why Kubernetes (Not Just Docker) — The Real Reason Prod Apps Break](https://youtu.be/h3My7YGOE0k)
- **History of Containers** — *16:49*  
  ▶️ YouTube: [Day 02 -  From chroot to Kubernetes: 40 Years of Containers in 10 Minutes](https://youtu.be/7zIoVb2g7do)
- **Why Kubernetes Became Essential** — *~10 min*  
  ▶️ YouTube: [Day 03 - Why Kubernetes Is Essential for Microservices (Monolith → Microservices → K8s)](https://youtu.be/G2PhGlfmNx8)

**Full playlist:** [Kubernetes Zero to Hero with Projects](https://youtube.com/playlist?list=PLVOdqXbCs7bUOYZcgEZYCbg_kolYzQN93&si=XwUYPTWazvCvK0-A)

---

## Self-Reflection & Discussion (post as a YouTube comment)
Write 4–6 lines answering:
1. In one sentence, what is the **biggest limitation** of “Docker-only” you noticed?  
2. Name **two Kubernetes features** that directly address that limitation.  
3. Share **one real scenario** (from your work or imagination) where auto-healing or scaling would help.  
4. What’s **one thing you still find unclear** and want explained in the next section?

**Optional stretch (1–2 lines):**  
If you were designing a product today, what **non-K8s alternative** (managed PaaS, serverless, etc.) might solve the same problem—and why would you still choose or skip Kubernetes?

---

## Self-Check (tick before moving on)
- [ ] I can explain why Docker alone isn’t enough for production.  
- [ ] I can state what “desired state” means in Kubernetes.  
- [ ] I can map a real problem to a Kubernetes primitive (e.g., scaling → HPA).  

**Next:** Continue to **Section 02 — Introduction to Kubernetes**.
