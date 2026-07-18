# Docker — Real Interview Questions & Answers

> This file is a personal log of actual Docker questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Capgemini | Azure DevOps | Technical Round 1](#interview-1)
  - [Q1. Have you created Dockerfiles? Which instructions are commonly used?](#q1-have-you-created-dockerfiles-which-dockerfile-instructions-are-commonly-used)
  - [Q2. What happens if you write only FROM without specifying a base image?](#q2-what-happens-if-you-write-only-from-without-specifying-a-base-image)
  - [Q3. Will Docker throw an error during runtime or build time?](#q3-will-docker-throw-an-error-during-runtime-or-build-time)
  - [Q4. Your Docker image size is 1.5 GB. How will you reduce it?](#q4-your-docker-image-size-is-15-gb-how-will-you-reduce-the-image-size)
  - [Q5. What is Multi-stage Build? Have you implemented it? Why do we use it?](#q5-what-is-multi-stage-build-have-you-implemented-multi-stage-build-in-your-project-why-do-we-use-multi-stage-build)
  - [Scenario 1. Application Container cannot connect to Database Container](#scenario-1-application-container-cannot-connect-to-database-container-both-containers-are-running-on-the-same-host-getting-connection-refused-how-will-you-troubleshoot)
  - [Scenario 2. Containers running, network fine, config correct — still Connection Refused](#scenario-2-interviewer-removed-all-obvious-causes----containers-running-network-fine-configuration-correct----still-getting-connection-refused-what-will-you-check-now)

---

## Interview #1

**Company:** Capgemini  
**Date:** 18-07-2026  
**Role Applied For:** Azure DevOps  
**Round:** Technical Round 1  
**Interviewer Level:** Tech Lead

---

### Questions Asked

#### Q1. Have you created Dockerfiles? Which Dockerfile instructions are commonly used?

**Answer:**

**Yes, absolutely. Writing Dockerfiles is something I do regularly as a DevOps Engineer.**

In the AzureShop project, I wrote Dockerfiles for multiple microservices — user-service, order-service, payment-service, and product-service. Each service had its own Dockerfile that packaged the application code, its dependencies, and its runtime into a Docker image that could run consistently anywhere — on a developer's laptop, in a CI pipeline, or in an AKS cluster in production.

Let me explain all the commonly used Dockerfile instructions clearly with real examples.

---

**What is a Dockerfile?**

A Dockerfile is a plain text file with a set of instructions that tells Docker how to build an image. Think of it like a recipe — the recipe lists every step needed to prepare the dish. Docker follows the recipe step by step and produces a Docker image at the end.

Every instruction in a Dockerfile creates a new **layer** in the image. Docker caches these layers — so if a layer has not changed, Docker reuses the cached version instead of rebuilding it. This makes builds faster.

---

**The most commonly used Dockerfile instructions:**

---

**1. `FROM` — Base Image**

Every Dockerfile must start with `FROM`. It defines the base image — the starting point your image is built on top of.

```dockerfile
FROM node:18-alpine
```

This means — start from the official Node.js 18 image built on Alpine Linux (a very lightweight Linux distribution — only 5MB).

**Real AzureShop example:**

```dockerfile
# For our Node.js user-service
FROM node:18-alpine

# For our Java order-service
FROM eclipse-temurin:17-jre-alpine

# For our Python payment-service
FROM python:3.11-slim
```

**Best practice:** Always pin a specific version tag — never use `FROM node:latest`. If you use `latest`, the base image can change without warning and break your build.

---

**2. `WORKDIR` — Set Working Directory**

Sets the working directory inside the container. All subsequent instructions run from this directory.

```dockerfile
WORKDIR /app
```

This is like doing `mkdir /app && cd /app` inside the container. If the directory does not exist, Docker creates it.

**Why use it:** Without `WORKDIR`, files land in the root `/` directory which gets messy. `WORKDIR` keeps everything organised in one place.

---

**3. `COPY` — Copy Files into the Image**

Copies files from your local machine (build context) into the image.

```dockerfile
# Copy a single file
COPY package.json .

# Copy everything from current directory into WORKDIR
COPY . .

# Copy from a specific source to a specific destination
COPY src/ /app/src/
```

**`COPY` vs `ADD`:**

Both copy files but `COPY` is preferred for simple file copying. `ADD` has extra features — it can extract `.tar` files and download from URLs — but those extra powers make it unpredictable. Use `COPY` unless you specifically need `ADD`'s extra features.

---

**4. `RUN` — Execute Commands During Build**

Runs a command inside the image during the build process. Used to install dependencies, update packages, compile code — anything that needs to happen to set up the image.

```dockerfile
# Install Node.js dependencies
RUN npm install

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Install system packages
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

**Important:** Each `RUN` instruction creates a new layer. To keep the image small, chain related commands with `&&` in a single `RUN`:

```dockerfile
# Bad — creates 3 separate layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — one layer, smaller image
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

---

**5. `ENV` — Set Environment Variables**

Sets environment variables inside the container. These variables are available both during the build and when the container runs.

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ENV APP_VERSION=1.0.0
```

**Why use it:** Applications read environment variables for configuration — database URLs, port numbers, feature flags. Setting them in the Dockerfile provides sensible defaults.

```dockerfile
# Then your app can read it
# process.env.PORT → 3000
# process.env.NODE_ENV → production
```

---

**6. `ARG` — Build-Time Arguments**

Similar to `ENV` but only available during the **build** — not when the container runs. Used to pass values into the Dockerfile at build time.

```dockerfile
ARG APP_VERSION=1.0.0
ARG BUILD_DATE

RUN echo "Building version ${APP_VERSION} on ${BUILD_DATE}"
```

Pass the value when building:
```bash
docker build --build-arg APP_VERSION=2.1.0 --build-arg BUILD_DATE=2026-07-18 .
```

**`ARG` vs `ENV`:**
- `ARG` — available only during build, not in the running container
- `ENV` — available both during build and in the running container
- Never use `ARG` for secrets — they are visible in `docker history`

---

**7. `EXPOSE` — Document the Port**

Documents which port the application inside the container listens on.

```dockerfile
EXPOSE 3000
```

**Important:** `EXPOSE` does NOT actually publish the port. It is documentation — it tells other developers and tools which port to map. The actual port mapping happens when you run the container:

```bash
docker run -p 8080:3000 my-app
# 8080 on your machine → 3000 inside the container
```

---

**8. `CMD` — Default Command to Run the Container**

Specifies the default command to run when the container starts.

```dockerfile
CMD ["node", "server.js"]
```

**Important about `CMD`:**
- There should only be one `CMD` in a Dockerfile — if you write multiple, only the last one takes effect
- `CMD` can be overridden at runtime:

```bash
docker run my-app node debug.js    # Overrides CMD — runs debug.js instead of server.js
```

---

**9. `ENTRYPOINT` — Fixed Command for the Container**

Similar to `CMD` but cannot be overridden as easily. Sets the main executable for the container.

```dockerfile
ENTRYPOINT ["node", "server.js"]
```

**`CMD` vs `ENTRYPOINT`:**

| | CMD | ENTRYPOINT |
|---|---|---|
| Can be overridden at `docker run`? | ✅ Yes — easily | ❌ No — only with `--entrypoint` flag |
| Best used for | Default command that can be replaced | Fixed executable that always runs |
| Common pattern | Use with ENTRYPOINT to pass default args | Set the main process |

**Common pattern — use both together:**

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```

- `docker run my-app` → runs `node server.js`
- `docker run my-app debug.js` → runs `node debug.js` (CMD overridden, ENTRYPOINT stays)

---

**10. `USER` — Run as Non-Root User**

Sets the user the container runs as. By default, Docker containers run as `root` — which is a security risk.

```dockerfile
# Create a non-root user and switch to it
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**Why this matters:** If a container running as root is compromised, the attacker has root access to the host system. Running as a non-root user limits the blast radius.

In AzureShop, our security scanning tool (Checkov) flagged any Dockerfile that ran as root. Every Dockerfile had a `USER` instruction.

---

**11. `VOLUME` — Define Mount Points**

Declares a directory inside the container as a mount point for persistent data or external volumes.

```dockerfile
VOLUME ["/app/data"]
VOLUME ["/var/log/app"]
```

Used when the container needs to persist data beyond its lifecycle — like log files or uploaded content.

---

**12. `HEALTHCHECK` — Container Health Monitoring**

Tells Docker how to test if the container is healthy and running correctly.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

- Every 30 seconds, Docker runs `curl http://localhost:3000/health`
- If it fails 3 times in a row, Docker marks the container as `unhealthy`
- Kubernetes uses this to decide whether to restart a pod

---

**Complete real Dockerfile from AzureShop — user-service:**

```dockerfile
# Stage 1 — Build
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY src/ ./src/

# Stage 2 — Production image
FROM node:18-alpine AS production

# Security — create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only what is needed from the build stage
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/src ./src
COPY package.json .

# Set environment
ENV NODE_ENV=production
ENV PORT=3000

# Document the port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Run as non-root
USER appuser

# Start the application
CMD ["node", "src/server.js"]
```

This is a **multi-stage build** — the first stage installs dependencies and builds the app, the second stage copies only the final output into a clean, minimal image. The final image has no build tools, no source files beyond what is needed — just the running application. This keeps the production image small and secure.

---

**Quick reference — all instructions:**

| Instruction | Purpose |
|---|---|
| `FROM` | Set the base image |
| `WORKDIR` | Set the working directory inside the container |
| `COPY` | Copy files from local machine into the image |
| `RUN` | Execute commands during image build |
| `ENV` | Set environment variables (build + runtime) |
| `ARG` | Set build-time arguments (build only) |
| `EXPOSE` | Document which port the app listens on |
| `CMD` | Default command to run (can be overridden) |
| `ENTRYPOINT` | Fixed main process (cannot be easily overridden) |
| `USER` | Run container as non-root user |
| `VOLUME` | Define mount points for persistent data |
| `HEALTHCHECK` | Define how Docker checks if container is healthy |

---

**Summary (what to say if time is short):**

*"Yes, I have written Dockerfiles for all the microservices in the AzureShop project. The most commonly used instructions are — `FROM` to set the base image (always with a pinned version, never latest), `WORKDIR` to set the working directory, `COPY` to bring files into the image, `RUN` to install dependencies during build, `ENV` for environment variables, `EXPOSE` to document the port, `CMD` or `ENTRYPOINT` to define what runs when the container starts, `USER` to run as a non-root user for security, and `HEALTHCHECK` so Kubernetes knows if the container is healthy. In AzureShop we also used multi-stage builds — a builder stage to install dependencies and compile, and a clean production stage that only contains the final output — keeping the production image small and secure."*

---

#### Q2. What happens if you write only `FROM` without specifying a base image?

**Answer:**

This question has two parts to it — and understanding both shows the interviewer you really know Docker deeply. Let me explain both scenarios.

---

**Scenario A — Writing `FROM` with absolutely nothing after it**

If you write just `FROM` with no image name at all:

```dockerfile
FROM
```

Docker will throw a **build error** immediately:

```
Error response from daemon: dockerfile parse error on line 1:
FROM requires either one or three arguments
```

`FROM` is a mandatory instruction and it requires at least one argument — the image name. Without it, the Dockerfile is syntactically invalid and Docker refuses to build it.

This is like writing `import` in Python with no module name — the interpreter rejects it immediately.

---

**Scenario B — `FROM scratch` — The intentional empty base image**

Now here is the more interesting and interview-worthy answer. Docker has a special reserved keyword called **`scratch`** — and it represents a completely empty base image with absolutely nothing in it.

```dockerfile
FROM scratch
```

This is valid, intentional, and used by advanced Docker users for very specific purposes.

---

**What is `FROM scratch`?**

`scratch` is not an image you download. It is a virtual, empty starting point built into Docker itself. When you use `FROM scratch`:

- There is no operating system
- There is no shell (`/bin/sh`, `/bin/bash` — none of these exist)
- There are no utilities (`ls`, `cd`, `curl` — nothing)
- There are no libraries
- The image is literally empty — zero bytes as a base

You are building an image from absolute zero.

---

**Who uses `FROM scratch` and why?**

**Use Case 1 — Go applications (most common)**

Go compiles into a single, fully self-contained binary that does not need any external libraries or OS to run. This makes it perfect for `FROM scratch`.

```dockerfile
# Stage 1 — Build the Go binary
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN go build -o main .

# Stage 2 — Copy only the binary into a scratch image
FROM scratch

COPY --from=builder /app/main /main

ENTRYPOINT ["/main"]
```

The final image contains:
- Just the Go binary — nothing else
- No OS, no shell, no libraries
- Image size: **a few MB** instead of hundreds of MB

**Why this matters:** The smaller the image, the faster it pulls from the registry, the smaller the attack surface, and the less storage it uses in your container registry.

---

**Use Case 2 — Building base images for other teams**

Docker's own official base images like `alpine`, `debian`, and `ubuntu` are themselves built `FROM scratch`. The image maintainers build the minimal OS filesystem and package it as a layer on top of `scratch`.

---

**Use Case 3 — Maximum security environments**

In very high-security environments, teams use `FROM scratch` so the container has the absolute minimum attack surface. There is no shell an attacker can drop into, no OS tools they can use, no package manager they can exploit — just the one binary the application needs.

---

**The practical limitation of `FROM scratch`:**

Because there is no shell and no OS utilities, you cannot use `RUN` commands in the final stage — `RUN` needs a shell to execute commands.

```dockerfile
FROM scratch
RUN apt-get install curl    # ERROR — no shell, no package manager, nothing
```

This fails immediately. With `FROM scratch`, the only instruction that meaningfully works in the final stage is `COPY` — to copy pre-built binaries or files from a previous stage.

This is why `FROM scratch` almost always goes hand in hand with **multi-stage builds** — you do all the heavy lifting (installing, compiling, building) in an earlier stage, and then copy just the final output into the scratch image.

---

**`FROM scratch` vs `FROM alpine` — size comparison:**

| Base Image | Size | Has Shell? | Has Package Manager? | Use Case |
|---|---|---|---|---|
| `FROM scratch` | 0 bytes | ❌ No | ❌ No | Go binaries, maximum security |
| `FROM alpine` | ~5 MB | ✅ Yes (ash) | ✅ Yes (apk) | Most microservices |
| `FROM debian:slim` | ~75 MB | ✅ Yes | ✅ Yes (apt) | When alpine compatibility issues arise |
| `FROM ubuntu` | ~77 MB | ✅ Yes | ✅ Yes (apt) | Full OS needed |

In AzureShop we used `FROM node:18-alpine` for our Node.js services — `alpine` gave us a minimal OS with a shell and package manager while keeping the image small. `scratch` was not suitable for Node.js because Node.js needs runtime libraries that do not exist in a scratch image.

---

**How to debug a `FROM scratch` container:**

Because there is no shell, you cannot `docker exec -it container /bin/sh` into a scratch-based container to debug it. This is the main operational challenge.

Solutions:
- Use `FROM gcr.io/distroless/static` instead — Google's distroless images have no shell but include minimal C libraries and are debuggable in limited ways
- Add debugging tools only in a separate debug build
- Rely entirely on application logs exported to a logging system

```dockerfile
# distroless — a middle ground between scratch and alpine
FROM gcr.io/distroless/nodejs18-debian11

COPY --from=builder /app /app
CMD ["/app/server.js"]
```

Distroless images are popular in production because they have no shell (secure) but include the runtime libraries the application needs (functional).

---

**Summary (what to say if time is short):**

*"If you write `FROM` with nothing after it, Docker throws a syntax error immediately — `FROM` requires at least one argument. However, there is a special valid case called `FROM scratch` — scratch is a built-in Docker keyword that represents a completely empty base image with no OS, no shell, no libraries, and no utilities — literally zero bytes as a base. It is used mainly for compiled Go applications that produce a single self-contained binary, because the binary can run without any OS. The final image is just a few MB instead of hundreds. The limitation is that you cannot use `RUN` commands in a scratch-based image because there is no shell. So `FROM scratch` almost always pairs with multi-stage builds — you compile in an earlier stage and just copy the binary into the scratch final stage. In AzureShop we used `FROM node:18-alpine` for our microservices because Node.js needs a runtime, but `FROM scratch` is a powerful technique to know for Go services or maximum-security environments."*

---

#### Q3. Will Docker throw an error during runtime or build time?

**Answer:**

This question is directly connected to Q2 — but it also tests a broader and very important concept: **knowing when Docker catches an error — before the image is built, or after the container starts running.**

Let me answer both — the specific context from Q2, and the broader concept.

---

**Direct answer for the `FROM` error from Q2:**

If you write `FROM` with no argument — the error is thrown at **build time** — the moment you run `docker build`.

```bash
docker build -t my-app .
```

```
Error response from daemon: dockerfile parse error on line 1:
FROM requires either one or three arguments
```

Docker does not even start building. It reads the Dockerfile, hits the invalid `FROM` instruction on line 1, and stops immediately. No image is created, no container runs. The error is caught as early as possible.

---

**Now the broader concept — Build Time vs Runtime errors:**

Understanding which errors happen at which stage is critical. It tells you:
- When you will discover the problem
- How quickly you can fix it
- How much damage it can cause

---

**BUILD TIME — Errors that happen during `docker build`**

Build time is when Docker reads your Dockerfile and executes each instruction to create the image. Any failure at this stage means **no image is produced**.

Build time errors are the **safest type** — they are caught before any container ever runs, before anything reaches production.

**Common build time errors and what causes them:**

**1. Invalid `FROM` instruction:**
```dockerfile
FROM
# or
FROM imagethatdoesnotexist:99.99
```
```
Error: pull access denied for imagethatdoesnotexist
```
Docker cannot find the base image on Docker Hub or your private registry.

---

**2. `COPY` file not found:**
```dockerfile
COPY app.js .
```
```
COPY failed: file not found in build context: app.js
```
The file `app.js` does not exist in the directory where you ran `docker build`. Docker cannot copy something that is not there.

---

**3. `RUN` command fails:**
```dockerfile
RUN npm install
```
```
npm ERR! Cannot find module 'package.json'
```
The `npm install` command failed because `package.json` was not copied before this `RUN` instruction. Any `RUN` command that exits with a non-zero exit code fails the build.

---

**4. Wrong instruction syntax:**
```dockerfile
COPY src/ dst/    # Fine
EXPOSE port       # Error — port must be a number, not the word "port"
```
Docker validates instruction arguments and fails if they are wrong.

---

**RUNTIME — Errors that happen during `docker run`**

Runtime is when the container starts and the application inside it runs. The image was built successfully — but something goes wrong when the container actually starts or while it is running.

Runtime errors are **more dangerous** — the image was built, pushed to the registry, possibly deployed to production — and only then does the problem show up.

**Common runtime errors and what causes them:**

**1. Wrong or missing `CMD` / `ENTRYPOINT`:**
```dockerfile
CMD ["node", "server.js"]
```
The image builds fine. But when the container starts:
```
Error: Cannot find module '/app/server.js'
```
The file `server.js` does not exist at the path the `CMD` expects. Docker built the image successfully — but the container exits immediately when it tries to run.

This is a very common mistake — the image builds, the pipeline goes green, but the container crashes in staging or production.

---

**2. Missing environment variable:**
```javascript
// app code
const dbUrl = process.env.DATABASE_URL;
if (!dbUrl) throw new Error("DATABASE_URL is required");
```
The image builds perfectly. But at runtime, if `DATABASE_URL` is not passed to the container:
```bash
docker run my-app    # No -e DATABASE_URL passed
```
The container starts and immediately crashes:
```
Error: DATABASE_URL is required
```

---

**3. Port conflict:**
```bash
docker run -p 3000:3000 my-app
```
```
Error: Bind for 0.0.0.0:3000 failed: port is already allocated
```
Port 3000 is already in use on the host machine. The image is fine — the problem is the environment it is running in.

---

**4. Out of memory:**

The container starts and runs — but the application consumes more memory than the container limit allows. Docker kills the container with an OOM (Out Of Memory) error:
```
Container killed — OOM (Out Of Memory)
```
This is a pure runtime error — nothing in the Dockerfile or build could have predicted this.

---

**5. Application crashes after startup:**

The container starts successfully, the health check passes — but a few minutes later the application crashes due to a bug, a failed database connection, or an unhandled exception. This is entirely a runtime error that no amount of Dockerfile inspection would catch.

---

**Side by side — Build Time vs Runtime:**

| | Build Time | Runtime |
|---|---|---|
| **When it happens** | During `docker build` | During `docker run` or while container is running |
| **What causes it** | Invalid Dockerfile syntax, missing files, failed RUN commands | Wrong CMD, missing env vars, port conflicts, app bugs, resource limits |
| **Is an image produced?** | ❌ No — build fails, nothing is created | ✅ Yes — image exists, but container fails |
| **How dangerous?** | Safe — caught early, nothing deployed | Risky — can reach production before being caught |
| **How to fix** | Fix the Dockerfile and rebuild | Fix the application config, env vars, or resources |
| **Detected in pipeline?** | ✅ Yes — `docker build` step fails | Sometimes — depends on whether integration tests run |

---

**Why this matters in a real CI/CD pipeline:**

In our AzureShop pipeline, we structured the stages specifically to catch both types as early as possible:

```
Stage 1 — docker build
  → Catches ALL build time errors immediately
  → If this fails, pipeline stops — nothing is pushed

Stage 2 — docker run (smoke test)
  → Starts the container and hits the /health endpoint
  → Catches runtime startup errors before the image is pushed to ACR

Stage 3 — docker push to ACR
  → Only runs if stages 1 and 2 both passed
  → Only a fully working image reaches the registry

Stage 4 — Deploy to AKS dev
  → Further runtime issues caught here with real integration tests
```

The key principle: **catch errors as early as possible in the pipeline.** A build time error caught in Stage 1 costs 30 seconds to fix. A runtime error discovered in production costs hours of incident response.

---

**A real example from AzureShop:**

One of our developers updated the `user-service` Dockerfile but forgot to copy `package.json` before running `npm install`:

```dockerfile
FROM node:18-alpine
WORKDIR /app
# COPY package.json .   ← forgot this line
RUN npm install          ← build time error — no package.json found
COPY . .
CMD ["node", "src/server.js"]
```

The pipeline failed at the `docker build` stage immediately:
```
npm warn saveError ENOENT: no such file or directory, open '/app/package.json'
npm notice created a lockfile as package-lock.json
npm warn enoent ENOENT: no such file or directory, open '/app/package.json'
```

**Build time error — caught in the pipeline, never reached staging.** The developer fixed it in 2 minutes by adding the missing `COPY package.json .` line before `RUN npm install`.

In contrast, if the error had been in the `CMD` pointing to the wrong file — the build would have passed, the image would have been pushed to ACR, deployed to AKS dev, and the pod would have crashed there. A runtime error discovered 10 minutes later instead of 30 seconds.

---

**Summary (what to say if time is short):**

*"For the `FROM` with no argument error from Q2 — that is a build time error. Docker reads the Dockerfile, hits the invalid instruction immediately, and stops — no image is produced. More broadly, build time errors happen during `docker build` — things like invalid instructions, missing files in COPY, or failed RUN commands. These are the safest errors because they are caught before anything is deployed. Runtime errors happen when the container actually starts — things like a wrong CMD pointing to a missing file, a missing environment variable the app needs, port conflicts, or the app crashing due to a bug. Runtime errors are more dangerous because the image builds successfully, the pipeline may go green, and the error only shows up in staging or production. In AzureShop, we added a smoke test stage after `docker build` that started the container and hit the health endpoint — this caught most runtime startup errors before the image was pushed to ACR."*

---

#### Q4. Your Docker image size is 1.5 GB. How will you reduce the image size?

**Answer:**

A 1.5 GB Docker image is a very common problem in real projects — especially when developers write Dockerfiles quickly without thinking about size. Large images cause slow pipeline builds, slow deployments, high registry storage costs, and a bigger security attack surface.

I have faced this exact situation in AzureShop. Let me walk through every technique I use to reduce image size — in the order I apply them.

---

**Why does image size matter?**

Before the techniques — let me quickly explain why 1.5 GB is a problem:

- **Slow CI/CD pipelines** — every build pushes a 1.5 GB image to the registry. Every deployment pulls it. That is minutes wasted on every pipeline run.
- **High registry costs** — Azure Container Registry charges for storage. 1.5 GB per image × multiple services × multiple tags = significant cost.
- **Slow pod startup** — when Kubernetes schedules a pod on a new node, it pulls the image first. A 1.5 GB image takes much longer to pull than a 150 MB image.
- **Larger attack surface** — every package and tool in the image is a potential vulnerability. Less in the image means less to exploit.

---

**Technique 1 — Use a Smaller Base Image**

This is usually the single biggest win. The base image you choose sets the floor for your image size.

**Common base image sizes:**

| Base Image | Size | When to use |
|---|---|---|
| `ubuntu:22.04` | ~77 MB | Full OS needed — rarely in production |
| `debian:bookworm` | ~117 MB | Full Debian — rarely needed |
| `debian:bookworm-slim` | ~75 MB | Reduced Debian |
| `node:18` | ~1.1 GB | Never use this — bloated with build tools |
| `node:18-slim` | ~240 MB | Slim Debian base — better |
| `node:18-alpine` | ~51 MB | Alpine base — much better |
| `python:3.11` | ~1.0 GB | Never in production |
| `python:3.11-slim` | ~130 MB | Slim Debian |
| `python:3.11-alpine` | ~50 MB | Best for production |
| `alpine:3.18` | ~7 MB | Minimal OS — use as base for custom images |
| `gcr.io/distroless/nodejs18` | ~115 MB | No shell, secure, Google maintained |
| `scratch` | 0 bytes | Go binaries only |

**Before:**
```dockerfile
FROM node:18        # 1.1 GB base image — bloated with build tools, docs, compilers
```

**After:**
```dockerfile
FROM node:18-alpine  # 51 MB base image — same Node.js, minimal OS
```

Just changing the base image can reduce size by 500-800 MB immediately.

**AzureShop example:**

Our `user-service` started with `FROM node:18` — the image was 1.4 GB. Switching to `FROM node:18-alpine` dropped it to 320 MB before we even did anything else.

---

**Technique 2 — Multi-Stage Builds (Biggest Impact)**

This is the most powerful technique. The idea is simple — use one stage to build your application (with all the build tools, compilers, dev dependencies) and then copy only the final output into a clean, minimal production image. The build tools never make it into the final image.

**Before (single stage — everything in one image):**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install          # Installs ALL dependencies — dev + production
COPY . .
RUN npm run build        # Compiles TypeScript to JavaScript

CMD ["node", "dist/server.js"]
# Final image includes: Node.js + npm + all dev packages + TypeScript compiler + source code
# Size: ~800 MB
```

**After (multi-stage build):**
```dockerfile
# Stage 1 — Builder (only used during build, thrown away after)
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci                    # Install ALL dependencies including dev
COPY . .
RUN npm run build             # Compile TypeScript → JavaScript output in /app/dist

# Stage 2 — Production (the actual final image)
FROM node:18-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production  # Install ONLY production dependencies — no dev tools
COPY --from=builder /app/dist ./dist   # Copy only compiled output from builder stage

CMD ["node", "dist/server.js"]
# Final image includes: Node.js + production packages + compiled JS only
# Size: ~180 MB
```

The builder stage (with TypeScript, dev dependencies, source code) is completely discarded. Only the compiled output and production dependencies go into the final image.

**AzureShop result:** Multi-stage builds reduced our `order-service` image from 1.5 GB to 210 MB.

---

**Technique 3 — Chain `RUN` Commands to Reduce Layers**

Every `RUN` instruction creates a new layer. Even if you delete files in a later `RUN`, the files still exist in the earlier layer — they are just hidden. The total image size includes all layers.

**Before — 3 separate layers, deleted files still counted:**
```dockerfile
RUN apt-get update                    # Layer 1 — cache files created (~30 MB)
RUN apt-get install -y curl           # Layer 2 — curl installed
RUN rm -rf /var/lib/apt/lists/*       # Layer 3 — cache deleted BUT layer 1 still has it
```

The apt cache from Layer 1 still exists in the image even though you deleted it in Layer 3. Total size = sum of all 3 layers.

**After — single layer, clean:**
```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

Everything happens in one layer. The cache is deleted before the layer is committed — it never persists in the image.

---

**Technique 4 — Use `.dockerignore` File**

When you run `docker build`, Docker sends everything in your current directory to the Docker daemon as the **build context**. Without a `.dockerignore`, this includes `node_modules`, `.git`, test files, logs — everything. These get into the image via `COPY . .`.

Create a `.dockerignore` file in the same directory as your Dockerfile:

```
# .dockerignore
node_modules/
.git/
.gitignore
*.log
*.md
tests/
coverage/
.env
.env.*
dist/
.DS_Store
Dockerfile
.dockerignore
```

**Why this matters:**
- `node_modules` alone can be 200-500 MB — you do not want this copied into the image (you install fresh inside anyway)
- `.git` folder adds significant size with no benefit in the image
- Smaller build context = faster `docker build`

---

**Technique 5 — Install Only Production Dependencies**

In Node.js projects, `npm install` installs both production and dev dependencies (testing frameworks, TypeScript compiler, linters). These dev tools are not needed at runtime.

**Before:**
```dockerfile
RUN npm install    # Installs everything — dev + production
```

**After:**
```dockerfile
RUN npm ci --only=production    # Production dependencies only
```

Or for Python:
```dockerfile
# Before
RUN pip install -r requirements.txt    # All dependencies

# After — split requirements files
RUN pip install --no-cache-dir -r requirements-prod.txt    # Production only
```

`--no-cache-dir` in pip prevents caching downloaded packages — keeping the layer lean.

---

**Technique 6 — Remove Package Manager Cache After Install**

Package managers leave behind caches after installation. Always clean them in the same `RUN` instruction:

**For Alpine (apk):**
```dockerfile
RUN apk add --no-cache curl git
# --no-cache flag means no cache is stored at all — cleanest approach
```

**For Debian/Ubuntu (apt):**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl git && \
    rm -rf /var/lib/apt/lists/*
# --no-install-recommends skips optional packages
# rm -rf /var/lib/apt/lists/* removes the package list cache
```

**For Python (pip):**
```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
# --no-cache-dir prevents pip from storing downloaded wheel files
```

---

**Technique 7 — Copy Only What You Need**

Instead of blindly copying everything:

```dockerfile
# Bad — copies everything including tests, docs, config files
COPY . .
```

Copy only what the application actually needs to run:

```dockerfile
# Good — copy only the specific files needed
COPY package*.json ./
COPY src/ ./src/
COPY config/ ./config/
```

---

**Technique 8 — Use `docker image ls` and `docker history` to Find What Is Bloating the Image**

Before optimising, diagnose the problem first. Find out which layers are large:

```bash
# See current image size
docker image ls my-app

# See size contribution of each layer
docker history my-app
```

Output of `docker history`:
```
IMAGE         CREATED      CREATED BY                                      SIZE
a1b2c3d4e5f6  2 hours ago  CMD ["node" "src/server.js"]                    0B
<missing>     2 hours ago  COPY . .                                         850MB   ← problem here
<missing>     2 hours ago  RUN npm install                                  420MB   ← and here
<missing>     2 hours ago  WORKDIR /app                                     0B
<missing>     2 hours ago  /bin/sh -c #(nop) FROM node:18                  1.1GB   ← and here
```

This immediately shows you which layers to target. In this example — the base image and `npm install` are the biggest culprits.

You can also use **Dive** — an open-source tool that visually shows what is in each layer:

```bash
brew install dive
dive my-app:latest
```

Dive shows which files each layer adds, so you can identify exactly what is eating space.

---

**Putting it all together — before and after in AzureShop:**

**Before (1.5 GB):**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "src/server.js"]
```

Problems:
- `FROM node:18` — 1.1 GB base with compilers and tools not needed at runtime
- `COPY . .` — copied `node_modules`, `.git`, test files, everything
- `npm install` — installed dev dependencies too
- No multi-stage build — build tools stay in the final image

**After (85 MB):**
```dockerfile
# Stage 1 — Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build

# Stage 2 — Production
FROM node:18-alpine AS production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder /app/dist ./dist

ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

USER appuser
CMD ["node", "dist/server.js"]
```

**Result: 1.5 GB → 85 MB — a 94% reduction.**

---

**Summary of all techniques:**

| Technique | Typical Size Saving | Effort |
|---|---|---|
| Smaller base image (alpine instead of full) | 500 MB – 1 GB | Low |
| Multi-stage build | 300 MB – 800 MB | Medium |
| Chain RUN commands | 30 MB – 100 MB | Low |
| `.dockerignore` | 100 MB – 500 MB | Very Low |
| Production dependencies only | 50 MB – 200 MB | Low |
| Remove package manager cache | 20 MB – 50 MB | Very Low |
| Copy only needed files | 10 MB – 100 MB | Low |

---

**Summary (what to say if time is short):**

*"A 1.5 GB image is a very common problem. I fix it using several techniques in order of impact. First — switch to a smaller base image like `node:18-alpine` instead of `node:18` — this alone can remove 500-800 MB. Second — use multi-stage builds — compile and build in one stage, then copy only the final output into a clean minimal production image, leaving all build tools behind. Third — add a `.dockerignore` file to prevent `node_modules`, `.git`, and test files from being copied into the image. Fourth — install only production dependencies, not dev packages. Fifth — chain all `RUN` commands with `&&` and clean package manager caches in the same layer so deleted files do not persist in earlier layers. In AzureShop, applying all these techniques reduced our user-service image from 1.5 GB to 85 MB — a 94% reduction — which significantly sped up our pipeline and reduced ACR storage costs."*

---

#### Q5. What is Multi-stage Build? Have you implemented Multi-stage Build in your project? Why do we use Multi-stage Build?

**Answer:**

**Yes, I have implemented multi-stage builds in the AzureShop project — for all four microservices. It was one of the most impactful changes we made to our Docker setup.**

Let me explain what it is, why we use it, and walk through the exact implementation we did in AzureShop.

---

**What is a Multi-stage Build?**

A multi-stage build is a Dockerfile technique where you divide the image building process into multiple separate stages — each with its own `FROM` instruction and its own purpose. Each stage is isolated from the others, and you can selectively copy files from one stage into another.

The key idea is:

> **Build your application in one stage. Run your application in another stage. Only the running stage goes into the final image.**

Everything used to build the application — compilers, build tools, test frameworks, source code, dev dependencies — stays in the build stage and is completely thrown away. It never makes it into the final image.

---

**Simple analogy:**

Think of it like manufacturing a car:

- The **factory floor** (build stage) has welding machines, paint booths, assembly robots, tools, spare parts, packaging materials — everything needed to build the car
- The **showroom** (production stage) has only the finished car — clean, minimal, ready to use

When you buy a car from the showroom, you do not get the factory floor too. You get just the car. Multi-stage builds work exactly the same way.

---

**How it works — the syntax:**

You write multiple `FROM` instructions in a single Dockerfile. Each `FROM` starts a new stage. You give each stage a name using `AS`.

```dockerfile
# Stage 1 — named "builder"
FROM node:18-alpine AS builder
# ... build the application ...

# Stage 2 — named "production" — this is the final image
FROM node:18-alpine AS production
# ... copy only what is needed from builder ...
```

The magic line is `COPY --from=builder`:

```dockerfile
COPY --from=builder /app/dist ./dist
```

This copies files from the `builder` stage into the `production` stage. Everything else in the `builder` stage is discarded — it never becomes part of the final image.

When you run `docker build`, Docker builds all stages but only the **last stage** becomes the final image that gets pushed to the registry.

---

**Why do we use Multi-stage Builds?**

There are four strong reasons:

---

**Reason 1 — Dramatically smaller image size**

This is the primary reason. Build tools, compilers, and dev dependencies are needed to build the application — but not to run it.

Without multi-stage:
```
Final image = base OS + build tools + compilers + dev dependencies + source code + compiled output
           = 1.5 GB
```

With multi-stage:
```
Final image = base OS + production dependencies + compiled output only
           = 85 MB
```

Everything the user needs is there. Everything the build needed but the user does not is gone.

---

**Reason 2 — Better Security**

A smaller image means a smaller attack surface. Every tool, library, and file in the image is a potential vulnerability.

Think about this — if your production image contains:
- A compiler — an attacker who breaks in can compile malicious code inside the container
- A shell and package manager — an attacker can install tools and escalate their attack
- Your full source code — an attacker can read your business logic, find API keys, understand your system

With multi-stage builds, the production image has:
- Only the compiled application output
- Only production runtime dependencies
- No compilers, no build tools, no source code

If an attacker somehow compromises the container, they have almost nothing to work with.

---

**Reason 3 — Faster CI/CD Pipelines**

A smaller image = less time to push to ACR and less time to pull from ACR.

In AzureShop:
- Before multi-stage: pushing a 1.5 GB image to ACR took ~4 minutes per service
- After multi-stage: pushing an 85 MB image took ~20 seconds

Across 4 microservices, every pipeline run saved over 14 minutes. That adds up significantly over hundreds of pipeline runs.

Similarly, when Kubernetes schedules a pod on a new node, it pulls the image first. A smaller image means pods start faster — which matters critically during auto-scaling events or incident recovery.

---

**Reason 4 — Clean Separation of Concerns**

Multi-stage builds force you to think clearly about what belongs in the build environment and what belongs in the production environment. This discipline leads to better, cleaner Dockerfiles.

---

**Real Implementation from AzureShop — Node.js user-service:**

This is the exact multi-stage Dockerfile I wrote for the `user-service` in AzureShop:

```dockerfile
# ─────────────────────────────────────────
# Stage 1 — Builder
# Purpose: Install all dependencies and compile TypeScript to JavaScript
# ─────────────────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files first — Docker layer cache optimisation
# If package.json hasn't changed, Docker reuses the cached npm ci layer
COPY package*.json ./

# Install ALL dependencies — including dev (TypeScript compiler needed here)
RUN npm ci

# Copy source code
COPY src/ ./src/
COPY tsconfig.json ./

# Compile TypeScript to JavaScript
RUN npm run build
# Output goes to /app/dist


# ─────────────────────────────────────────
# Stage 2 — Production
# Purpose: Run the compiled application — minimal, secure, lean
# ─────────────────────────────────────────
FROM node:18-alpine AS production

# Security — create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install ONLY production dependencies — no TypeScript, no dev tools
RUN npm ci --only=production && npm cache clean --force

# Copy ONLY the compiled JavaScript output from builder stage
# TypeScript source, tsconfig, node_modules with dev packages — all left behind
COPY --from=builder /app/dist ./dist

# Environment configuration
ENV NODE_ENV=production
ENV PORT=3000

# Document the port
EXPOSE 3000

# Health check for Kubernetes liveness and readiness probes
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Switch to non-root user
USER appuser

# Start the application
CMD ["node", "dist/server.js"]
```

**What each stage contains:**

| | Builder Stage | Production Stage |
|---|---|---|
| Base image | `node:18-alpine` | `node:18-alpine` |
| TypeScript compiler | ✅ Yes (dev dependency) | ❌ No |
| All dev dependencies | ✅ Yes | ❌ No |
| Source `.ts` files | ✅ Yes | ❌ No |
| `tsconfig.json` | ✅ Yes | ❌ No |
| Compiled `/dist` output | ✅ Yes (created here) | ✅ Yes (copied from builder) |
| Production `node_modules` | ❌ No | ✅ Yes |
| Non-root user | ❌ Not needed | ✅ Yes |
| Health check | ❌ Not needed | ✅ Yes |
| **Becomes final image?** | ❌ Discarded | ✅ Yes |

---

**Real Implementation from AzureShop — Java order-service:**

Multi-stage builds are even more dramatic for Java because Maven downloads hundreds of MB of dependencies and the JDK (Java Development Kit) is much larger than the JRE (Java Runtime Environment):

```dockerfile
# Stage 1 — Build with Maven and full JDK
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Copy Maven wrapper and pom.xml — layer cache optimisation
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies — cached if pom.xml unchanged
RUN ./mvnw dependency:resolve

# Copy source and build the JAR
COPY src/ ./src/
RUN ./mvnw package -DskipTests

# Stage 2 — Run with minimal JRE only
FROM eclipse-temurin:17-jre-alpine AS production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only the final JAR — no Maven, no JDK, no source code
COPY --from=builder /app/target/order-service-*.jar app.jar

ENV JAVA_OPTS="-Xmx512m -Xms256m"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

USER appuser

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**Size impact for Java:**
- Single stage with JDK: ~850 MB
- Multi-stage with JRE only: ~195 MB

The builder stage has the full JDK (600 MB) + Maven + all source code. The production stage has only the JRE (180 MB) + the compiled JAR. Maven and the JDK are completely gone from the final image.

---

**Layer caching optimisation inside multi-stage builds:**

One important pattern to follow inside each stage — always copy `package.json` (or `pom.xml`) before copying source code, and run dependency installation before copying your application code:

```dockerfile
# Correct order — maximises Docker layer cache
COPY package*.json ./          # Layer A — changes rarely
RUN npm ci                     # Layer B — only rebuilds when package.json changes
COPY src/ ./src/               # Layer C — changes often (your code)
RUN npm run build              # Layer D — only runs when Layer C changes
```

If you change only your source code (which happens 90% of the time), Docker reuses the cached `npm ci` layer (Layer B) — it does not reinstall all dependencies. This makes builds significantly faster in the CI pipeline.

---

**Results in AzureShop after implementing multi-stage builds:**

| Service | Before | After | Reduction |
|---|---|---|---|
| user-service (Node.js) | 1.4 GB | 85 MB | 94% |
| order-service (Java) | 850 MB | 195 MB | 77% |
| payment-service (Python) | 980 MB | 120 MB | 88% |
| product-service (Node.js) | 1.3 GB | 90 MB | 93% |

Pipeline push time per service went from ~4 minutes to ~20 seconds. Pod startup time on new AKS nodes dropped from ~3 minutes to under 20 seconds.

---

**Summary (what to say if time is short):**

*"A multi-stage build is a Dockerfile technique where you use multiple `FROM` instructions to create separate stages — typically a builder stage and a production stage. The builder stage has all the tools needed to compile and build the application — compilers, dev dependencies, build frameworks. The production stage receives only the final compiled output using `COPY --from=builder` and contains none of the build tools. The final Docker image that gets pushed to the registry is only the production stage — the builder stage is completely discarded. We use multi-stage builds for four reasons — smaller image size, better security (no build tools or source code in production), faster CI/CD pipelines (less data to push and pull), and clean separation between build and runtime environments. In AzureShop, I implemented multi-stage builds for all four microservices — reducing the user-service from 1.4 GB to 85 MB and the Java order-service from 850 MB to 195 MB. Pipeline push times dropped from 4 minutes to 20 seconds per service."*

---

<!-- Add more questions as Q6, Q7... -->

---

### Scenario Based Questions

> These are real scenario questions asked by the interviewer to test how you think and handle real-world problems. The interviewer wants to see your thought process and step-by-step approach — not just the final answer.

---

#### Scenario 1. Application Container cannot connect to Database Container. Both containers are running on the same host. Getting Connection Refused. How will you troubleshoot?

**Answer:**

**Connection Refused** between two containers on the same host is one of the most common Docker problems. The key thing to understand upfront is — **just because two containers are running on the same host does NOT mean they can talk to each other automatically.** Docker networking determines whether containers can communicate, and there are several things that can go wrong.

Let me walk through my troubleshooting approach step by step — in the exact order I follow in real production situations.

---

**Step 1 — Confirm both containers are actually running**

Before anything else, verify both containers are up and healthy:

```bash
docker ps
```

Output:
```
CONTAINER ID   IMAGE         STATUS                   NAMES
a1b2c3d4e5f6   my-app        Up 5 minutes             app-container
b2c3d4e5f6a1   postgres:15   Up 5 minutes (healthy)   db-container
```

Things to check:
- Is the STATUS `Up`? If it shows `Exited` or `Restarting` — the container is not actually running
- Is there a health check? Does it show `healthy` or `unhealthy`?
- Are both containers there? Sometimes a container fails to start silently

If the database container is in a restart loop:
```bash
docker logs db-container
```

Read the logs — a wrong password, missing environment variable, or corrupt data volume will show here.

---

**Step 2 — Check if both containers are on the same Docker network**

This is the most common root cause. By default, when you run containers with `docker run` without specifying a network, Docker puts them on the **default bridge network**. On the default bridge network, containers cannot resolve each other by name — they can only communicate by IP address, which changes every restart.

Check what network each container is on:

```bash
docker inspect app-container --format '{{json .NetworkSettings.Networks}}' | jq
docker inspect db-container --format '{{json .NetworkSettings.Networks}}' | jq
```

Or inspect the network directly:

```bash
docker network ls
docker network inspect bridge
```

**The problem — containers on different networks:**

```
app-container  → network: bridge (default)
db-container   → network: my-custom-network
```

Two different networks — they cannot communicate at all. This gives `Connection Refused` or `Name resolution failure`.

**The fix — put both on the same custom network:**

```bash
# Create a custom network
docker network create azureshop-network

# Run containers on the same network
docker run -d --name db-container --network azureshop-network postgres:15
docker run -d --name app-container --network azureshop-network my-app
```

On a **custom bridge network**, containers can reach each other by **container name** — no IP addresses needed:

```
# From inside app-container, you can reach the DB like this:
postgres://db-container:5432/mydb
```

Docker's built-in DNS resolves `db-container` to the correct IP automatically. This is the correct and recommended approach.

---

**Step 3 — Test connectivity from inside the application container**

Do not just assume the network is the problem — test it directly from inside the container. This tells you exactly where the connection is failing.

```bash
# Get a shell inside the application container
docker exec -it app-container sh
```

Now from inside the container, test the connection:

```bash
# Test if the DB port is reachable
nc -zv db-container 5432
# or
telnet db-container 5432
# or
curl -v telnet://db-container:5432
```

**Possible outcomes:**

**Outcome A — `Name or service not known`:**
```
nc: getaddrinfo for host "db-container" port 5432: Name or service not known
```
DNS resolution is failing. The containers are not on the same custom network — they cannot resolve each other by name. Go back to Step 2.

**Outcome B — `Connection refused`:**
```
nc: connect to db-container port 5432 (tcp) failed: Connection refused
```
DNS resolved — the name is known — but nothing is listening on port 5432 inside the DB container. The database process may not be running or is listening on a different port. Go to Step 4.

**Outcome C — `Connection timed out`:**
```
nc: connect to db-container port 5432 (tcp) failed: Connection timed out
```
The network exists but there is a firewall rule or network policy blocking the connection.

**Outcome D — Connected successfully:**
```
Connection to db-container 5432 port [tcp/postgresql] succeeded!
```
The network is fine — the problem is at the application level (wrong credentials, wrong database name, wrong URL format). Go to Step 5.

---

**Step 4 — Check if the database is actually listening on the expected port**

Even if the DB container is running, the database process inside it might not be ready yet, or might be listening on the wrong interface.

Check from inside the DB container:

```bash
docker exec -it db-container sh

# Check what is listening on which ports
netstat -tlnp
# or
ss -tlnp
```

Output:
```
Proto  Local Address         State    PID/Program name
tcp    0.0.0.0:5432          LISTEN   1/postgres
```

**Good** — `0.0.0.0:5432` means PostgreSQL is listening on all interfaces — reachable from other containers.

**Bad** — `127.0.0.1:5432` means PostgreSQL is listening only on localhost — only the local process inside the container can connect. This is a database configuration issue — `bind-address` or `listen_addresses` is set to `127.0.0.1` instead of `0.0.0.0`.

**For PostgreSQL — fix the bind address:**

Check `postgresql.conf` inside the container:
```bash
docker exec -it db-container psql -U postgres -c "SHOW listen_addresses;"
```

If it shows `localhost`, update `postgresql.conf`:
```
listen_addresses = '*'
```

---

**Step 5 — Check the application's database connection string**

Even if networking is perfect, the application might be using the wrong connection details. This is a very common mistake.

Check the environment variables passed to the application container:

```bash
docker inspect app-container --format '{{json .Config.Env}}' | jq
```

Output:
```json
[
  "DATABASE_URL=postgres://admin:password@localhost:5432/appdb",
  "NODE_ENV=production"
]
```

**Spot the problem:** `DATABASE_URL` is pointing to `localhost` — but inside a container, `localhost` means the container itself, not the database container. The database is not running inside the app container.

**The fix — use the container name as the hostname:**

```bash
docker run -d \
  --name app-container \
  --network azureshop-network \
  -e DATABASE_URL=postgres://admin:password@db-container:5432/appdb \
  my-app
```

`db-container` (the container name) is the correct hostname to use, not `localhost`.

---

**Step 6 — Check database credentials and database name**

Even with correct networking and hostname, authentication can fail. Check the DB container's environment variables:

```bash
docker inspect db-container --format '{{json .Config.Env}}' | jq
```

Output:
```json
[
  "POSTGRES_USER=admin",
  "POSTGRES_PASSWORD=secret123",
  "POSTGRES_DB=appdb"
]
```

Now check the app's connection string and confirm:
- Username matches: `admin` ✅
- Password matches: `secret123` ✅
- Database name matches: `appdb` ✅
- Port is correct: `5432` ✅

A mismatch in any of these gives an authentication error — which sometimes also manifests as `Connection Refused` depending on the database.

Test directly from inside the app container:

```bash
docker exec -it app-container sh

# Test actual DB connection with psql
psql postgres://admin:secret123@db-container:5432/appdb -c "SELECT 1;"
```

If this works, the issue is in how the application is reading the connection string.

---

**Step 7 — Check container logs for both containers**

Always check logs on both sides — the application and the database:

```bash
# App container logs — look for the exact error message
docker logs app-container --tail 50

# Database container logs — look for connection attempts and rejections
docker logs db-container --tail 50
```

**Common messages in DB logs that reveal the problem:**

```
# Wrong password
FATAL: password authentication failed for user "admin"

# Database does not exist
FATAL: database "appdb" does not exist

# Connection from app rejected by pg_hba.conf
FATAL: no pg_hba.conf entry for host "172.17.0.3", user "admin", database "appdb"

# DB not ready yet
PostgreSQL init process complete; ready to start
```

The last one is very common — the application starts and tries to connect before the database is fully initialised. The fix is to add a health check and a startup dependency.

---

**Step 8 — Verify using Docker Compose (the right approach)**

If you are running both containers manually with `docker run`, you are doing it the hard way. Docker Compose handles networking automatically — all services in a `docker-compose.yml` are automatically placed on the same custom network and can reach each other by service name.

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: my-app
    environment:
      DATABASE_URL: postgres://admin:secret123@db:5432/appdb
    depends_on:
      db:
        condition: service_healthy    # Wait for DB to be healthy before starting app
    networks:
      - azureshop-network

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: appdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - azureshop-network

networks:
  azureshop-network:
    driver: bridge
```

With this setup:
- Both containers are on `azureshop-network` — they can reach each other by service name
- The app uses `db` (the service name) as the hostname — not `localhost`
- `depends_on` with `condition: service_healthy` ensures the DB is fully ready before the app starts

```bash
docker-compose up -d
```

This eliminates almost all the common networking and startup-order problems.

---

**Complete troubleshooting flow:**

```
Connection Refused between App and DB
            |
            ↓
Step 1: Both containers running? (docker ps)
  → No: docker logs <container> to find why
  → Yes: continue
            |
            ↓
Step 2: Same Docker network? (docker network inspect)
  → No: create custom network, reconnect both containers
  → Yes: continue
            |
            ↓
Step 3: Test from inside app container (nc -zv db-container 5432)
  → Name not known: DNS issue → Step 2
  → Connection refused: DB not listening → Step 4
  → Timeout: firewall/network policy blocking
  → Connected: networking is fine → Step 5
            |
            ↓
Step 4: DB listening on 0.0.0.0 or 127.0.0.1? (netstat inside DB container)
  → 127.0.0.1: fix listen_addresses in DB config
            |
            ↓
Step 5: App using correct hostname? (not localhost — use container name)
  → localhost found: update DATABASE_URL to use container name
            |
            ↓
Step 6: Credentials correct? (user, password, DB name, port)
  → Mismatch: fix env vars
            |
            ↓
Step 7: Check logs on both sides (docker logs)
  → DB not ready: add healthcheck + depends_on
            |
            ↓
Step 8: Use Docker Compose — handles networking automatically
```

---

**AzureShop Real Example:**

During local development of AzureShop, a developer ran the `user-service` and `postgres` containers manually:

```bash
docker run -d --name postgres -e POSTGRES_PASSWORD=secret postgres:15
docker run -d --name user-service -e DATABASE_URL=postgres://postgres:secret@localhost:5432/users my-app
```

The `user-service` immediately threw `Connection Refused`.

**Root cause found in two steps:**

1. `docker inspect` showed both containers were on the default `bridge` network — but the app was using `localhost` as the hostname. Inside the container, `localhost` is the container itself — not the Postgres container.

2. Fix — create a custom network and use the container name:

```bash
docker network create azureshop-dev
docker run -d --name postgres --network azureshop-dev -e POSTGRES_PASSWORD=secret postgres:15
docker run -d --name user-service --network azureshop-dev \
  -e DATABASE_URL=postgres://postgres:secret@postgres:5432/users \
  my-app
```

Connection worked immediately. After this, we moved everything to Docker Compose so developers never had to manage networks manually again.

---

**Summary (what to say if time is short):**

*"Connection Refused between two containers on the same host is almost always a Docker networking issue. My troubleshooting approach has 8 steps. First, confirm both containers are actually running with `docker ps`. Second — and this is the most common root cause — check if both containers are on the same Docker network using `docker network inspect`. By default, containers cannot reach each other by name on the default bridge network. The fix is to create a custom network and run both containers on it. Third, I test connectivity from inside the app container using `nc -zv db-container 5432` to get the exact error — name resolution failure means network issue, connection refused means the DB is not listening. Fourth, I check if the database is listening on `0.0.0.0` and not just `127.0.0.1`. Fifth, I verify the app is using the container name as the hostname — not `localhost` — because inside a container, localhost means the container itself. Finally, I check credentials and read logs on both sides. The long-term fix is always to use Docker Compose, which puts all services on the same network automatically and handles startup order with `depends_on` and health checks."*

---

#### Scenario 2. Interviewer removed all obvious causes — Containers are running. Network is fine. Configuration is correct. Still getting Connection Refused. What will you check now?

**Answer:**

This is where the interview gets really interesting. The interviewer has eliminated the obvious causes — containers running, network fine, configuration correct. Now they want to see if you can think deeper. This is the difference between a junior engineer and a senior engineer.

When all the obvious things check out but the connection still fails, the problem is usually one of these hidden causes. Let me go through each one.

---

**Hidden Cause 1 — Database Container is Running but the Database Process is NOT Ready Yet**

This is the single most common hidden cause that trips everyone up.

The container status shows `Up` — but `Up` means the **container** is running, not that the **database process inside it** is fully initialised and ready to accept connections. PostgreSQL, MySQL, and MongoDB all go through an initialisation phase when they first start — creating data directories, running init scripts, setting up users.

During this init phase, the port may not be open yet — or it opens briefly and then closes while init scripts run. The application starts, tries to connect, and gets `Connection Refused` because the DB is not ready yet.

**How to verify:**

```bash
docker logs db-container --tail 30
```

If you see something like this, the DB is still initialising:

```
PostgreSQL init process in progress...
running /docker-entrypoint-initdb.d/init.sql
...
PostgreSQL init process complete; ready to start.
server started
```

The app tried to connect during `init process in progress` — before `server started`.

**The fix — proper health check on the DB container:**

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 15s    # Give DB 15 seconds to start before health checks begin

  app:
    image: my-app
    depends_on:
      db:
        condition: service_healthy    # Wait until DB health check passes
```

`pg_isready` checks if PostgreSQL is actually accepting connections — not just if the container is running. The app will not start until `pg_isready` returns success.

**Check health status manually:**

```bash
docker inspect db-container --format '{{.State.Health.Status}}'
# Returns: starting | healthy | unhealthy
```

If it shows `starting` — the DB is not ready. Wait and try again. If `unhealthy` — read the health check logs.

---

**Hidden Cause 2 — Database Has Reached Maximum Connections Limit**

The DB is running, network is fine, credentials are correct — but the database has hit its `max_connections` limit and is refusing new connections. This feels exactly like `Connection Refused` from the application's perspective.

**How to verify:**

```bash
# Connect to PostgreSQL and check current connections
docker exec -it db-container psql -U admin -d appdb -c "
SELECT count(*) AS current_connections,
       max_conn AS max_connections,
       max_conn - count(*) AS connections_available
FROM pg_stat_activity, (SELECT setting::int AS max_conn FROM pg_settings WHERE name = 'max_connections') AS mc
GROUP BY max_conn;"
```

Output:
```
current_connections | max_connections | connections_available
--------------------+-----------------+----------------------
                100 |             100 |                     0
```

Zero connections available — the DB is full. New connections are rejected.

**The fix:**

Option 1 — Increase `max_connections` in PostgreSQL config:
```bash
docker exec -it db-container psql -U admin -c "ALTER SYSTEM SET max_connections = 200;"
# Restart DB for the change to take effect
```

Option 2 — Use a connection pooler like **PgBouncer** in front of PostgreSQL. PgBouncer manages a pool of connections and reuses them efficiently — the application thinks it has many connections but PgBouncer multiplexes them into fewer real DB connections.

Option 3 — Check the application for connection leaks — connections being opened but never closed.

---

**Hidden Cause 3 — pg_hba.conf Blocking the Connection (PostgreSQL Specific)**

PostgreSQL has a host-based authentication file called `pg_hba.conf` that controls which hosts are allowed to connect, with which user, to which database, and using which authentication method. Even if credentials are correct and network is fine, if `pg_hba.conf` does not have an entry allowing connections from the app container's IP — the connection is refused.

**How to verify:**

Check the DB logs for this specific message:

```bash
docker logs db-container | grep "pg_hba"
```

```
FATAL: no pg_hba.conf entry for host "172.18.0.3", user "admin", database "appdb", SSL off
```

This is the dead giveaway. The connection was received but rejected by `pg_hba.conf`.

**How to fix:**

```bash
# View current pg_hba.conf
docker exec -it db-container cat /var/lib/postgresql/data/pg_hba.conf
```

Add an entry to allow the app container's subnet:

```
# pg_hba.conf
# TYPE   DATABASE   USER    ADDRESS          METHOD
host     appdb      admin   172.18.0.0/16    md5
# or allow all Docker internal IPs:
host     all        all     172.0.0.0/8      md5
```

Or set this via environment variable when running the PostgreSQL container:

```bash
docker run -d \
  -e POSTGRES_HOST_AUTH_METHOD=trust \   # Allow all connections (dev only — never in prod)
  postgres:15
```

For production, always use `md5` or `scram-sha-256` with specific IP ranges.

---

**Hidden Cause 4 — Container Resource Limits Causing Silent Crashes**

The DB container was healthy when you checked — but it keeps crashing and restarting due to memory limits. The status briefly shows `Up` between restarts — making it look like it is running fine. But during the crash/restart cycle, connections are refused.

**How to verify:**

```bash
# Watch for restarts in real time
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.RunningFor}}"

# Check restart count
docker inspect db-container --format '{{.RestartCount}}'
# If this is > 0 — the container has been restarting
```

```bash
# Check if OOM (Out of Memory) killed the container
docker inspect db-container --format '{{.State.OOMKilled}}'
# Returns: true — OOM killer terminated the container
```

```bash
# Check resource usage right now
docker stats db-container --no-stream
```

Output:
```
NAME           CPU %   MEM USAGE / LIMIT   MEM %
db-container   45.2%   495MiB / 512MiB     96.7%
```

Memory is at 96.7% — the container is about to be OOM killed again.

**The fix:**

Increase the memory limit:
```bash
docker run -d \
  --name db-container \
  --memory="2g" \           # Increase from 512MB to 2GB
  --memory-swap="2g" \
  postgres:15
```

Or in `docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:15
    deploy:
      resources:
        limits:
          memory: 2g
```

---

**Hidden Cause 5 — Application Connection Pool is Exhausted**

The problem is not on the database side at all — it is on the application side. The application uses a connection pool to manage database connections. If the pool is exhausted — all connections are in use and none are being released — new requests queue up and eventually time out or get refused.

**How to verify:**

```bash
# Check application logs for pool-related errors
docker logs app-container | grep -i "pool\|timeout\|connection"
```

Common messages:
```
Error: Connection pool timeout — could not obtain a connection within 30000ms
Error: remaining connection slots are reserved for non-replication superuser connections
```

**The fix:**

In Node.js (using `pg` library):
```javascript
const pool = new Pool({
  max: 20,              // Maximum pool size — increase if needed
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
})
```

Check for connection leaks — every `pool.connect()` must have a corresponding `client.release()`.

---

**Hidden Cause 6 — Firewall / iptables Rules on the Host**

Even though Docker manages its own networking, the host machine's `iptables` rules can interfere with Docker network traffic. A system administrator or a security tool may have added rules that block inter-container communication.

**How to verify:**

```bash
# List all iptables rules — look for REJECT or DROP rules affecting Docker subnets
sudo iptables -L -n -v | grep -i "172\|docker\|reject\|drop"
```

Also check Docker's own iptables rules:
```bash
sudo iptables -L DOCKER -n -v
sudo iptables -L DOCKER-USER -n -v
```

A `REJECT` or `DROP` rule targeting the Docker bridge subnet would silently block inter-container traffic.

**How to fix:**

Add an explicit allow rule for Docker's subnet:
```bash
sudo iptables -I DOCKER-USER -s 172.18.0.0/16 -d 172.18.0.0/16 -j ACCEPT
```

Or restart Docker — Docker rebuilds its iptables rules on restart:
```bash
sudo systemctl restart docker
```

---

**Hidden Cause 7 — DNS Caching — Stale IP Address**

Docker containers get new IP addresses every time they restart. If the database container was restarted but the application container cached the old IP address, it tries to connect to an IP that no longer belongs to the DB container.

**How to verify:**

```bash
# Check what IP the DB container has now
docker inspect db-container --format '{{.NetworkSettings.Networks.azureshop-network.IPAddress}}'
# Output: 172.18.0.4

# Check what IP the app is trying to connect to — from inside the app container
docker exec -it app-container nslookup db-container
# If this returns 172.18.0.3 (old IP) — stale DNS cache
```

**The fix:**

Restart the application container — this clears its DNS cache:
```bash
docker restart app-container
```

This is another reason to always use container names as hostnames instead of IP addresses. On a custom Docker network, Docker's internal DNS automatically resolves container names to their current IP — but only if the application is re-resolving the hostname, not caching the IP indefinitely.

Configure the application's connection pool to re-resolve DNS periodically:
```javascript
// Node.js — set family to force fresh DNS lookup
const pool = new Pool({
  host: 'db-container',
  // Do not cache the resolved IP — let it re-resolve on reconnect
})
```

---

**Hidden Cause 8 — Special Characters in Password Breaking the Connection String**

The credentials are correct — but the password contains special characters (`@`, `#`, `!`, `/`, `?`) that break the connection URL when they are not URL-encoded.

**Example:**
```
DATABASE_URL=postgres://admin:p@ss#word!@db-container:5432/appdb
```

The `@` in `p@ss#word!` is interpreted as the host separator — so the parser reads:
- username: `admin`
- password: `p`
- host: `ss#word!@db-container`

Connection fails because the host is wrong.

**The fix — URL-encode special characters:**

```
@ → %40
# → %23
! → %21
/ → %2F
? → %3F

Correct URL:
DATABASE_URL=postgres://admin:p%40ss%23word%21@db-container:5432/appdb
```

Or better — pass credentials as separate environment variables instead of a single connection URL:

```yaml
environment:
  DB_HOST: db-container
  DB_PORT: 5432
  DB_USER: admin
  DB_PASSWORD: p@ss#word!    # No URL encoding needed when passed as separate vars
  DB_NAME: appdb
```

---

**Complete deeper troubleshooting checklist:**

```
All obvious causes eliminated — still Connection Refused
            |
  ┌─────────┼──────────┐──────────────┐──────────────┐
  ↓         ↓          ↓              ↓              ↓
DB ready?  Max conn   pg_hba.conf   OOM restart   Pool
(logs)     reached?   blocking?     (OOMKilled)   exhausted?
  ↓         ↓          ↓              ↓              ↓
depends_on PgBouncer  Fix hba.conf  Increase      Fix pool
healthcheck or raise  entry         memory limit  leak
           max_conn
            |
  ┌─────────┼──────────┐
  ↓         ↓          ↓
iptables  Stale DNS  Special chars
rules?    cache?     in password?
  ↓         ↓          ↓
Add allow  Restart   URL-encode
iptables   app       or split vars
rule       container
```

---

**AzureShop Real Example:**

In AzureShop, after a DB container restart during a maintenance window, the `payment-service` kept throwing `Connection Refused` even though all the obvious things checked out — containers running, same network, correct credentials.

The root cause took 20 minutes to find: the `payment-service` connection pool was caching the old IP address of the DB container. After the DB restart, the DB had a new IP (`172.18.0.5` instead of `172.18.0.4`). The app was still trying to connect to `.4` which no longer existed.

Fix: restarted the `payment-service` container to flush the DNS cache. Immediate fix.

Long-term fix: added `idleTimeoutMillis` and `connectionTimeoutMillis` to the connection pool config so idle connections are released and re-established — forcing fresh DNS resolution on reconnect.

---

**Summary (what to say if time is short):**

*"When the obvious causes are eliminated, I look at eight deeper causes. First — the database container is running but the database process inside is still initialising — the fix is a proper health check with `pg_isready` and `depends_on: condition: service_healthy`. Second — the database has hit its `max_connections` limit and is rejecting new connections — check with `pg_stat_activity` and use PgBouncer or increase the limit. Third — `pg_hba.conf` in PostgreSQL is blocking connections from the app container's IP — check DB logs for the `pg_hba.conf entry` error. Fourth — OOM kills are causing silent DB restarts — check `OOMKilled` flag and `docker stats`. Fifth — the application's connection pool is exhausted due to a connection leak. Sixth — host-level iptables rules are blocking inter-container traffic. Seventh — stale DNS cache in the app container is pointing to the DB's old IP after a restart — fix by restarting the app container. Eighth — special characters in the password are breaking the connection URL and need URL-encoding. In AzureShop, we hit the stale DNS cache issue after a DB container restart during maintenance — the payment-service was still pointing to the old IP."*

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
