# Release Engineering — Real Interview Questions & Answers

> This file is a personal log of actual Release Engineer-level questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Banking Domain Client | Release Engineer | Technical Round](#interview-1)
  - [Q1. Have you worked with GitLab CI/CD in your project(s) — past or present?](#q1-have-you-worked-with-gitlab-cicd-in-your-projects--past-or-present)
  - [Q2. Can you explain the stages of GitLab CI/CD?](#q2-can-you-explain-the-stages-of-gitlab-cicd)
  - [Q3. What is GitLab Runner, and what's the difference between GitLab and GitLab Runner?](#q3-what-is-gitlab-runner-and-whats-the-difference-between-gitlab-and-gitlab-runner)

---

## Interview #1

**Company:** Banking Domain Client (confidential)
**Date:** 07-09-2026
**Role Applied For:** Release Engineer
**Round:** Technical Round
**Interviewer Level:** Not specified

---

### Questions Asked

#### Q1. Have you worked with GitLab CI/CD in your project(s) — past or present?

**Answer:**

Upfront honesty, same approach I use for every tool question: I haven't run GitLab CI/CD in production. My real, hands-on pipeline experience is **Jenkins** (FinBank, a banking-style platform on EKS) and **Azure Pipelines** (AzureShop). But a pipeline is a pipeline — the same design thinking (fail fast, scan before push, promote one immutable artifact through environments, gate production behind approval) transfers directly to GitLab CI/CD. I know the tool conceptually well enough to design and read a `.gitlab-ci.yml` correctly, and I'd rather say that plainly than pretend to a tool I haven't actually run — it survives follow-up questions better.

---

**Why this is a safe, credible answer rather than a weak one**

The interviewer isn't really testing "have you clicked buttons in GitLab specifically" — they're testing whether you understand what a CI/CD pipeline needs to *do*. Jenkins and GitLab CI/CD solve the exact same problem with different syntax:

| Concept | Jenkins (what I've actually built) | GitLab CI/CD |
|---|---|---|
| Pipeline-as-code file, versioned in the repo | `Jenkinsfile` | `.gitlab-ci.yml` |
| The machine executing a job | Jenkins **agent/node** | **GitLab Runner** (shell, Docker, or Kubernetes executor) |
| Where secrets/credentials live | Jenkins credential store, injected via `withCredentials([...])` | **CI/CD Variables** (Settings → CI/CD → Variables), can be marked masked + protected |
| A logical phase (Build/Test/Deploy) | `stage('Build') { steps { ... } }` | `stage: build` set on a job |
| The actual unit of work | steps inside a `stage` block | a **job** — multiple jobs can share one stage and run in parallel |
| Passing a build output to the next phase | `stash` / `unstash` | **artifacts** |
| Speeding up repeat builds (deps, `node_modules`, `.m2`) | not a first-class Jenkins concept — usually just leaned on agent disk | **cache**, keyed explicitly |
| Auto-discovering every branch | Multibranch Pipeline | pipelines run per-branch/per-MR automatically — no separate job type needed |
| Requiring a human click before prod | `input` step | `when: manual` on the job |

One structural difference worth calling out if asked directly: Jenkins is a **separate CI server** you install and point at a Git host — GitLab CI/CD is **built into GitLab itself**, so there's no separate server to stand up; you just add the YAML file and register a Runner against the project or group.

---

**Summary (what to say if time is short):**

*"My hands-on pipeline experience is Jenkins, on a banking-style platform on EKS, and Azure Pipelines on another project — not GitLab CI/CD specifically. But I understand it well conceptually: `.gitlab-ci.yml` is the direct equivalent of a Jenkinsfile — pipeline-as-code versioned with the repo — GitLab Runners are the equivalent of Jenkins agents, and CI/CD Variables are the equivalent of the Jenkins credential store. The pipeline design principles I've actually applied — fail fast on tests before building, scan the image before it's ever pushed, promote the exact same artifact through every environment, and gate production behind a manual approval — apply identically in GitLab CI/CD, just expressed with `stages`, `rules`, and `when: manual` instead of Jenkins' `stage`/`when`/`input` syntax."*

---

#### Q2. Can you explain the stages of GitLab CI/CD?

**Answer:**

At its simplest, a GitLab pipeline is a sequence of **stages**, and each stage contains one or more **jobs**. Stages run in order — top to bottom; jobs *within* the same stage run in parallel, and the next stage only starts once every job in the current stage has succeeded. The classic default GitLab uses in its own docs is three stages: `build → test → deploy`. In practice, for anything production-grade — and especially for a banking application, where an interviewer is going to want to hear "security gate" and "approval before prod" — I'd extend that to something much closer to the real pipeline I already run in Jenkins for FinBank, just re-expressed in GitLab's terms.

---

**1. The real 9-stage Jenkins pipeline I run today (FinBank), for grounding**

```
1. Checkout          — pull source, log branch + commit
2. Install Tools     — AWS CLI, Trivy (installed fresh per build agent)
3. Build             — mvn clean package -DskipTests (compile first, fast)
4. Unit Tests        — mvn test (pipeline stops here on failure — fail fast)
5. SonarQube         — static analysis, non-blocking (marks build "unstable")
6. Docker Build      — local image build, for scanning
7. Trivy Scan        — --exit-code 1 --severity CRITICAL,HIGH  ← BLOCKING GATE
8. Build + Push       — multi-arch buildx, push to ECR
9. Update Helm Chart  — bump image tag in Infra repo → ArgoCD deploys
```

**2. The same design, expressed as GitLab CI/CD stages**

```yaml
stages:
  - build
  - test
  - package
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  IMAGE: registry.example.com/finbank/backend

build-job:
  stage: build
  script:
    - mvn clean package -DskipTests      # compile first — cheap, fast signal
  artifacts:
    paths:
      - target/*.jar                     # passed forward to later stages

unit-test-job:
  stage: test
  script:
    - mvn test                           # pipeline stops here on failure

sonarqube-job:
  stage: test
  script:
    - sonar-scanner
  allow_failure: true                    # visible, doesn't hard-fail — same trade-off as Jenkins

package-job:
  stage: package
  # build, scan, and push all live in ONE job here on purpose — a Docker image built
  # in one job isn't automatically visible to a separate job (different runner/context
  # unless it's pushed or saved as an artifact), so build→scan→push runs as one
  # sequential script, same as the real Jenkins pipeline. Needs docker:dind on the
  # runner, or a daemonless builder like Kaniko/BuildKit if the runner is on Kubernetes.
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHORT_SHA .
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE:$CI_COMMIT_SHORT_SHA   # only reached if the trivy line above exits 0

deploy-dev-job:
  stage: deploy-dev
  script:
    - ./scripts/bump-helm-tag.sh dev $CI_COMMIT_SHORT_SHA
  environment:
    name: dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-prod-job:
  stage: deploy-prod
  script:
    - ./scripts/bump-helm-tag.sh prod $CI_COMMIT_SHORT_SHA
  environment:
    name: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual                        # human approval before prod — critical for banking
```

**Two stage-ordering decisions worth explaining out loud, because they're the actual substance behind "why these stages, in this order" — not just tool syntax:**

- **Tests run before the image is even built.** `mvn test` runs as its own stage right after compile — if it fails, the pipeline stops immediately, so no time is wasted building or scanning an image for code that's already broken.
- **The security scan gates the push, not just the build.** Inside `package-job`, the `trivy image` line runs *before* `docker push`, in the same script — if Trivy finds a HIGH/CRITICAL CVE it exits non-zero, GitLab fails the job right there, and the `docker push` line never executes. That means a vulnerable image is never made available in the registry at all — it's not "built then flagged," it's "built then blocked." (Note: `when: manual` has to live *inside* a `rules:` entry, not as a separate top-level key alongside `rules:` — GitLab's linter rejects that combination.)

---

**3. Extending this to a full banking-grade pipeline — every stage a regulated environment expects**

Honest framing on this part too: the two gates I've actually wired up and run for real are **SonarQube** (code quality) and **Trivy** (container image CVE scanning). The rest of what follows — SAST, secret detection, dependency/SCA scanning, license scanning, DAST, SBOM generation, image signing, and a change-ticket-gated production approval — I haven't personally hooked all of these into one pipeline. But I know exactly *why* a bank needs each one and where it slots in, and I'd rather design the complete, correct answer and say plainly which pieces are real vs. designed, than pretend the two I've built are the whole picture.

**Why a bank specifically needs more than "build → test → deploy":** a banking application handles money movement and PII, so the threat model isn't just "does the code work" — it's "did anyone commit a credential," "does a dependency have a known CVE," "does the running app have an exploitable endpoint," "can we prove exactly what's in this artifact if an auditor asks six months from now," and "did a human with change-management authority actually approve this touching production." Each stage below maps to one of those questions.

| Stage | What it actually catches | Example tool | Blocking? |
|---|---|---|---|
| **Build** | compiles the code | `mvn package` | — |
| **Unit Test** | logic regressions | `mvn test` | Yes |
| **Lint / Code Quality** | code smells, complexity | SonarQube | No — visible, not hard-fail |
| **SAST** (Static App Security Testing) | vulnerable *patterns in source code* — SQL injection, hardcoded secrets in logic, unsafe deserialization | GitLab SAST (built-in `include:` template, Semgrep-based) | Yes |
| **Secret Detection** | credentials/API keys/tokens accidentally committed to the repo | GitLab Secret Detection / gitleaks | Yes — this is the one most banks treat as non-negotiable |
| **Dependency Scanning (SCA)** | known CVEs in third-party libraries the app depends on | GitLab Dependency Scanning / OWASP Dependency-Check | Yes, on Critical/High |
| **License Compliance** | a transitive dependency pulled in under a license legal hasn't approved (e.g. AGPL) | GitLab License Scanning / FOSSA | Manual review, not auto-block |
| **Container Build + Image Scan** | known CVEs baked into the final image (OS packages, base image) | Trivy | Yes, on Critical/High |
| **SBOM Generation** | produces a Software Bill of Materials — a signed inventory of every component in the artifact, so if a CVE is disclosed *later*, you can grep the SBOM instead of re-scanning every running service | Syft / `trivy image --format cyclonedx` | — (artifact, not a gate) |
| **Image Signing** | proves the image running in prod is *exactly* the one that passed every gate above, not something swapped in later | cosign (Sigstore) | Yes, if the cluster enforces signature verification on deploy |
| **Deploy to Dev / QA** | — | Helm/GitOps | — |
| **DAST** (Dynamic App Security Testing) | vulnerabilities only visible in a *running* app — auth bypass, exposed debug endpoints, missing security headers | OWASP ZAP / GitLab DAST, run against the deployed QA/staging URL | Yes, on High/Critical |
| **Deploy to Staging** | pre-prod sign-off environment | Helm/GitOps | usually a manual click |
| **Change-ticket gate** | proves a *specific, approved* change record authorized this deploy — standard CAB (Change Advisory Board) practice at banks | pipeline checks a `CHANGE_TICKET` variable against a Jira/ServiceNow API before allowing the prod job to run | Yes |
| **Deploy to Production** | — | `when: manual`, gated behind the change-ticket check above | human approval required |
| **Post-deploy Verify / Smoke Test** | confirms the new version is actually healthy before calling the release done | scripted health-check hitting `/actuator/health` or similar | triggers rollback if it fails |
| **Notify / Audit Log** | leaves a compliance-traceable record of who approved what, when | Slack/Teams webhook + a logged pipeline event | — |

**A couple of these deserve one more sentence each, because they're the ones most likely to get a follow-up question:**

- **Secret detection is separate from SAST** even though both scan source code — SAST looks for *insecure patterns* in how code is written; secret detection looks for *literal committed values* (a real key, a real password) using regex/entropy checks. A bank cares enormously about the second one specifically because a leaked credential is an immediate, exploitable incident, not a theoretical weakness.
- **SBOM + image signing exist to answer one question**: "prove to an auditor that what's running in production right now is the exact, unmodified artifact that passed every gate." Without signing, someone with registry access could push an unscanned image with the same tag; without an SBOM, when a new CVE drops for some library, you'd have to re-scan everything instead of just checking a manifest.
- **The change-ticket gate is the release-engineering-specific piece** — it's what separates "CI/CD pipeline" from "a pipeline a bank's change-management process will actually sign off on." In practice this is a small script step before the `deploy-prod-job` that calls out to Jira/ServiceNow's API, checks the ticket referenced in a CI/CD variable is in an "Approved" state, and fails the job if it isn't — so `when: manual` alone isn't the whole control, it's "manual, *and* provably tied to an approved change record."

**Illustrative snippets for the new pieces most worth being able to sketch on a whiteboard** — note these assume the top-level `stages:` list from section 2 is extended to also declare `security-scan`, `deploy-qa`, and `verify`; GitLab requires every job's `stage:` to already exist in that list or the pipeline fails to parse before it even runs:

```yaml
secret-detection-job:
  stage: security-scan
  script:
    - gitleaks detect --source . --exit-code 1     # fails the job if any secret pattern is found

sast-job:
  stage: security-scan
  script:
    - semgrep ci --config auto                     # or: include GitLab's SAST.gitlab-ci.yml template

change-ticket-gate:
  stage: deploy-prod
  script:
    - ./scripts/verify-change-ticket.sh "$CHANGE_TICKET"   # calls Jira/ServiceNow API, fails if not "Approved"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-prod-job:
  stage: deploy-prod
  needs: ["change-ticket-gate"]                     # never runs unless the ticket check passed
  script:
    - ./scripts/bump-helm-tag.sh prod $CI_COMMIT_SHORT_SHA
  environment:
    name: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

post-deploy-verify:
  stage: verify
  script:
    - ./scripts/smoke-test.sh production || ./scripts/rollback.sh production
```

**The full flow, put together:**

```
Build → Unit Test → [SAST | Secret Detection | Dependency Scan | License Scan] (parallel)
  → Package (Docker build → Trivy scan → SBOM generate → cosign sign → push, gated sequentially)
  → Deploy Dev → Deploy QA → DAST (against the live QA URL)
  → Deploy Staging (manual sign-off)
  → Change-ticket gate → Deploy Prod (when: manual)
  → Post-deploy smoke test → auto-rollback on failure, or Notify/Audit log on success
```

---

**4. GitLab-specific concepts worth knowing on top of the stage list itself**

| Concept | What it's for |
|---|---|
| `.gitlab-ci.yml` | the pipeline file itself, root of the repo — direct equivalent of a `Jenkinsfile` |
| **GitLab Runner** | the actual executor that runs a job — shared runners on GitLab.com, or self-hosted (Docker/Kubernetes/shell executor) — equivalent of a Jenkins agent |
| **artifacts** vs **cache** | artifacts pass files *forward between stages of the same pipeline run* (e.g. the built `.jar`); cache persists dependencies *across different pipeline runs* (e.g. `node_modules`, `.m2`) to speed up repeat builds — these solve different problems and get confused often |
| **rules / only-except** | controls *when* a job runs — by branch, tag, merge request, or schedule; `rules:` is the modern, more flexible syntax |
| **environment:** | tracks a named deployment target (dev/staging/production) and gives GitLab a deployment history/dashboard per environment |
| **when: manual** | the job won't run until a human clicks it in the GitLab UI — this is exactly the approval gate a banking client will expect before a production deploy |
| **needs:** | lets a job start as soon as its specific dependency finishes, instead of waiting for the whole previous stage — turns a strictly linear pipeline into a faster DAG |
| **Merge Request pipelines** | a pipeline that runs specifically against a merge request (not just a branch push), so code is validated before it merges — same intent as a Jenkins Multibranch Pipeline building a PR |
| Built-in security templates | GitLab ships ready-made `include:` templates for SAST, Dependency Scanning, and Container Scanning — the same category of gate as the Trivy stage above, but maintained by GitLab rather than hand-rolled |

---

**Summary (what to say if time is short):**

*"A GitLab pipeline is made of stages that run in order, and within a stage, jobs run in parallel — the next stage only starts once every job in the current one succeeds. The baseline is build, test, package, and deploy per environment, and that's what I've actually built and run in Jenkins for a banking platform: tests run first so a broken build never wastes time on an image build, a Trivy scan blocks the image from ever being pushed if it has a HIGH or CRITICAL CVE, and the same pushed image gets promoted through dev, staging, and production — never rebuilt per environment."*

**Extended answer, if they push specifically for a banking-grade list:**

*"For a regulated banking environment I'd add security gates at every layer, not just the container image: SAST and secret detection on the source code itself, dependency/SCA scanning for vulnerable third-party libraries, license compliance for legal, then the container scan, an SBOM and image signing so you can prove exactly what's running in production, DAST against the deployed app before it reaches staging, and — the release-engineering-specific piece — a change-ticket gate before production that checks a Jira or ServiceNow ticket is actually in an approved state, not just a human clicking `when: manual`. I want to be upfront that SonarQube and Trivy are the two gates I've actually run in production; the rest — SAST, secret detection, DAST, SBOM/signing, the change-ticket gate — I know precisely why each one matters and where it sits in the pipeline, but I'm describing the complete design rather than claiming I've operated all of them personally."*

---

#### Q3. What is GitLab Runner, and what's the difference between GitLab and GitLab Runner?

**Answer:**

**GitLab** is the platform itself — it hosts the source code, the merge requests, and the pipeline *definition*. The `.gitlab-ci.yml` file lives in the repo, and GitLab reads it to know what stages and jobs exist, in what order, and under what conditions each one should run. GitLab is the **orchestrator** — it decides *what* needs to happen and *when*, and it's where you see the pipeline dashboard, logs, and approval buttons.

**GitLab Runner** is a separate, lightweight agent that does the actual **execution**. GitLab itself never runs `mvn test` or `docker build` — it just makes a job available. GitLab Runner is the program that picks that job up, runs the real shell commands on some actual compute — a VM, a Docker container, or a Kubernetes pod — and reports the result (pass/fail, full logs) back to GitLab.

**In one line:** GitLab is where the pipeline is defined and tracked; GitLab Runner is the agent that actually provides the compute to run each job.

---

**How they actually talk to each other**

1. A Runner is **registered** against a project, a group, or the whole GitLab instance, using a registration token.
2. The Runner continuously polls GitLab (or is notified) asking "any jobs waiting for me?"
3. When a pipeline triggers, GitLab queues each job and hands it to any available, matching Runner (matching can be scoped by **tags** — e.g. a job tagged `docker` only goes to a Runner that's registered with that tag).
4. The Runner executes the job's `script:` lines using its configured **executor** — `shell` (runs directly on the Runner's host), `docker` (runs inside a container per job — the most common choice), or `kubernetes` (runs as a pod in a cluster).
5. Logs stream back to GitLab in real time, and the final pass/fail result updates the pipeline dashboard.

---

**Shared Runners vs. self-hosted Runners — the part worth raising unprompted in a banking interview**

GitLab.com provides free **shared runners** it manages for you — fine for a portfolio project, zero setup. A bank will almost always require **self-hosted Runners** instead, registered inside the company's own private network, for two concrete reasons:

- **Network access:** jobs often need to reach internal systems directly — an internal database, an internal artifact repository, an internal deployment target sitting in a private subnet. A shared, public Runner outside the company's network simply can't reach those.
- **Compliance/data residency:** a bank generally can't have its source code, build artifacts, or secrets ever touch infrastructure it doesn't control or audit — which shared SaaS runners don't satisfy.

---

**Summary (what to say if time is short):**

*"GitLab is the platform that hosts the code and the pipeline definition — it decides what jobs need to run and in what order, and it's where the pipeline dashboard and approval steps live. GitLab Runner is a separate agent that does the actual execution: it's registered against the project, polls for jobs, and runs the real script for each job using an executor like Docker or Kubernetes, then reports the result back. So GitLab is the orchestrator and Runner is the worker providing compute. For a banking environment specifically, I'd expect self-hosted Runners running inside the company's own network rather than GitLab's shared SaaS runners, both so jobs can reach internal systems directly and so code and secrets never leave infrastructure the company controls."*

---
