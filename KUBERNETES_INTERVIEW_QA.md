# Kubernetes — Real Interview Questions & Answers

> This file is a personal log of actual Kubernetes questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Capgemini | Azure DevOps | Technical Round 1](#interview-1)
  - [Q1. What are the different Pod States?](#q1-what-are-the-different-pod-states)
  - [Q2. What is CrashLoopBackOff and how do you fix it?](#q2-what-is-crashloopbackoff-and-how-do-you-fix-it)
  - [Q3. What is ImagePullBackOff? How do you fix it?](#q3-what-is-imagepullbackoff-how-do-you-fix-it)
  - [Q4. Difference between CrashLoopBackOff and ImagePullBackOff?](#q4-difference-between-crashloopbackoff-and-imagepullbackoff)
  - [Scenario 1. ImagePullBackOff — Possible Reasons?](#scenario-1-imagepullbackoff----possible-reasons)
  - [Scenario 2. Same image works for another app — only my app gets ImagePullBackOff](#scenario-2-advanced-same-image-same-tag-same-registry-works-for-another-app-only-your-application-gets-imagepullbackoff-what-could-be-the-reason)

---

## Interview #1

**Company:** Capgemini  
**Date:** 18-07-2026  
**Role Applied For:** Azure DevOps  
**Round:** Technical Round 1  
**Interviewer Level:** Tech Lead

---

### Questions Asked

#### Q1. What are the different Pod States?

**Answer:**

This is a fundamental Kubernetes question. Understanding pod states helps you diagnose problems quickly — when something goes wrong in production, the pod state is the first thing you look at.

Kubernetes has two levels of state to understand:
1. **Pod Phase** — the overall lifecycle state of the pod
2. **Container State** — the state of each individual container inside the pod

Let me explain both clearly.

---

**Pod Phases — The 5 Overall States**

A pod phase is the high-level summary of where the pod is in its lifecycle. You see it when you run:

```bash
kubectl get pods
```

Output:
```
NAME                          READY   STATUS      RESTARTS   AGE
user-service-7d9f8b-xk2p9    1/1     Running     0          2d
order-service-5c6d7e-mn3q1   0/1     Pending     0          5m
payment-service-9a8b7c-pq4r  0/1     CrashLoopBackOff  5   10m
job-processor-abc123          0/1     Completed   0          1h
```

---

**Phase 1 — Pending**

The pod has been accepted by Kubernetes but is not running yet. It is waiting for something to happen before it can start.

**What it means:**
- The pod definition has been submitted to the API server
- Kubernetes has not yet scheduled it to a node, OR
- It has been scheduled but the containers inside have not started yet

**Common reasons a pod stays in Pending:**

| Reason | What is happening |
|---|---|
| **Insufficient resources** | No node has enough CPU or memory to run the pod |
| **No matching node** | Pod has a `nodeSelector` or `affinity` rule that no node satisfies |
| **Image being pulled** | The container image is being downloaded from the registry |
| **PVC not bound** | The pod needs a Persistent Volume Claim that has not been provisioned yet |
| **Taints and tolerations** | All nodes are tainted and the pod does not have the matching toleration |

**How to diagnose:**
```bash
kubectl describe pod <pod-name>
```

Look at the **Events** section at the bottom — it tells you exactly why the pod is pending:
```
Events:
  Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient memory.
```

**AzureShop Example:**

In AzureShop, after a deployment we saw pods stuck in `Pending` for over 10 minutes. `kubectl describe pod` showed:
```
0/3 nodes are available: 3 Insufficient cpu.
```
Our new deployment requested more CPU than any node had available. We fixed it by adjusting the CPU request in the deployment spec and enabling AKS cluster autoscaler to add a new node.

---

**Phase 2 — Running**

The pod has been scheduled to a node, all containers have been created, and at least one container is running.

**What it means:**
- The pod is on a node
- All containers are either running, starting, or restarting
- This is the healthy, expected state for long-running applications

**Important:** `Running` does NOT necessarily mean the application inside is healthy. The container process is running — but the app inside might be throwing errors. This is why we have **Readiness Probes** — a pod can be `Running` but not `Ready` if the readiness probe is failing.

```bash
NAME                        READY   STATUS    RESTARTS
user-service-7d9f8b-xk2p9  0/1     Running   0
```

`READY 0/1` means the container is running but the readiness probe is failing — traffic is NOT being sent to this pod yet.

```bash
NAME                        READY   STATUS    RESTARTS
user-service-7d9f8b-xk2p9  1/1     Running   0
```

`READY 1/1` means the container is running AND the readiness probe is passing — pod is healthy and receiving traffic.

---

**Phase 3 — Succeeded**

All containers in the pod have completed successfully — exited with exit code 0 — and will not be restarted.

**What it means:**
- The pod ran its task and finished cleanly
- This is the expected final state for **Jobs** and **CronJobs** — not for long-running services

```bash
NAME               READY   STATUS      RESTARTS   AGE
db-migration-job   0/1     Completed   0          1h
```

**AzureShop Example:**

In AzureShop, we ran database migration jobs before each deployment. After the migration completed successfully, the pod showed `Completed`. This was expected and correct — the job ran once, finished, and we did not want it to restart.

---

**Phase 4 — Failed**

At least one container in the pod terminated with a non-zero exit code (failure) or was killed by the system, and will not be restarted.

**What it means:**
- The pod tried to run but something went wrong — the application crashed, ran out of memory, or the container exited with an error
- Kubernetes will not automatically restart it (depending on the `restartPolicy`)

**Common reasons:**
- Application threw an unhandled exception and exited
- OOM (Out of Memory) — the container exceeded its memory limit
- The command in `CMD` or `ENTRYPOINT` failed

**How to diagnose:**
```bash
kubectl logs <pod-name>                   # Current logs
kubectl logs <pod-name> --previous        # Logs from before the crash
kubectl describe pod <pod-name>           # Exit code and reason
```

In `kubectl describe`, look for:
```
Last State:   Terminated
  Reason:     OOMKilled
  Exit Code:  137
```

Exit code `137` = OOM kill. Exit code `1` = application error. Exit code `0` = clean exit (Succeeded).

---

**Phase 5 — Unknown**

Kubernetes cannot determine the state of the pod. This usually means the node the pod is running on has lost communication with the Kubernetes API server.

**What it means:**
- The kubelet on the node is not reporting back to the API server
- The node may be down, having network issues, or the kubelet process crashed
- Kubernetes does not know if the pod is running, stopped, or what

**How to diagnose:**
```bash
kubectl get nodes
# Look for a node in NotReady status

kubectl describe node <node-name>
# Look at Conditions — KubeletReady = False
```

**AzureShop Example:**

In AzureShop, we once had an AKS node go into `Unknown` state after a VM host issue in Azure. The pods on that node showed `Unknown`. We cordoned the node, drained the pods to healthy nodes, and let AKS replace the faulty node:

```bash
kubectl cordon <node-name>    # Stop new pods being scheduled here
kubectl drain <node-name> --ignore-daemonsets --delete-emtpydir-data
```

---

**Common STATUS values you see in `kubectl get pods` beyond the 5 phases:**

These are not official phases — they are more detailed status messages Kubernetes shows:

| STATUS | What it means |
|---|---|
| `Running` | Pod is running normally |
| `Pending` | Waiting to be scheduled or image being pulled |
| `Completed` | Job finished successfully |
| `CrashLoopBackOff` | Container keeps crashing — Kubernetes is backing off before retrying |
| `ImagePullBackOff` | Cannot pull the container image from the registry |
| `ErrImagePull` | First failed attempt to pull the image |
| `OOMKilled` | Container was killed because it exceeded its memory limit |
| `Terminating` | Pod is being deleted — waiting for graceful shutdown |
| `Init:0/1` | Init container is running (0 of 1 complete) |
| `PodInitializing` | Init containers completed, main containers starting |
| `ContainerCreating` | Container is being created on the node |
| `Error` | Container exited with an error |

**`CrashLoopBackOff` deserves special mention** — this is one of the most common and important states you will see in production:

It means the container is crashing repeatedly. Kubernetes tries to restart it, it crashes again, Kubernetes waits longer before retrying (exponential backoff — 10s, 20s, 40s, 80s... up to 5 minutes), crashes again. This loop continues.

**Common causes of `CrashLoopBackOff`:**
- Application exits immediately due to a missing environment variable or config
- Wrong `CMD` pointing to a file that does not exist
- Application cannot connect to the database on startup
- Out of memory

**How to diagnose:**
```bash
kubectl logs <pod-name> --previous    # CRITICAL — logs from BEFORE the crash
kubectl describe pod <pod-name>       # Exit code, restart count, last state
```

---

**Container States — inside each pod:**

Each container inside a pod also has its own state. You see these in `kubectl describe pod`:

**1. `Waiting`** — the container is not running yet. It is waiting for something — image pull, init containers to finish, etc. The `Reason` field tells you why: `ContainerCreating`, `PodInitializing`, `CrashLoopBackOff`, `ImagePullBackOff`.

**2. `Running`** — the container is executing. `startedAt` shows when it started.

**3. `Terminated`** — the container has finished. `exitCode` tells you if it was success (0) or failure (non-zero). `reason` tells you how it ended: `Completed`, `Error`, `OOMKilled`.

```bash
kubectl describe pod user-service-7d9f8b-xk2p9
```

```
Containers:
  user-service:
    State:          Running
      Started:      Fri, 18 Jul 2026 09:00:00 +0530
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:     250m
      memory:  256Mi
    Liveness:   http-get http://:3000/health delay=30s timeout=5s period=10s
    Readiness:  http-get http://:3000/ready delay=10s timeout=3s period=5s
```

---

**Quick diagnostic commands:**

```bash
# See all pods and their status
kubectl get pods -n <namespace>

# See pods on all namespaces
kubectl get pods -A

# Watch pods in real time
kubectl get pods -w

# Get detailed info and events for a specific pod
kubectl describe pod <pod-name>

# Get logs of a running pod
kubectl logs <pod-name>

# Get logs of a crashed pod (previous run)
kubectl logs <pod-name> --previous

# Get logs with timestamps
kubectl logs <pod-name> --timestamps

# Stream logs live
kubectl logs <pod-name> -f
```

---

**Summary (what to say if time is short):**

*"Kubernetes pods have 5 phases. Pending — the pod is accepted but not running yet, either waiting to be scheduled or the image is being pulled. Running — the pod is on a node and at least one container is running — but Running does not mean healthy, the readiness probe might still be failing. Succeeded — all containers completed with exit code 0, expected for Jobs. Failed — at least one container exited with an error or was killed. Unknown — Kubernetes cannot determine the state, usually because the node lost communication with the API server. Beyond these 5 phases, common status values you see in kubectl get pods include CrashLoopBackOff — where the container keeps crashing and Kubernetes backs off before retrying — and ImagePullBackOff where the image cannot be pulled from the registry. Each container inside a pod also has its own state — Waiting, Running, or Terminated. The most useful diagnostic commands are kubectl describe pod for events and exit codes, and kubectl logs --previous for logs from a crashed container."*

---

#### Q2. What is CrashLoopBackOff and how do you fix it?

**Answer:**

`CrashLoopBackOff` is one of the most common and most important Kubernetes states you will encounter in real production environments. Every DevOps engineer must know this deeply — both what it means and how to systematically fix it.

---

**What is CrashLoopBackOff?**

`CrashLoopBackOff` means:
- The container inside the pod **started**
- Then it **crashed** — the process exited with a non-zero exit code or was killed
- Kubernetes tried to **restart** it (because that is what Kubernetes does by default)
- It **crashed again**
- This **loop** keeps repeating

The **BackOff** part is important — it means Kubernetes does not restart the container immediately every time. It uses an **exponential backoff** strategy — it waits longer and longer between each restart attempt to avoid burning resources on a container that keeps failing:

```
Restart 1 → crash → wait 10 seconds
Restart 2 → crash → wait 20 seconds
Restart 3 → crash → wait 40 seconds
Restart 4 → crash → wait 80 seconds
Restart 5 → crash → wait 160 seconds
Restart 6+ → crash → wait 300 seconds (5 minutes — maximum backoff)
```

The maximum wait time is 5 minutes. After that, Kubernetes keeps retrying every 5 minutes indefinitely until you fix the problem or delete the pod.

You can see the restart count climbing:

```bash
kubectl get pods

NAME                          READY   STATUS             RESTARTS   AGE
user-service-7d9f8b-xk2p9    0/1     CrashLoopBackOff   8          25m
```

`RESTARTS: 8` means it has crashed and been restarted 8 times already.

---

**Simple analogy:**

Think of it like a light switch that keeps tripping a circuit breaker. Every time you flip it on, it trips. You flip it again — trips again. Eventually the breaker starts taking longer and longer to reset — that is the backoff. The real fix is to find out WHY it is tripping (the root cause), not to keep flipping the switch.

---

**How to diagnose CrashLoopBackOff — Step by Step:**

---

**Step 1 — Get the logs from the crashed container**

This is the most important command:

```bash
kubectl logs <pod-name> --previous
```

The `--previous` flag is critical. Without it, you get logs from the current (waiting) state — which may be empty. `--previous` gives you the logs from the last run before the crash — where the actual error is.

```bash
# Also try without --previous for any logs currently available
kubectl logs <pod-name>

# If multiple containers in the pod
kubectl logs <pod-name> -c <container-name> --previous

# Get last 100 lines
kubectl logs <pod-name> --previous --tail=100
```

---

**Step 2 — Describe the pod to get the exit code and reason**

```bash
kubectl describe pod <pod-name>
```

Look for the `Last State` section:

```
Containers:
  user-service:
    Last State:   Terminated
      Reason:     Error
      Exit Code:  1
      Started:    Fri, 18 Jul 2026 09:15:00 +0530
      Finished:   Fri, 18 Jul 2026 09:15:02 +0530
    Ready:        False
    Restart Count: 8
```

**The exit code tells you a lot:**

| Exit Code | Meaning |
|---|---|
| `0` | Clean exit — not a crash, the process finished intentionally |
| `1` | General application error — check logs |
| `2` | Misuse of shell command |
| `137` | Killed by signal 9 (SIGKILL) — usually OOM kill |
| `139` | Segmentation fault — memory access violation |
| `143` | Killed by signal 15 (SIGTERM) — graceful termination requested but timed out |

Also look at the **Events** section at the bottom of `kubectl describe`:

```
Events:
  Warning  BackOff  2m   kubelet  Back-off restarting failed container
  Warning  Failed   5m   kubelet  Error: failed to create containerd task: OOM
```

---

**Now let me explain every common cause and its fix:**

---

**Cause 1 — Missing or Wrong Environment Variable**

The application starts, immediately checks for a required environment variable, finds it missing, and exits with an error. This is the most common cause.

**Symptoms in logs:**
```
Error: DATABASE_URL environment variable is required but not set
Process exited with code 1
```

The container started and crashed in 1-2 seconds — that is a dead giveaway for a missing env var.

**How to verify:**
```bash
# Check what env vars are set on the pod
kubectl exec <pod-name> -- env | grep DATABASE
# OR describe the pod and look at Environment section
kubectl describe pod <pod-name>
```

**Fix — add the missing environment variable to the deployment:**

```yaml
spec:
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:latest
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: connection-string
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
```

---

**Cause 2 — Application Cannot Connect to Database on Startup**

The application starts, tries to connect to the database, fails (DB not ready, wrong hostname, wrong credentials), and crashes. With the default `restartPolicy: Always`, Kubernetes keeps restarting it — CrashLoopBackOff.

**Symptoms in logs:**
```
Error: Connection refused — could not connect to postgres://db-service:5432/appdb
Application failed to start
```

**Fix 1 — Add an init container that waits for the DB to be ready before the main container starts:**

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -zv db-service 5432; do echo waiting for database; sleep 3; done']

  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:latest
```

The init container keeps trying until the DB is reachable. The main container only starts after the init container succeeds.

**Fix 2 — Add retry logic in the application code:**

```javascript
async function connectWithRetry(retries = 5) {
  for (let i = 0; i < retries; i++) {
    try {
      await db.connect();
      console.log('Database connected');
      return;
    } catch (err) {
      console.log(`DB connection attempt ${i + 1} failed. Retrying in 5s...`);
      await sleep(5000);
    }
  }
  throw new Error('Could not connect to database after retries');
}
```

A well-written application should not crash on the first DB connection failure — it should retry.

---

**Cause 3 — OOM Kill — Out of Memory**

The container exceeds its memory limit. Kubernetes kills it with SIGKILL (exit code 137). The application restarts, reaches the same memory usage, gets killed again — CrashLoopBackOff.

**Symptoms:**
```bash
kubectl describe pod <pod-name>
```
```
Last State:   Terminated
  Reason:     OOMKilled
  Exit Code:  137
```

**Fix — increase the memory limit in the deployment:**

```yaml
spec:
  containers:
    - name: user-service
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"    # Increase this if OOMKilled
          cpu: "500m"
```

Also investigate the application for memory leaks — if it keeps growing until it hits the limit, increasing the limit is only a temporary fix.

**Check current resource usage:**
```bash
kubectl top pod <pod-name>
```

---

**Cause 4 — Wrong Command or Entrypoint**

The `CMD` or `ENTRYPOINT` in the Dockerfile (or the `command`/`args` in the deployment spec) points to a file that does not exist, or the command itself is wrong.

**Symptoms in logs:**
```
Error: Cannot find module '/app/dist/server.js'
# or
/bin/sh: /app/start.sh: not found
```

Container crashes in under a second — the process could not even start.

**Fix — correct the command in the deployment spec:**

```yaml
spec:
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:latest
      command: ["node"]
      args: ["dist/server.js"]    # Make sure this path exists inside the image
```

**Debug by running a shell in the image to verify the file exists:**

```bash
# Run the image interactively with a shell instead of the default CMD
docker run -it --entrypoint sh azureshopacr.azurecr.io/user-service:latest
ls /app/dist/    # Check if server.js is actually there
```

---

**Cause 5 — Liveness Probe Failing**

The application starts and runs — but the liveness probe keeps failing. Kubernetes concludes the container is unhealthy, kills it, and restarts it. This creates CrashLoopBackOff even though the application itself may be running fine.

**Symptoms:**
```bash
kubectl describe pod <pod-name>
```
```
Events:
  Warning  Unhealthy  3m   kubelet  Liveness probe failed:
           HTTP probe failed with statuscode: 500
  Warning  Killing    3m   kubelet  Container user-service failed liveness probe,
           will be restarted
```

**Common liveness probe mistakes:**

```yaml
# Wrong — probing too early before app is ready
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5    # App needs 30 seconds to start — probe fails in first 5s

# Correct
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30   # Give app enough time to start
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

**Fix — adjust the liveness probe settings:**
- Increase `initialDelaySeconds` — give the app more time to start before probing
- Increase `failureThreshold` — allow more failures before killing
- Increase `timeoutSeconds` — allow more time for the health check to respond
- Make sure the `/health` endpoint actually works and returns 200

---

**Cause 6 — Application Bug or Unhandled Exception**

The application itself has a bug — an unhandled exception that causes it to crash. The logs will show the stack trace.

**Symptoms in logs:**
```
UnhandledPromiseRejectionWarning: TypeError: Cannot read property 'id' of undefined
    at processOrder (/app/dist/order.js:45:23)
Process exited with code 1
```

**Fix:**
- Read the stack trace carefully — it tells you exactly which file and line number crashed
- Fix the bug in the application code
- Add proper error handling so one bad request does not crash the entire process
- Build and push a new image, update the deployment

---

**Cause 7 — ConfigMap or Secret Not Found**

The pod references a ConfigMap or Secret that does not exist in the namespace. Kubernetes cannot start the container because it cannot mount the volume or inject the env vars.

**Symptoms in `kubectl describe`:**
```
Events:
  Warning  Failed  2m  kubelet  Error: secret "db-secret" not found
```

**Fix — create the missing Secret or ConfigMap:**

```bash
# Create the missing secret
kubectl create secret generic db-secret \
  --from-literal=connection-string="postgres://admin:password@db-service:5432/appdb" \
  -n production
```

Or via YAML:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  connection-string: "postgres://admin:password@db-service:5432/appdb"
```

---

**Complete CrashLoopBackOff diagnosis flow:**

```
kubectl get pods → CrashLoopBackOff
          |
          ↓
kubectl logs <pod> --previous
  → Missing env var?     → Add env var to deployment
  → DB connection error? → Add init container or retry logic
  → File not found?      → Fix CMD/ENTRYPOINT path
  → App exception?       → Fix the bug, redeploy
          |
          ↓
kubectl describe pod <pod>
  → Exit code 137?       → OOMKilled → increase memory limit
  → Liveness probe fail? → Increase initialDelaySeconds
  → Secret not found?    → Create the missing Secret/ConfigMap
  → OOMKilled?           → Check kubectl top pod, increase limit
          |
          ↓
Fix the root cause → kubectl rollout restart deployment/<name>
```

---

**AzureShop Real Examples:**

**Incident 1 — Missing Secret**

After deploying the `payment-service` to a new namespace, pods immediately went into `CrashLoopBackOff`. `kubectl describe` showed:
```
Error: secret "payment-api-keys" not found
```

We had deployed the application but forgotten to create the Secret in the new namespace. Fix:
```bash
kubectl create secret generic payment-api-keys \
  --from-literal=stripe-key="sk_live_xxx" \
  -n production
```
Pods came up healthy immediately.

**Incident 2 — OOMKill on order-service**

The `order-service` kept crashing with exit code `137`. `kubectl top pod` showed it was hitting 512Mi — exactly the memory limit — every time. We increased the limit to 1Gi and the crashes stopped. Then we profiled the application and found a memory leak in the order processing logic that was fixed in the next release.

**Incident 3 — Liveness probe too aggressive**

After a new deployment, the `user-service` pods were stuck in CrashLoopBackOff. The app itself was fine — but the liveness probe was hitting `/health` after only 5 seconds, while the app needed 25 seconds to connect to the DB and warm up. Kubernetes killed the pod before it was ready. Fix: increased `initialDelaySeconds` from 5 to 30.

---

**Summary (what to say if time is short):**

*"CrashLoopBackOff means a container keeps starting, crashing, and being restarted by Kubernetes in a loop. The BackOff means Kubernetes waits increasingly longer between restarts — 10s, 20s, 40s, up to 5 minutes — to avoid wasting resources. To diagnose it, I always start with `kubectl logs <pod> --previous` to see logs from before the crash, and `kubectl describe pod` to get the exit code and events. The most common causes are — missing or wrong environment variables where the app exits immediately on startup, application cannot connect to the database on startup fixed with an init container, OOM kill where the container exceeds its memory limit shown by exit code 137, wrong CMD or ENTRYPOINT pointing to a file that does not exist, liveness probe failing too early before the app has time to start, application bugs causing unhandled exceptions, and missing Secrets or ConfigMaps. In AzureShop I fixed CrashLoopBackOff from a missing Secret, an OOM kill on the order-service, and an overly aggressive liveness probe — each required a different fix but the diagnosis approach was always the same: logs first, then describe."*

---

#### Q3. What is ImagePullBackOff? How do you fix it?

**Answer:**

`ImagePullBackOff` is another very common Kubernetes error that every DevOps engineer encounters in production. It is different from `CrashLoopBackOff` — in `CrashLoopBackOff` the container starts and then crashes. In `ImagePullBackOff` the container never even starts — Kubernetes cannot pull the Docker image from the registry.

---

**What is ImagePullBackOff?**

When Kubernetes schedules a pod to a node, the first thing the kubelet does is pull the container image from the registry (Docker Hub, Azure Container Registry, etc.). If that pull fails, Kubernetes reports the error.

There are actually **two related statuses** you will see:

| Status | What it means |
|---|---|
| `ErrImagePull` | The very **first** failed attempt to pull the image |
| `ImagePullBackOff` | After the first failure, Kubernetes **backs off** and retries — same exponential backoff as CrashLoopBackOff (10s → 20s → 40s → 5 min) |

So `ErrImagePull` comes first, then transitions to `ImagePullBackOff` as Kubernetes keeps retrying.

```bash
kubectl get pods

NAME                          READY   STATUS             RESTARTS   AGE
user-service-7d9f8b-xk2p9    0/1     ImagePullBackOff   0          5m
payment-service-5c6d7e-mn3q   0/1     ErrImagePull       0          1m
```

Notice `RESTARTS: 0` — the container never even started. The problem is entirely at the image pull stage.

---

**Simple analogy:**

Think of it like ordering a package online. Before you can use what you ordered, the delivery must arrive. `ImagePullBackOff` means the delivery failed — the package never arrived. You cannot open something that was never delivered. The container cannot start if the image cannot be pulled.

---

**How to diagnose ImagePullBackOff:**

**Step 1 — Describe the pod:**

```bash
kubectl describe pod <pod-name>
```

Look at the **Events** section — it always tells you exactly why the pull failed:

```
Events:
  Warning  Failed     3m   kubelet  Failed to pull image
           "azureshopacr.azurecr.io/user-service:v2.1.0":
           rpc error: code = Unknown desc = failed to pull and unpack image:
           unauthorized: authentication required
```

Or:

```
Events:
  Warning  Failed     2m   kubelet  Failed to pull image
           "azureshopacr.azurecr.io/user-service:v99.0.0":
           rpc error: code = NotFound desc = failed to pull and unpack image:
           not found
```

The error message in Events is your starting point. It tells you exactly what went wrong.

---

**Now let me cover every common cause and its fix:**

---

**Cause 1 — Wrong Image Name or Tag (Most Common)**

The image name is misspelled, or the tag does not exist in the registry. This is the most frequent cause — a simple typo or a tag that was never pushed.

**Symptoms in Events:**
```
Failed to pull image "azureshopacr.azurecr.io/user-servce:v2.1.0": not found
#                                               ↑ typo — "servce" instead of "service"
```
Or:
```
Failed to pull image "azureshopacr.azurecr.io/user-service:v99.0.0": not found
# Tag v99.0.0 was never pushed to the registry
```

**How to verify:**

```bash
# Check what image the deployment is trying to use
kubectl describe pod <pod-name> | grep Image

# List available tags in ACR
az acr repository show-tags \
  --name azureshopacr \
  --repository user-service \
  --output table
```

Output:
```
Result
--------
latest
v1.0.0
v2.0.0
v2.1.0
```

If `v99.0.0` is not in the list — that is your problem.

**Fix — correct the image name or tag in the deployment:**

```yaml
spec:
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:v2.1.0    # Correct tag
```

Apply the fix:
```bash
kubectl set image deployment/user-service \
  user-service=azureshopacr.azurecr.io/user-service:v2.1.0

# Or edit the deployment directly
kubectl edit deployment user-service
```

---

**Cause 2 — Missing Image Pull Secret (Authentication Failed)**

The registry is private — like Azure Container Registry — and Kubernetes does not have the credentials to authenticate. Without credentials, the registry rejects the pull request.

**Symptoms in Events:**
```
Failed to pull image "azureshopacr.azurecr.io/user-service:v2.1.0":
unauthorized: authentication required
```

Or:
```
Failed to pull image "azureshopacr.azurecr.io/user-service:v2.1.0":
403 Forbidden
```

**How Kubernetes authenticates with a private registry:**

Kubernetes uses a special Secret of type `kubernetes.io/dockerconfigjson` — called an **image pull secret** — to store registry credentials. This secret is referenced in the pod spec.

**Fix — Create the image pull secret and reference it:**

**For Azure Container Registry:**

```bash
# Option 1 — Create a pull secret using ACR credentials
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<acr-password> \
  --docker-email=devops@azureshop.com \
  -n production
```

Then reference it in the deployment:

```yaml
spec:
  imagePullSecrets:
    - name: acr-pull-secret    # Reference the secret here
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:v2.1.0
```

**Option 2 — Use AKS managed identity (better approach for Azure):**

Instead of storing credentials in a Secret, attach the AKS kubelet managed identity to ACR with the `AcrPull` role. AKS nodes then authenticate automatically — no secret needed.

```bash
# Assign AcrPull role to AKS kubelet identity
az role assignment create \
  --assignee <aks-kubelet-object-id> \
  --role AcrPull \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/azureshopacr
```

Or through Terraform (as we did in AzureShop):
```hcl
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.acr.id
}
```

With this, no `imagePullSecrets` is needed in the pod spec at all — AKS handles authentication transparently.

**Option 3 — Attach pull secret to the default service account:**

If you do not want to add `imagePullSecrets` to every deployment:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "acr-pull-secret"}]}' \
  -n production
```

Now every pod in the `production` namespace automatically uses this pull secret.

---

**Cause 3 — Image Does Not Exist in the Registry**

The image was never pushed to the registry, or it was pushed to a different registry than what the deployment is pointing to.

**Symptoms in Events:**
```
Failed to pull image "azureshopacr.azurecr.io/new-service:v1.0.0": not found
```

**How to verify:**

```bash
# Check if the image exists in ACR
az acr repository show \
  --name azureshopacr \
  --image new-service:v1.0.0
```

If it returns `Repository 'new-service' does not exist` — the image was never pushed.

**Fix:**

Either push the image to the registry first:
```bash
docker build -t azureshopacr.azurecr.io/new-service:v1.0.0 .
docker push azureshopacr.azurecr.io/new-service:v1.0.0
```

Or check the deployment — it might be pointing to the wrong registry:
```bash
# Is it pointing to the right registry?
kubectl describe pod <pod-name> | grep Image
# Image: dockerhub.io/user-service:v2.1.0  ← wrong registry
# Should be: azureshopacr.azurecr.io/user-service:v2.1.0
```

---

**Cause 4 — Registry is Unreachable (Network Issue)**

The AKS node cannot reach the container registry — a firewall rule, network policy, or DNS issue is blocking access.

**Symptoms in Events:**
```
Failed to pull image "azureshopacr.azurecr.io/user-service:v2.1.0":
dial tcp: lookup azureshopacr.azurecr.io: no such host
```
Or:
```
connection refused
context deadline exceeded
```

**How to verify:**

```bash
# SSH into an AKS node and test connectivity
# Or run a debug pod on the node
kubectl run debug --image=busybox --restart=Never -- sh

# Inside the debug pod:
nslookup azureshopacr.azurecr.io
wget -qO- https://azureshopacr.azurecr.io/v2/
```

**Fix:**

- Check NSG (Network Security Group) rules on the AKS subnet — port 443 must be open outbound to the registry
- Check Azure Firewall rules if using a private cluster
- If using ACR with Private Endpoint — verify the private DNS zone is configured correctly and the AKS VNet is linked

**AzureShop Example:**

In AzureShop, after enabling a Private Endpoint on ACR for security, all pods suddenly showed `ImagePullBackOff`. The issue was that the Private DNS Zone for `azureshopacr.azurecr.io` was not linked to the AKS VNet. Nodes could not resolve the private IP of ACR. Fix: linked the Private DNS Zone to the AKS VNet in Terraform:

```hcl
resource "azurerm_private_dns_zone_virtual_network_link" "acr_aks" {
  name                  = "acr-aks-dns-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.main.id
}
```

---

**Cause 5 — `imagePullPolicy: Always` with No Internet Access**

If `imagePullPolicy` is set to `Always`, Kubernetes tries to pull the image from the registry every time the pod starts — even if the image is already cached on the node. In an air-gapped or restricted network environment, this always fails.

**Check the current imagePullPolicy:**
```bash
kubectl describe pod <pod-name> | grep "Image Pull Policy"
```

**Fix — change `imagePullPolicy` to `IfNotPresent`:**

```yaml
spec:
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:v2.1.0
      imagePullPolicy: IfNotPresent    # Only pull if not already on the node
```

| imagePullPolicy | Behaviour |
|---|---|
| `Always` | Always pull from registry — even if cached |
| `IfNotPresent` | Use cached image if available, pull only if not present |
| `Never` | Never pull — use only what is already on the node |

**Best practice:** Use `IfNotPresent` for tagged versions. Use `Always` only with `latest` tag — but avoid using `latest` in production altogether.

---

**Complete diagnosis flow:**

```
ImagePullBackOff
      |
      ↓
kubectl describe pod → check Events
      |
      |─── "not found" or "does not exist"
      |         → Wrong image name/tag → fix the image tag in deployment
      |
      |─── "unauthorized" or "403 Forbidden"
      |         → Missing credentials → create imagePullSecret or assign AcrPull role
      |
      |─── "no such host" or "connection refused"
      |         → Registry unreachable → check NSG, firewall, DNS
      |
      |─── Image tag correct but not pushed yet
                → Push the image to registry first
```

---

**AzureShop Real Examples:**

**Incident 1 — Wrong tag in pipeline**

After a pipeline deployment, the `order-service` pods showed `ImagePullBackOff`. `kubectl describe` showed:
```
Failed to pull image "azureshopacr.azurecr.io/order-service:v3.0.0": not found
```
The pipeline had a bug — it built the image with tag `v3.0.1` (from the git tag) but the deployment manifest was hardcoded to `v3.0.0`. We fixed the pipeline to dynamically inject the correct tag into the deployment:
```bash
kubectl set image deployment/order-service \
  order-service=azureshopacr.azurecr.io/order-service:$(git describe --tags)
```

**Incident 2 — AcrPull role missing after AKS upgrade**

After an AKS version upgrade, all pods across all namespaces showed `ImagePullBackOff` with `unauthorized`. The AKS upgrade had created a new kubelet identity, but the `AcrPull` role assignment was on the old identity. Terraform plan showed the role assignment was missing. We applied the Terraform fix to assign `AcrPull` to the new kubelet identity — all pods recovered within 2 minutes.

---

**Summary (what to say if time is short):**

*"`ImagePullBackOff` means Kubernetes cannot pull the container image from the registry — the container never even starts. It starts as `ErrImagePull` on the first failure and transitions to `ImagePullBackOff` as Kubernetes applies exponential backoff between retries. To diagnose it, I always run `kubectl describe pod` and look at the Events section — it tells me the exact reason. The five common causes are — wrong image name or tag where the image simply does not exist in the registry, missing authentication credentials for a private registry like ACR fixed by creating an imagePullSecret or assigning AcrPull role to the AKS managed identity, the image was never pushed to the registry, the registry is unreachable due to a firewall or DNS issue, and `imagePullPolicy: Always` in a restricted network. In AzureShop, I encountered this twice — once due to a wrong tag in the pipeline, and once after an AKS upgrade where the new kubelet identity had lost the AcrPull role assignment on ACR."*

---

#### Q4. Difference between CrashLoopBackOff and ImagePullBackOff?

**Answer:**

Both look similar on the surface — the pod is not running, READY shows `0/1`, and something is going wrong. But they fail at completely different stages and for completely different reasons. Knowing the difference immediately tells you where to look when troubleshooting.

---

**The single most important difference — WHERE in the lifecycle the failure happens:**

```
Pod Lifecycle:

Schedule pod to node
        ↓
Pull the container image from registry
        ↓  ← ImagePullBackOff fails HERE — never gets past this point
Create the container
        ↓
Start the container
        ↓
Run the application
        ↓  ← CrashLoopBackOff fails HERE — gets past image pull, crashes after starting
```

- **ImagePullBackOff** — fails **before** the container ever starts. The image cannot be downloaded.
- **CrashLoopBackOff** — fails **after** the container starts. The image was pulled successfully, container started, but then crashed.

---

**Simple real-world analogy:**

Imagine you order food online:

- **ImagePullBackOff** = The delivery person **cannot find your address** — the food never arrives. You cannot eat something that was never delivered.
- **CrashLoopBackOff** = The food **arrived** — but when you opened the box it was spoiled, you got sick, and you keep trying to eat it and getting sick every time.

In one case the delivery fails. In the other case the delivery succeeded but the product itself is broken.

---

**Side by side comparison:**

| | CrashLoopBackOff | ImagePullBackOff |
|---|---|---|
| **What fails** | The application inside the container | Pulling the image from the registry |
| **When it fails** | After the container starts | Before the container starts |
| **Did the image pull succeed?** | ✅ Yes — image is on the node | ❌ No — image never downloaded |
| **Did the container ever run?** | ✅ Yes — started and crashed | ❌ No — never started |
| **RESTARTS count** | High — keeps incrementing | 0 — container never ran |
| **First diagnostic command** | `kubectl logs <pod> --previous` | `kubectl describe pod` → Events |
| **Root cause area** | Application, config, resources, probes | Registry, credentials, image name, network |
| **Common causes** | Missing env var, OOM, wrong CMD, app bug, bad liveness probe | Wrong image tag, missing pull secret, registry unreachable |
| **Fix involves** | Fixing the application or its configuration | Fixing the image reference or registry access |

---

**How to tell them apart instantly — look at RESTARTS:**

```bash
kubectl get pods

NAME                          READY   STATUS             RESTARTS   AGE
user-service-7d9f8b-xk2p9    0/1     CrashLoopBackOff   8          25m
order-service-5c6d7e-mn3q1   0/1     ImagePullBackOff   0          5m
```

- `RESTARTS: 8` → `CrashLoopBackOff` — the container has started and crashed 8 times
- `RESTARTS: 0` → `ImagePullBackOff` — the container never started even once

**RESTARTS will always be 0 for ImagePullBackOff** — because the container never ran, it never had a chance to crash and be restarted.

---

**How to diagnose each one:**

**For CrashLoopBackOff:**
```bash
# Step 1 — Get logs from the crashed container
kubectl logs <pod-name> --previous

# Step 2 — Check exit code and events
kubectl describe pod <pod-name>
# Look for: Exit Code, Last State, OOMKilled, liveness probe failures
```

**For ImagePullBackOff:**
```bash
# Step 1 — Describe the pod immediately — logs are useless here (container never ran)
kubectl describe pod <pod-name>
# Look for: Events section — "Failed to pull image" with the exact reason

# Step 2 — Verify the image exists in the registry
az acr repository show-tags --name azureshopacr --repository user-service
```

**Key point:** For `ImagePullBackOff`, `kubectl logs` gives you nothing — there are no logs because the container never ran. `kubectl describe` is your only tool. For `CrashLoopBackOff`, `kubectl logs --previous` is your most important tool.

---

**What they have in common — the BackOff:**

Both use the same **exponential backoff** mechanism — Kubernetes waits increasingly longer between retries:

```
10s → 20s → 40s → 80s → 160s → 300s (max — 5 minutes)
```

This is why both say "BackOff" — Kubernetes is backing off before retrying to avoid hammering a broken system. The pod stays stuck in this retry loop until you fix the root cause.

---

**Quick mental model — which question to ask first:**

```
Pod is not running — what do I ask?
          |
          ↓
Did the image pull succeed?
(Check RESTARTS — is it 0?)
          |
    ______|______
   |             |
  YES            NO
RESTARTS > 0   RESTARTS = 0
   |             |
   ↓             ↓
CrashLoop     ImagePull
BackOff       BackOff
   |             |
kubectl logs  kubectl describe
--previous    → Events section
```

---

**AzureShop Example — both happening at the same time:**

After a major release in AzureShop, two different services had issues:

- `payment-service` showed `ImagePullBackOff` with `RESTARTS: 0` — the pipeline had pushed the image to ACR with tag `v3.0.0` but the deployment manifest referenced `v3.1.0` which did not exist. Fix: corrected the tag in the deployment.

- `order-service` showed `CrashLoopBackOff` with `RESTARTS: 5` — the image pulled successfully but the service crashed on startup because a new required environment variable `PAYMENT_SERVICE_URL` was missing from the Kubernetes Secret. Fix: added the missing env var.

Both looked similar from `kubectl get pods` but required completely different investigations and fixes.

---

**Summary (what to say if time is short):**

*"The key difference is where in the pod lifecycle the failure happens. ImagePullBackOff fails before the container ever starts — Kubernetes cannot pull the image from the registry due to a wrong image tag, missing credentials, or registry being unreachable. The RESTARTS count stays at 0 because the container never ran. CrashLoopBackOff fails after the container starts — the image pulled successfully, the container started, but the application inside crashed due to a missing env var, OOM kill, wrong command, or application bug. The RESTARTS count keeps climbing. This difference tells you exactly where to look — for ImagePullBackOff, kubectl logs is useless and kubectl describe Events is everything. For CrashLoopBackOff, kubectl logs --previous is the most important command. Both use the same exponential backoff between retries — 10s, 20s, 40s, up to 5 minutes."*

---

<!-- Add more questions as Q5, Q6... -->

---

### Scenario Based Questions

> These are real scenario questions asked by the interviewer to test your depth of knowledge and real-world troubleshooting ability. The interviewer wants to see how many possible reasons you can think of and how you would verify each one.

---

#### Scenario 1. ImagePullBackOff — Possible Reasons?

**Answer:**

When an interviewer asks for all possible reasons for `ImagePullBackOff`, they want to see the breadth of your knowledge — how many root causes you can identify and explain. This is a checklist question. Let me give you every possible reason, clearly explained.

The first thing I always do is run:

```bash
kubectl describe pod <pod-name>
```

The **Events** section at the bottom tells me the exact error message — and that message points directly to the reason. Let me walk through every possible reason you might see.

---

**Reason 1 — Wrong Image Name**

The image name in the deployment spec is misspelled or points to an image that does not exist in the registry.

```yaml
image: azureshopacr.azurecr.io/user-servce:v2.1.0
#                                    ↑ typo — "servce" instead of "service"
```

**Event message:**
```
Failed to pull image: not found
```

**How to verify:**
```bash
kubectl describe pod <pod-name> | grep Image
az acr repository list --name azureshopacr --output table
```

**Fix:** Correct the image name in the deployment spec.

---

**Reason 2 — Wrong or Non-Existent Tag**

The image exists in the registry but the specific tag referenced in the deployment does not exist — it was never pushed, deleted, or has a typo.

```yaml
image: azureshopacr.azurecr.io/user-service:v99.0.0
#                                              ↑ this tag was never pushed
```

**Event message:**
```
Failed to pull image "azureshopacr.azurecr.io/user-service:v99.0.0": not found
```

**How to verify:**
```bash
az acr repository show-tags \
  --name azureshopacr \
  --repository user-service \
  --output table
```

**Fix:** Push the correct image with the right tag, or update the deployment to use an existing tag.

**Important:** Using `latest` tag is a common cause of this in pipelines — if `latest` was not explicitly pushed after the build, the tag may not exist or may point to an old image. Always use specific version tags in production.

---

**Reason 3 — Missing imagePullSecret — Private Registry Authentication Failed**

The container registry is private (ACR, ECR, GCR, private Docker Hub). Kubernetes has no credentials to authenticate — the registry rejects the pull with `401 Unauthorized` or `403 Forbidden`.

**Event message:**
```
Failed to pull image: unauthorized: authentication required
# or
Failed to pull image: 403 Forbidden
```

**How to verify:**
```bash
# Check if imagePullSecrets is defined in the pod spec
kubectl get pod <pod-name> -o yaml | grep -A5 imagePullSecrets

# Check if the secret exists in the namespace
kubectl get secrets -n <namespace>
```

**Fix:** Create the imagePullSecret and reference it in the deployment:

```bash
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<password> \
  -n production
```

```yaml
spec:
  imagePullSecrets:
    - name: acr-pull-secret
  containers:
    - name: user-service
      image: azureshopacr.azurecr.io/user-service:v2.1.0
```

---

**Reason 4 — imagePullSecret Exists but Has Wrong or Expired Credentials**

The Secret exists — but the credentials inside it are wrong, expired, or the ACR password was rotated without updating the Secret.

**Event message:**
```
Failed to pull image: unauthorized: authentication required
```

Looks identical to Reason 3 but the Secret does exist. The credentials inside are stale.

**How to verify:**
```bash
# Decode the secret and inspect credentials
kubectl get secret acr-pull-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

# Test the credentials manually
docker login azureshopacr.azurecr.io \
  --username azureshopacr \
  --password <decoded-password>
```

**Fix:** Delete the old Secret and recreate it with fresh credentials:

```bash
kubectl delete secret acr-pull-secret -n production

kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<new-password> \
  -n production
```

---

**Reason 5 — AcrPull Role Not Assigned to AKS Managed Identity**

In Azure, if using managed identity instead of imagePullSecrets — the AKS kubelet identity must have the `AcrPull` role on the ACR. If this role assignment is missing or was removed, every pod in the cluster gets `ImagePullBackOff`.

**Event message:**
```
Failed to pull image: unauthorized: authentication required
```

**How to verify:**
```bash
# Get the kubelet identity object ID
az aks show \
  --resource-group azureshop-rg \
  --name azureshop-aks \
  --query identityProfile.kubeletidentity.objectId \
  --output tsv

# Check if AcrPull role assignment exists
az role assignment list \
  --assignee <kubelet-object-id> \
  --role AcrPull \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/azureshopacr
```

If the output is empty — the role assignment is missing.

**Fix:**
```bash
az role assignment create \
  --assignee <kubelet-object-id> \
  --role AcrPull \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/azureshopacr
```

**AzureShop Example:** After an AKS version upgrade, the kubelet identity changed. The old AcrPull role assignment was on the old identity. All pods across all namespaces showed `ImagePullBackOff` simultaneously. This was the cause — one Terraform apply fixed it in 2 minutes.

---

**Reason 6 — Image in Wrong Registry or Wrong Registry URL**

The deployment points to the correct image name and tag — but the registry URL itself is wrong. For example, pointing to Docker Hub when the image is in ACR, or a typo in the registry hostname.

```yaml
# Wrong — pointing to Docker Hub
image: user-service:v2.1.0

# Wrong — typo in registry URL
image: azureshopacr.azurecr.com/user-service:v2.1.0
#                           ↑ .com instead of .io

# Correct
image: azureshopacr.azurecr.io/user-service:v2.1.0
```

**Event message:**
```
Failed to pull image "user-service:v2.1.0":
pull access denied, repository does not exist or may require 'docker login'
```

**Fix:** Use the fully qualified image name including the registry URL.

---

**Reason 7 — Registry is Unreachable — Network / Firewall Issue**

The AKS nodes cannot reach the container registry due to a firewall rule, NSG, or DNS configuration blocking outbound traffic on port 443.

**Event message:**
```
Failed to pull image: dial tcp: lookup azureshopacr.azurecr.io: no such host
# or
Failed to pull image: context deadline exceeded
# or
Failed to pull image: connection refused
```

**How to verify:**
```bash
# Run a debug pod and test connectivity from inside the cluster
kubectl run debug --image=busybox --restart=Never -it -- sh

# Inside the pod:
nslookup azureshopacr.azurecr.io
wget --spider https://azureshopacr.azurecr.io/v2/
```

**Common causes in Azure:**
- NSG rule blocking outbound 443 from the AKS subnet
- Azure Firewall blocking the registry FQDN
- ACR has Private Endpoint enabled but AKS VNet is not linked to the Private DNS Zone
- Corporate proxy blocking traffic

**Fix:** Open port 443 outbound in NSG or Firewall rules, or configure the private DNS zone link.

---

**Reason 8 — ACR Private Endpoint — DNS Not Configured Correctly**

ACR is configured with a Private Endpoint for security — it should only be accessible from within the VNet. But the Private DNS Zone resolving `azureshopacr.azurecr.io` to the private IP is either missing or not linked to the AKS VNet.

AKS nodes resolve `azureshopacr.azurecr.io` to the public IP (which is now blocked) instead of the private IP.

**Event message:**
```
Failed to pull image: connection refused
# or dial tcp <public-ip>:443: connection refused
```

**How to verify:**
```bash
# From inside a pod or node — check what IP is resolved
nslookup azureshopacr.azurecr.io
# If it returns the public IP (e.g. 20.x.x.x) instead of private (e.g. 10.x.x.x)
# → DNS is not resolving to the private endpoint
```

**Fix:** Link the Private DNS Zone to the AKS VNet:

```hcl
resource "azurerm_private_dns_zone_virtual_network_link" "acr_aks_link" {
  name                  = "acr-aks-dns-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.acr.name
  virtual_network_id    = azurerm_virtual_network.aks.id
}
```

---

**Reason 9 — Image Exists but is in a Different Namespace's Secret**

The imagePullSecret was created in the `default` namespace but the pod is running in the `production` namespace. Secrets are namespace-scoped — they are not shared across namespaces.

**Event message:**
```
Failed to pull image: unauthorized: authentication required
```

Even though the secret `acr-pull-secret` exists — it exists in a different namespace.

**How to verify:**
```bash
# Check which namespace the pod is in
kubectl get pod <pod-name> -o jsonpath='{.metadata.namespace}'

# Check if the secret exists IN THAT namespace
kubectl get secrets -n production | grep acr-pull-secret
```

**Fix:** Create the Secret in the correct namespace:

```bash
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<password> \
  -n production    # Must be in the same namespace as the pod
```

---

**Reason 10 — Rate Limiting (Docker Hub)**

If pulling from Docker Hub (not a private registry), Docker Hub enforces rate limits — 100 pulls per 6 hours for anonymous users, 200 for free accounts. If the AKS cluster is pulling many images from Docker Hub, it hits this limit and gets `429 Too Many Requests`.

**Event message:**
```
Failed to pull image: toomanyrequests: You have reached your pull rate limit.
```

**Fix:**
- Authenticate to Docker Hub using an imagePullSecret with a paid account
- Mirror commonly used public images to your private ACR and pull from there
- Use `imagePullPolicy: IfNotPresent` to avoid pulling the same image repeatedly

```bash
# Import a public image into ACR to avoid Docker Hub rate limits
az acr import \
  --name azureshopacr \
  --source docker.io/library/nginx:alpine \
  --image nginx:alpine
```

---

**Reason 11 — Node Disk Pressure — No Space to Store the Image**

The AKS node is running low on disk space. It cannot store the pulled image locally.

**Event message:**
```
Failed to pull image: write /var/lib/containerd/...: no space left on device
```

**How to verify:**
```bash
# Check node conditions
kubectl describe node <node-name> | grep -A5 Conditions

# Look for:
# DiskPressure = True
```

**Fix:**
- Delete unused images on the node: `docker image prune` (or `crictl rmi --prune` for containerd)
- Increase the node disk size
- Enable AKS node image garbage collection
- Add more nodes via cluster autoscaler to distribute the load

---

**All possible reasons at a glance:**

| # | Reason | Event Message Clue |
|---|---|---|
| 1 | Wrong image name | `not found` |
| 2 | Wrong or non-existent tag | `not found` |
| 3 | Missing imagePullSecret | `unauthorized` / `403` |
| 4 | Expired/wrong credentials in Secret | `unauthorized` |
| 5 | AcrPull role missing on AKS identity | `unauthorized` |
| 6 | Wrong registry URL | `pull access denied` |
| 7 | Registry unreachable — network/firewall | `deadline exceeded` / `no such host` |
| 8 | ACR private endpoint — DNS not linked | `connection refused` |
| 9 | Secret in wrong namespace | `unauthorized` |
| 10 | Docker Hub rate limit | `toomanyrequests` |
| 11 | Node disk pressure | `no space left on device` |

---

**The event message is the key — always read it carefully:**

```bash
kubectl describe pod <pod-name>
```

```
Events:
  Warning  Failed  2m  kubelet  Failed to pull image "...": <THIS MESSAGE TELLS YOU EVERYTHING>
```

Each different message points to a different reason. Never guess — always read the event message first.

---

**Summary (what to say if time is short):**

*"There are 11 possible reasons for ImagePullBackOff. The most common are — wrong image name or tag that does not exist in the registry, missing or expired imagePullSecret credentials for a private registry, the AcrPull role not being assigned to the AKS kubelet managed identity, the registry being unreachable due to a firewall or NSG rule, ACR private endpoint with DNS not linked to the AKS VNet, the imagePullSecret existing in a different namespace than the pod, and Docker Hub rate limiting for public images. The first thing I always do is run `kubectl describe pod` and read the Events section carefully — the exact error message tells me which of these 11 reasons is causing the problem. `unauthorized` means credentials, `not found` means wrong name or tag, `deadline exceeded` or `no such host` means network or DNS, `toomanyrequests` means rate limiting, and `no space left on device` means disk pressure on the node."*

---

#### Scenario 2. (Advanced) Same image, same tag, same registry — works for another app. Only your application gets ImagePullBackOff. What could be the reason?

**Answer:**

This is an excellent advanced question. The interviewer has eliminated all the obvious causes — the image is correct, the tag exists, the registry is reachable because another application is pulling successfully from the same place. So the problem is not with the image or registry at all.

**The problem must be something specific to YOUR application or the environment it runs in.**

This narrows the investigation to differences between your application and the working one. Let me think through every possible difference systematically.

---

**My first question to the interviewer — where is the working application running?**

Before checking anything, I ask:
- Is the working application in the **same namespace** or a different one?
- Is it running on the **same node** or a different one?
- Does it use the **same service account**?

The answer to these three questions immediately points me to the right area to investigate.

---

**Reason 1 — Different Namespace — imagePullSecret Not Present in YOUR Namespace**

This is the most common reason. The working application is in the `default` namespace (or another namespace) where the `imagePullSecret` exists. Your application is in the `production` namespace where the Secret does not exist.

**Secrets in Kubernetes are namespace-scoped — they do not cross namespace boundaries.**

```bash
# Working app namespace
kubectl get secrets -n default | grep pull-secret
# acr-pull-secret    kubernetes.io/dockerconfigjson   1      30d  ← EXISTS

# Your app namespace
kubectl get secrets -n production | grep pull-secret
# (nothing) ← DOES NOT EXIST
```

**How to verify:**
```bash
# Check which namespace your pod is in
kubectl get pod <your-pod-name> -o jsonpath='{.metadata.namespace}'

# Check if the pull secret exists in THAT namespace
kubectl get secrets -n <your-namespace> | grep pull-secret
```

**Fix — create the Secret in your namespace:**
```bash
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<password> \
  -n production    # Must match your pod's namespace
```

Or if you have many namespaces, automate this with a script:
```bash
for ns in production staging dev; do
  kubectl create secret docker-registry acr-pull-secret \
    --docker-server=azureshopacr.azurecr.io \
    --docker-username=azureshopacr \
    --docker-password=<password> \
    -n $ns
done
```

---

**Reason 2 — Different Service Account — imagePullSecret Attached to a Different Service Account**

The working application uses a service account that has `imagePullSecrets` attached to it. Your application uses a different service account (or the `default` service account) that does not have the pull secret attached.

In Kubernetes, you can patch a service account to automatically inject pull secrets into every pod that uses it — without needing to define `imagePullSecrets` in every deployment.

```bash
# Working app — uses service account "app-sa" which has pull secret attached
kubectl get serviceaccount app-sa -n production -o yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
imagePullSecrets:
  - name: acr-pull-secret    # ← pull secret is attached here
```

```bash
# Your app — uses "default" service account which has NO pull secret
kubectl get serviceaccount default -n production -o yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
# No imagePullSecrets here
```

**How to verify:**
```bash
# Check which service account your pod uses
kubectl get pod <your-pod-name> -o jsonpath='{.spec.serviceAccountName}'

# Check if that service account has imagePullSecrets
kubectl get serviceaccount <sa-name> -n <namespace> -o yaml | grep -A3 imagePullSecrets
```

**Fix Option 1 — Add imagePullSecrets directly to your deployment:**
```yaml
spec:
  imagePullSecrets:
    - name: acr-pull-secret
  containers:
    - name: your-app
      image: azureshopacr.azurecr.io/user-service:v2.1.0
```

**Fix Option 2 — Patch the default service account to include pull secrets:**
```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "acr-pull-secret"}]}' \
  -n production
```

Now every pod in the `production` namespace that uses the `default` service account automatically gets the pull secret — no changes to individual deployments needed.

---

**Reason 3 — Pod Scheduled on a Problematic Node**

The working application's pods are on healthy nodes. Your application's pod is scheduled on a specific node that has a problem — disk pressure, network issue, or a corrupted container runtime credential cache.

The image pulls fine on other nodes — but fails on this specific node.

**How to verify:**
```bash
# Find which node your pod is scheduled on
kubectl get pod <your-pod-name> -o wide
# OUTPUT: NAME   READY  STATUS             NODE
#         ...    0/1    ImagePullBackOff   aks-nodepool-12345-3   ← problematic node

# Check the node's conditions
kubectl describe node aks-nodepool-12345-3 | grep -A10 Conditions
```

Look for:
```
DiskPressure    True    → Node is running out of disk space — cannot store the image
NetworkUnavailable  True  → Node has network issues — cannot reach registry
MemoryPressure  True    → Node is under memory pressure
```

**If it is a disk issue on the node:**
```bash
# Force the pod to reschedule on a different node
kubectl delete pod <your-pod-name>
# Kubernetes will reschedule it — if it lands on a healthy node, it will work

# Cordon the problematic node so no new pods land on it
kubectl cordon aks-nodepool-12345-3
```

**If the node has a corrupted credential cache:**
```bash
# SSH into the node and clear the container runtime credential cache
# For containerd:
sudo crictl rmi --prune
sudo systemctl restart containerd

# Alternatively — drain and replace the node
kubectl drain aks-nodepool-12345-3 --ignore-daemonsets --delete-emptydir-data
# In AKS — delete the node and let the node pool recreate it
az aks nodepool scale --resource-group azureshop-rg --cluster-name azureshop-aks \
  --name nodepool1 --node-count 3
```

---

**Reason 4 — Network Policy Blocking Egress from Your Pod**

Kubernetes **NetworkPolicies** control which pods can send traffic where. The working application may have no NetworkPolicy restrictions, or its NetworkPolicy allows egress to the registry. Your application may be in a namespace or pod with a restrictive NetworkPolicy that blocks outbound traffic — including to the registry.

**How to verify:**
```bash
# Check if any NetworkPolicy exists in your namespace
kubectl get networkpolicy -n <your-namespace>

# Describe the policy and read the egress rules
kubectl describe networkpolicy <policy-name> -n <your-namespace>
```

Example of a restrictive policy that blocks registry access:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector: {}    # Applies to ALL pods in namespace
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system    # Only allows traffic to kube-system
      ports:
        - port: 53    # DNS only
# No rule allowing 443 to the registry → image pull is blocked
```

**Fix — add an egress rule allowing HTTPS to the registry:**
```yaml
egress:
  - to: []    # Allow all destinations
    ports:
      - port: 443    # HTTPS — needed for registry pulls
        protocol: TCP
  - ports:
      - port: 53    # DNS
        protocol: UDP
```

---

**Reason 5 — Admission Webhook / OPA Policy Blocking the Pod**

Your cluster has an **OPA (Open Policy Agent) Gatekeeper** or a **custom admission webhook** that enforces policies. The working application satisfies the policy. Your application violates it — for example:
- Your image is not from an approved registry list
- Your pod does not have required labels
- Your container runs as root and a policy blocks root containers

The pod is created, but the admission webhook rejects the image pull or mutates the pod in a way that breaks it.

**How to verify:**
```bash
# Check for admission webhooks
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Check OPA Gatekeeper constraints
kubectl get constraints

# Look for policy violations
kubectl describe pod <your-pod-name>
# Events may show: admission webhook denied the request
```

**Example OPA constraint that could cause this:**
```yaml
# Policy: only images from approved registries are allowed
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRepos
metadata:
  name: allowed-repos
spec:
  parameters:
    repos:
      - "azureshopacr.azurecr.io"
      - "mcr.microsoft.com"
```

If your image reference is `user-service:v2.1.0` (without the full registry URL), Gatekeeper rejects it because it does not start with an approved registry prefix.

**Fix:** Use the fully qualified image name including the registry URL:
```yaml
image: azureshopacr.azurecr.io/user-service:v2.1.0    # Correct — passes OPA check
```

---

**Reason 6 — imagePullPolicy: Always on a Node Without Outbound Internet**

The working application uses `imagePullPolicy: IfNotPresent` — it uses the cached image already on the node. Your application uses `imagePullPolicy: Always` — it tries to pull from the registry every time, even when the image is cached.

If the node this pod lands on has restricted outbound internet (like a private AKS cluster), `Always` fails while `IfNotPresent` works fine on the same node.

**How to verify:**
```bash
# Check imagePullPolicy of your pod vs the working pod
kubectl get pod <your-pod> -o jsonpath='{.spec.containers[0].imagePullPolicy}'
# OUTPUT: Always  ← this is pulling every time

kubectl get pod <working-pod> -o jsonpath='{.spec.containers[0].imagePullPolicy}'
# OUTPUT: IfNotPresent  ← this uses cache
```

**Fix:**
```yaml
spec:
  containers:
    - name: your-app
      image: azureshopacr.azurecr.io/user-service:v2.1.0
      imagePullPolicy: IfNotPresent    # Use cache if available
```

---

**Reason 7 — Resource Quota Preventing the Pull**

The namespace has a **ResourceQuota** that limits the number of pods, secrets, or compute resources. Your pod is hitting the quota — Kubernetes cannot fully initialise it, causing the image pull to fail indirectly.

**How to verify:**
```bash
kubectl describe resourcequota -n <your-namespace>
```

```
Name:       production-quota
Namespace:  production
Resource    Used   Hard
--------    ----   ----
pods        10     10      ← at the limit — cannot create new pods
secrets     5      5       ← at the limit — cannot create imagePullSecret
```

**Fix:**
```bash
# Increase the quota
kubectl edit resourcequota production-quota -n production

# Or delete unused pods/secrets to free up quota
kubectl delete pod <old-pod> -n production
```

---

**Complete thought process — how I approach this in the interview:**

When the interviewer says "same image, same tag, same registry, other app works" — I immediately think:

```
The image and registry are fine → problem is in MY APPLICATION'S environment

Ask: Same namespace?
  → No  → Secret may not exist in my namespace (Reason 1)

Ask: Same service account?
  → No  → SA may not have pull secret attached (Reason 2)

Ask: Same node?
  → No  → Node may have disk/network issues (Reason 3)

Ask: Any NetworkPolicy in my namespace?
  → Yes → Egress to registry may be blocked (Reason 4)

Ask: Any OPA/admission webhooks?
  → Yes → Policy may be rejecting my image reference (Reason 5)

Ask: What is my imagePullPolicy?
  → Always → Must pull from registry every time, other app uses IfNotPresent (Reason 6)

Ask: Any ResourceQuota on my namespace?
  → Yes → May be hitting quota limits (Reason 7)
```

---

**AzureShop Real Example:**

In AzureShop, we deployed a new `analytics-service` to the `analytics` namespace. The `user-service` in the `production` namespace was pulling the same base image (`node:18-alpine`) from our internal ACR mirror successfully. But `analytics-service` kept getting `ImagePullBackOff`.

Root cause — **Reason 1**: The `acr-pull-secret` existed in the `production` namespace but we had forgotten to create it in the `analytics` namespace. The `analytics` namespace was newly created and had no secrets.

The fix took 30 seconds:
```bash
kubectl create secret docker-registry acr-pull-secret \
  --docker-server=azureshopacr.azurecr.io \
  --docker-username=azureshopacr \
  --docker-password=<password> \
  -n analytics
```

After this incident, we added a step in our namespace creation runbook:
> **Whenever a new namespace is created, immediately copy the `acr-pull-secret` into it.**

We later automated this with an **OPA Gatekeeper mutation** that automatically injects `imagePullSecrets` into every pod across all namespaces — so no deployment ever needs to specify it manually.

---

**Summary (what to say if time is short):**

*"If the same image works for another application but only mine gets ImagePullBackOff, the problem is not with the image or registry — it is something specific to my application's environment. There are 7 possible reasons. First and most common — the imagePullSecret exists in the other app's namespace but not in mine — Secrets are namespace-scoped and do not cross namespaces. Second — the other app uses a service account that has the pull secret attached, mine uses a different service account without it. Third — my pod is scheduled on a specific node that has disk pressure, network issues, or a corrupted credential cache — image pulls succeed on all other nodes. Fourth — a NetworkPolicy in my namespace blocks egress traffic on port 443 to the registry. Fifth — an OPA Gatekeeper policy rejects my image reference because it is not fully qualified with the registry URL. Sixth — my deployment uses `imagePullPolicy: Always` while the other uses `IfNotPresent` — and my node cannot reach the registry. Seventh — a ResourceQuota in my namespace is preventing pod or Secret creation. In AzureShop, we hit Reason 1 when we forgot to create the pull secret in a new namespace — fixed in 30 seconds and then automated with OPA mutation to prevent recurrence."*

---

<!-- Add more scenario questions as Scenario 3, Scenario 4... -->

---

<!--
To add a new interview, copy the block below and paste it at the bottom:

## Interview #2

**Company:**
**Date:**
**Role Applied For:**
**Round:**
**Interviewer Level:**

### Questions Asked

#### Q1.

**Answer:**

---
-->
