# OpenShift 101 — Deploy, Scale & Fix

A practical, copy-pasteable companion to the **OpenShift 101: Deploy, Scale, & Fix** slide deck.
Follow along step-by-step — every command can be copied with one click on GitHub.

> **How to use:** open the deck side-by-side with this guide. Each section below maps to one slide and contains the exact commands you need to run.

---

## Table of Contents

1. [Architecture overview](#1-architecture-overview)
2. [Pre-flight check: login & sandbox](#2-pre-flight-check-login--sandbox)
3. [Act I — Lift off (deployment)](#3-act-i--lift-off-deployment)
4. [Opening the doors: exposing the service](#4-opening-the-doors-exposing-the-service)
5. [Visualizing the traffic flow](#5-visualizing-the-traffic-flow)
6. [Act II — Gain altitude (manual scaling)](#6-act-ii--gain-altitude-manual-scaling)
7. [Watching the formation](#7-watching-the-formation)
8. [Autopilot — Horizontal Pod Autoscaler](#8-autopilot--horizontal-pod-autoscaler-hpa)
9. [Beyond pods — scaling the cluster itself](#9-beyond-pods--scaling-the-cluster-itself)
10. [MachineSets — scaling worker nodes](#10-machinesets--scaling-worker-nodes)
11. [Act III — Course correction (troubleshooting)](#11-act-iii--course-correction-troubleshooting)
12. [Investigating state: `oc describe`](#12-investigating-state-oc-describe)
13. [Listening to the engine: `oc logs`](#13-listening-to-the-engine-oc-logs)
14. [Advanced diagnostics: the debug shell](#14-advanced-diagnostics-the-debug-shell)
15. [Mission checklist — command summary](#15-mission-checklist--command-summary)

---

## Prerequisites

You need the `oc` CLI installed and a reachable OpenShift cluster.

```bash
oc version
```

If `oc` is missing, download it from the OpenShift web console (top-right → Help → Command Line Tools), or from [mirror.openshift.com](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/).

---

## 1. Architecture overview

Every OpenShift cluster has two halves:

- **Control Plane (Brain)** — Kubernetes API, scheduler, and operators. Decides *where* pods land.
- **Worker Nodes (Muscle)** — the VMs that actually run your pods.

No commands here — just keep this model in mind for the rest of the guide.

---

## 2. Pre-flight check: login & sandbox

### Step 1 — Authenticate against the cluster

```bash
oc login -u <user> <api_url>
```

Expected output:

```
Login successful.
You have access to 63 projects...
```

### Step 2 — Create your isolated project (namespace)

```bash
oc new-project my-hello-world
```

Expected output:

```
Now using project 'my-hello-world' on server ...
```

> Everything you do from now on lives inside this project — your sandbox.

---

## 3. Act I — Lift off (deployment)

Deploy the sample `hello-openshift` image. One command, three Kubernetes objects (Deployment ▸ ReplicaSet ▸ Pod).

```bash
oc new-app openshift/hello-openshift
```

Watch the objects appear:

```bash
oc get deployment,rs,pod
```

---

## 4. Opening the doors: exposing the service

Pods are isolated by default. A **Route** opens an HTTP door to the outside world.

```bash
oc expose service hello-openshift
```

Verify the route was created:

```bash
oc get route
```

Expected output (yours will differ in hostname):

```
NAME             HOST/PORT                                  PATH   SERVICES         PORT   WILDCARD
hello-openshift  hello-openshift.apps.cluster-411.example  /      hello-openshift  8080   None
```

Test it from your terminal:

```bash
curl http://hello-openshift.apps.cluster-411.example
```

---

## 5. Visualizing the traffic flow

The path of a request:

```
Public Network ▸ DNS (*.apps.ocp4...) ▸ External LB ▸ Router ▸ Service ▸ Pod
```

### Verification checklist

```bash
# 1. Pods are Running
oc get pods

# 2. Service has a ClusterIP
oc get svc

# 3. Route hostname is reachable
oc get route

# 4. End-to-end curl test
curl http://<your-host>
```

---

## 6. Act II — Gain altitude (manual scaling)

Scaling is **declarative**. You set the desired state; OpenShift reconciles reality to match.

```bash
oc scale deployment hello-openshift --replicas=3
```

Confirm:

```bash
oc get deployment hello-openshift
```

---

## 7. Watching the formation

Stream pod state changes live with the `-w` (watch) flag.

```bash
oc get pods -w
```

You'll see something like:

```
hello-openshift-1     Running
hello-openshift-2     ContainerCreating
hello-openshift-3     ContainerCreating
hello-openshift-2     Running
hello-openshift-3     Running
```

Press `Ctrl+C` to stop watching.

---

## 8. Autopilot — Horizontal Pod Autoscaler (HPA)

Let OpenShift adjust the replica count automatically based on live CPU pressure.

```bash
oc autoscale deployment hello-openshift --min=1 --max=10 --cpu-percent=50
```

Inspect the HPA:

```bash
oc get hpa
oc describe hpa hello-openshift
```

The feedback loop:

1. **Pod Count** — live count of running replicas.
2. **Metrics Server** — samples CPU/memory every ~15s.
3. **HPA Policy** — compares usage to your target (50%).
4. **Scale Adjustment** — adds or removes pods accordingly.

---

## 9. Beyond pods — scaling the cluster itself

Pods need somewhere to run. When demand exceeds node capacity, you can scale the **underlying VMs**, not just the pods on them.

| Level | What you scale | Command |
|---|---|---|
| **Pod level** | Replicas of your container | `oc scale deployment ... --replicas=N` |
| **Pod level** | Auto-replicas by CPU | `oc autoscale deployment ... --min --max` |
| **Machine level** | Worker VMs themselves | `oc scale machineset ... --replicas=N` |

---

## 10. MachineSets — scaling worker nodes

A `MachineSet` is the Kubernetes-native object that controls how many worker VMs the cluster runs.

### Step 1 — List the MachineSets in your cluster

```bash
oc get machineset -n openshift-machine-api
```

Expected output:

```
NAME                          DESIRED  CURRENT  READY  AVAILABLE  AGE
cluster-411-worker-us-east-1a    1        1        1      1         14d
cluster-411-worker-us-east-1b    1        1        1      1         14d
cluster-411-worker-us-east-1c    1        1        1      1         14d
```

### Step 2 — Scale a worker MachineSet

Drain pods from a node and remove it (set replicas to `0`):

```bash
oc scale --replicas=0 machineset <worker-machineset-name> -n openshift-machine-api
```

To grow the cluster instead, set a higher number:

```bash
oc scale --replicas=3 machineset <worker-machineset-name> -n openshift-machine-api
```

> **⚠ Pro Tip:** `--replicas=0` will **cordon and drain** the node before removing it. Use it to right-size your cluster or retire infrastructure — never on the only node that hosts your workload.

Verify the change:

```bash
oc get machineset -n openshift-machine-api
oc get nodes
```

---

## 11. Act III — Course correction (troubleshooting)

Three commands, three lenses on a misbehaving workload:

| Lens | Command | Answers |
|---|---|---|
| **Observe** | `oc get` | Is it running? |
| **Inspect** | `oc describe` | Why is it pending? |
| **Listen** | `oc logs` | Why did it crash? |

```bash
oc get pods
```

---

## 12. Investigating state: `oc describe`

`describe` surfaces metadata and — most importantly — **events**. Most pod problems are explained right there.

```bash
oc describe pod <pod-name>
```

Look for the `Events:` section at the bottom, e.g.:

```
Events:
  Type     Reason             Message
  Warning  FailedScheduling   0/5 nodes are available: Insufficient cpu.
  Normal   Pulling            Pulling image 'openshift/hello-openshift'
```

### Key statuses to watch

| Status | Meaning |
|---|---|
| `ErrImagePull` | Registry issue — wrong image name or auth missing. |
| `ImagePullBackOff` | Repeated registry failure. |
| `CrashLoopBackOff` | App crashes on startup, kubelet keeps retrying. |
| `Pending` | No node has resources to schedule this pod. |

---

## 13. Listening to the engine: `oc logs`

Logs are the conversation your app is having with the world. Read them.

```bash
oc logs <pod-name>
```

### Useful flags

```bash
# Follow logs in real time
oc logs -f <pod-name>

# Pick a specific container in a multi-container pod
oc logs -c <container-name> <pod-name>

# See the dead container's log (after a crash)
oc logs --previous <pod-name>
```

---

## 14. Advanced diagnostics: the debug shell

Pod crashes the instant it starts? You can't `oc exec` into a dead container.
`oc debug` launches a **sibling pod** with the same image but a `/bin/sh` entrypoint, so you can poke around exactly as the failing pod would have seen things.

```bash
oc debug deployment/hello-openshift
```

Once inside, inspect filesystem, environment variables, config, etc.:

```bash
ls -la /app
env
cat /etc/config/app.yaml
```

Type `exit` to leave the debug pod.

---

## 15. Mission checklist — command summary

Every command from this deck, on one page.

### Setup
```bash
oc login -u <user> <api-url>
oc new-project <name>
```

### Deploy
```bash
oc new-app <image>
oc expose service <name>
oc get route
```

### Scale — pod level
```bash
oc scale deployment <name> --replicas=3
oc autoscale deployment <name> --min=1 --max=10 --cpu-percent=50
```

### Scale — machine level
```bash
oc get machineset -n openshift-machine-api
oc scale machineset <name> --replicas=N -n openshift-machine-api
```

### Fix
```bash
oc get pods
oc describe pod <name>
oc logs -f <name>
oc logs --previous <name>
oc debug deployment/<name>
```

---

## Next steps

- **Web Console** — explore the *Developer* perspective for visual workflows.
- **OperatorHub** — install databases, monitoring, and pipelines with one click.
- **Documentation** — [docs.openshift.com](https://docs.openshift.com)

---

*End of guide. Companion to **OpenShift 101: Deploy, Scale & Fix** (17 slides).*
