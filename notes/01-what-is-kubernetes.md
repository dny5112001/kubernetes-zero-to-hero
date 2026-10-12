# 01 · What is Kubernetes?

## The problem

You built an app and packed it into a **container** (chapter 02). Running one
container on one computer is easy: `docker run my-app`.

Now real life happens:
- 10,000 users arrive. You need **20 copies** of the app, spread over **5 servers**.
- A server dies at 3 a.m. Who **restarts** those copies somewhere else?
- You release a new version. How do you swap 20 copies **without downtime** — and **undo** it if it's broken?
- Copies keep moving between servers. How do users **find** them?

Doing this by hand doesn't scale. You need a manager.

## The answer: Kubernetes

**Kubernetes** (say "koo-ber-NET-eez", short **K8s** — 8 letters between K and s)
is a system that **runs and looks after containers across many computers** for you.

Think of it as a **ship's captain** (the name means "helmsman" in Greek):
- You tell the captain **what you want**: "keep 20 copies of my app running".
- The captain decides **where** each one goes, **watches** them all the time,
  and **fixes** things when they go wrong — day and night.

## What Kubernetes does for you

| Job | In plain words |
|---|---|
| **Scheduling** | Picks which computer each container runs on, based on free space |
| **Self-healing** | A container crashes or a computer dies → it starts a replacement |
| **Scaling** | More copies when busy, fewer when quiet — even automatically |
| **Rolling updates** | Replaces old versions with new ones a few at a time; rolls back if needed |
| **Service discovery** | Gives each app a stable name, so others can find it as copies move around |
| **Load balancing** | Spreads traffic across all the copies |
| **Config and secrets** | Keeps settings and passwords outside your image |
| **Storage** | Attaches disks that survive restarts |

## The big idea: desired state

Like Terraform, Kubernetes is **declarative**. You don't give step-by-step
orders. You describe **what you want**, in a file:

```yaml
replicas: 3      # "I want 3 copies running"
```

Kubernetes then **loops forever**:

```
   ┌─────────────────────────────────────────────┐
   │ 1. Look: how many copies are running? (2)   │
   │ 2. Compare with what you want.        (3)   │
   │ 3. Fix the difference: start 1 more.        │
   └──────────────── repeat forever ─────────────┘
```

This is called a **control loop** or **reconciliation**. It's like a
thermostat: you set the temperature, and it keeps correcting towards it.
That's why Kubernetes "heals itself" — it's always pulling reality back
to what you described.

## What Kubernetes is NOT

- It doesn't **build** your app or your container image (that's Docker / your CI).
- It's not a **cloud** — it runs *on* clouds (AWS EKS, Google GKE, Azure AKS) or your own machines.
- It's not always needed. One small app on one server? A simpler platform
  (a single VM, Render, Fly.io, AWS App Runner, Cloud Run) may be less work.
  Kubernetes shines when you have **many services**, **many copies**, or need
  **the same platform everywhere**.

## Where Terraform fits

- **Terraform** builds the *ground*: the cluster, networks, databases, storage buckets.
- **Kubernetes** runs your *apps* on that ground.

You saw both together in the [k8s-terraform-lab](https://github.com/dny5112001/k8s-terraform-lab).

## Remember

- Kubernetes runs and looks after containers across many computers.
- You describe **what you want**; it keeps making reality match — forever.
- Self-healing, scaling, rolling updates and service discovery come built in.
