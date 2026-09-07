# Release Engineering — Real Interview Questions & Answers

> This file is a personal log of actual Release Engineer-level questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Banking Domain Client | Release Engineer | Technical Round](#interview-1)
  - [Q1. Have you worked with GitLab CI/CD in your project(s) — past or present?](#q1-have-you-worked-with-gitlab-cicd-in-your-projects--past-or-present)
  - [Q2. Can you explain the stages of GitLab CI/CD?](#q2-can-you-explain-the-stages-of-gitlab-cicd)
  - [Q3. What is GitLab Runner, and what's the difference between GitLab and GitLab Runner?](#q3-what-is-gitlab-runner-and-whats-the-difference-between-gitlab-and-gitlab-runner)
  - [Q4. Walk me through a basic CI/CD pipeline](#q4-walk-me-through-a-basic-cicd-pipeline)
  - [Q5. What happens when a CI/CD pipeline fails before production?](#q5-what-happens-when-a-cicd-pipeline-fails-before-production)
  - [Q6. What happens when a CI/CD pipeline fails during production?](#q6-what-happens-when-a-cicd-pipeline-fails-during-production)
  - [Q7. How do you troubleshoot a failed CI/CD pipeline?](#q7-how-do-you-troubleshoot-a-failed-cicd-pipeline)

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

#### Q4. Walk me through a basic CI/CD pipeline

**Answer:**

I'll walk through this as the actual production-grade pipeline a banking application needs in GitLab CI/CD — not the toy 3-stage `build → test → deploy` example — since that's what a banking interviewer is really probing for. Same honesty caveat as the earlier answers: the design below is built directly on the real Jenkins pipeline I run for FinBank (compile → test → SonarQube → Docker build → Trivy scan → push → Helm/ArgoCD deploy), extended with the additional banking-specific security and change-control stages a regulated environment requires. I'll say plainly which pieces I've personally operated versus designed, as I go.

---

**The full pipeline, as one connected story**

1. **A developer pushes code** to a feature branch and opens a merge request. GitLab reads `.gitlab-ci.yml` at the repo root and creates a pipeline automatically — nothing manual triggers it.

2. **Build** — the pipeline compiles the application (`mvn clean package -DskipTests` for a Java service, compiling first and fast, before anything else runs). The compiled artifact is kept as a GitLab **artifact** so later stages don't have to rebuild it.

3. **Test** — automated unit tests run (`mvn test`). If a test fails, the pipeline **stops immediately** — no time is wasted building an image or running security scans against code that's already broken. In parallel, a non-blocking **SonarQube** job flags code-quality issues without hard-failing the build — a real trade-off I've made on FinBank: a code smell shouldn't block a release the way a security issue should.

4. **Source-level security scanning** — this is the stage a generic answer skips and a banking interviewer specifically listens for. Three or four jobs run in parallel here, all against the source code itself, before anything gets built into an image:
   - **SAST** (Static Application Security Testing) — scans the code for vulnerable patterns like SQL injection or unsafe deserialization (GitLab's built-in SAST template, Semgrep-based).
   - **Secret Detection** — scans for literally committed credentials, API keys, or tokens (gitleaks or GitLab's Secret Detection). This is the one most banks treat as non-negotiable — a leaked credential is an immediate, exploitable incident, not a theoretical weakness.
   - **Dependency Scanning (SCA)** — checks every third-party library the app pulls in against known CVE databases.
   - **License Compliance** — flags a dependency pulled in under a license legal hasn't approved (e.g. AGPL) — routed to manual review rather than an automatic block.
   All four of these are blocking except license compliance, and all run against **source code**, before a container image even exists.

5. **Package** — only once build, tests, and source-level security scans pass does the pipeline build the deployable artifact. For a containerized app that means, in one sequential job on one runner: `docker build`, then **Trivy** scans that exact image for CRITICAL/HIGH CVEs — genuinely blocking, `--exit-code 1` — then a **Software Bill of Materials (SBOM)** gets generated so there's a signed inventory of everything in the artifact, then the image is signed with **cosign** so anyone can later prove the image running in prod is the exact one that passed every gate, and only then is it pushed to the registry, tagged with the immutable Git commit SHA — never `latest`.

6. **Deploy to Dev, then QA** — the pushed image gets promoted automatically to Dev, and then QA, by updating the image tag in a GitOps repo's Helm values — the same pushed artifact, never rebuilt per environment, so what was tested is exactly what ships.

7. **DAST** (Dynamic Application Security Testing) — once the app is actually running in QA, an OWASP ZAP-style scan hits the live URL looking for things you can only find in a running app — auth bypasses, exposed debug endpoints, missing security headers. This has to happen *after* a deploy, since there's nothing running to scan before that.

8. **Deploy to Staging** — a pre-prod sign-off environment, typically gated behind a manual click even before production, so the release can be validated one last time under production-like conditions.

9. **Change-ticket gate** — the release-engineering-specific control a bank actually asks for: before the production job is even allowed to run, a script step checks that a `CHANGE_TICKET` CI/CD variable references a ticket that's genuinely in an "Approved" state in Jira or ServiceNow — standard CAB (Change Advisory Board) practice. If that check fails, the production job is blocked outright, not just waiting on a human click.

10. **Deploy to Production** — `when: manual` inside a `rules:` block, so a human has to actively click deploy in the GitLab UI, and — because of step 9 — that click is only even available once there's a provably approved change record behind it.

11. **Post-deploy verify** — a scripted smoke test hits a health endpoint right after the deploy. If it fails, an automatic rollback script runs immediately; if it passes, a Slack/Teams notification and an audit log entry close out the release — the compliance-traceable record of who approved what, and when.

---

**The whole thing as one `.gitlab-ci.yml` skeleton**

```yaml
stages:
  - build
  - test
  - security-scan
  - package
  - deploy-dev
  - deploy-qa
  - dast-scan
  - deploy-staging
  - deploy-prod
  - verify

variables:
  IMAGE: registry.example.com/finbank/backend

build-job:
  stage: build
  script: [ "mvn clean package -DskipTests" ]
  artifacts:
    paths: [ "target/*.jar" ]

unit-test-job:
  stage: test
  script: [ "mvn test" ]

sonarqube-job:
  stage: test
  script: [ "sonar-scanner" ]
  allow_failure: true

sast-job:
  stage: security-scan
  script: [ "semgrep ci --config auto" ]

secret-detection-job:
  stage: security-scan
  script: [ "gitleaks detect --source . --exit-code 1" ]

dependency-scan-job:
  stage: security-scan
  script: [ "dependency-check --project finbank --scan . --failOnCVSS 7" ]

license-scan-job:
  stage: security-scan
  script: [ "license-finder" ]
  allow_failure: true          # routed to manual legal review, not an auto-block

package-job:
  stage: package
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHORT_SHA .
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE:$CI_COMMIT_SHORT_SHA
    - syft $IMAGE:$CI_COMMIT_SHORT_SHA -o cyclonedx-json > sbom.json
    - cosign sign --key cosign.key $IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE:$CI_COMMIT_SHORT_SHA   # only reached if every line above succeeded
  artifacts:
    paths: [ "sbom.json" ]

deploy-dev-job:
  stage: deploy-dev
  script: [ "./scripts/bump-helm-tag.sh dev $CI_COMMIT_SHORT_SHA" ]
  environment: { name: dev }
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-qa-job:
  stage: deploy-qa
  script: [ "./scripts/bump-helm-tag.sh qa $CI_COMMIT_SHORT_SHA" ]
  environment: { name: qa }
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

dast-job:
  stage: dast-scan
  script: [ "zap-baseline.py -t $QA_URL" ]
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy-staging-job:
  stage: deploy-staging
  script: [ "./scripts/bump-helm-tag.sh staging $CI_COMMIT_SHORT_SHA" ]
  environment: { name: staging }
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

change-ticket-gate:
  stage: deploy-prod
  script: [ './scripts/verify-change-ticket.sh "$CHANGE_TICKET"' ]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-prod-job:
  stage: deploy-prod
  needs: [ "change-ticket-gate" ]
  script: [ "./scripts/bump-helm-tag.sh prod $CI_COMMIT_SHORT_SHA" ]
  environment: { name: production }
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

post-deploy-verify:
  stage: verify
  script: [ "./scripts/smoke-test.sh production || ./scripts/rollback.sh production" ]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

**Summary (what to say if time is short):**

*"A basic pipeline is code push → build → test → package → deploy, but for a banking application I'd extend that with security gates at every layer: SAST and secret detection on the source code itself, dependency and license scanning for third-party risk, a container image scan plus an SBOM and signature on the built artifact, and DAST against the running app once it's deployed to QA. The same pushed artifact then promotes through Staging and Production without ever being rebuilt, and production specifically is gated behind two things, not one — a manual click, and a change-ticket check that confirms a real, approved change record authorized the deploy. After deploying, a smoke test either confirms the release is healthy or triggers an automatic rollback. The gates I've actually operated in production are SonarQube and Trivy on FinBank's Jenkins pipeline; the rest — SAST, secret detection, SCA, DAST, SBOM/signing, and the change-ticket gate — I'm describing as the complete, correct design for a regulated environment, and I'd say so directly if asked which parts I've personally run."*

---

#### Q5. What happens when a CI/CD pipeline fails before production?

**Answer:**

Before production, a failure is cheap — nothing has reached a real user yet, so the entire response is "stop, notify, fix, rerun." The specific mechanics, in GitLab terms:

**1. The pipeline halts exactly at the failing job — nothing downstream runs.** GitLab stages run in order, and the next stage only starts once every job in the current one succeeds. So if `unit-test-job` fails in the `test` stage, `security-scan`, `package`, and every `deploy-*` stage after it simply never execute — no image gets built, nothing gets pushed, nothing gets promoted anywhere. The blast radius is zero by construction, not by luck.

**2. Not every failure blocks — the pipeline distinguishes blocking from non-blocking.** A job like `sonarqube-job` or `license-scan-job` is marked `allow_failure: true`: it shows up as a visible, orange "failed (allowed to fail)" warning, but the pipeline keeps going. A job like `unit-test-job`, `sast-job`, `secret-detection-job`, or the Trivy line inside `package-job` has no such flag — those are genuinely blocking. That split is a deliberate design decision I've actually made on FinBank's real Jenkins pipeline (SonarQube non-blocking, Trivy blocking) for exactly the same reasoning: a code smell shouldn't stop a release the way an exploitable CVE or a leaked credential should.

**3. Merging is blocked at the source, not just the pipeline.** In GitLab, a protected branch can require "pipelines must succeed" as a merge condition — so even before anyone manually checks, a merge request with a red pipeline literally cannot be merged into `main`. That's the mechanism that turns "the test failed" into "this code cannot reach production," automatically.

**4. Notification is immediate, and fixing is fast to retry.** GitLab marks the job red in the UI, and typically that's wired to Slack/email/MR comments so the developer knows within minutes, not at the next standup. A genuinely useful detail worth mentioning: GitLab lets you **retry just the single failed job** without re-running every job that already passed — so if `dependency-scan-job` failed while `build-job` and `unit-test-job` already succeeded, fixing the dependency and hitting retry doesn't waste time recompiling and re-testing.

**5. One banking-specific nuance worth raising unprompted:** if the failure is specifically `secret-detection-job` catching a real committed credential, "it never reached production" doesn't mean the incident is over — that credential still exists in the branch's git history at the point it was committed. The correct response isn't just "remove it and continue," it's **remove it and rotate the actual credential**, since a real value briefly existed in a repository regardless of whether the pipeline ever let it ship anywhere. I've done exactly this for real on FinBank — a self-audit caught plaintext DB passwords and a JWT secret committed to a `.env` file — and the fix was purge-from-git plus full credential rotation, not just deleting the file going forward.

---

**Summary (what to say if time is short):**

*"Before production, a pipeline failure is contained by design — the pipeline stops at that stage, nothing downstream builds or deploys, so nothing reaches a real user. Non-blocking checks like SonarQube are allowed to fail and just show a warning; genuinely blocking checks like unit tests, SAST, secret detection, or the Trivy image scan stop the pipeline outright, and GitLab's protected-branch setting means a merge request with a failing pipeline literally can't be merged. The developer gets notified immediately, fixes the issue, and either reruns the whole pipeline or retries just the failed job without re-running what already passed. The one exception where 'it never reached prod' doesn't mean 'nothing to do' is a caught secret — that credential still existed in git history, so the real fix is removing it and rotating the credential, which is exactly what I had to do for a real leaked `.env` file on FinBank."*

---

#### Q6. What happens when a CI/CD pipeline fails during production?

**Answer:**

This is a fundamentally different situation from Q5 — real users may be affected, so the priority order flips from "fix the root cause" to "stop the bleeding first, understand it second." I'd split this into two distinct failure modes, because the right response is different for each, and being able to draw that distinction is exactly what a banking interviewer is listening for.

---

**Mode 1 — the `deploy-prod` job itself fails (Helm upgrade error, connection timeout, bad manifest)**

If it's configured correctly, this is actually the *safe* outcome: a Kubernetes rolling update only sends traffic to a new pod once its **readiness probe** passes. If the new version fails to come up healthy, the readiness check simply never passes, traffic never routes to it, and the old pods — which were never torn down — keep serving every request. The deploy job shows red in GitLab, but from a user's perspective, **nothing happened**. This is real behavior I rely on today on FinBank: a bad build fails its readiness check and traffic simply never reaches it, instead of users hitting a broken pod.

---

**Mode 2 — the deploy succeeds, but the new version is actually broken (the dangerous case)**

Here the new version is live and potentially already serving some traffic. The response has three layers, roughly in order of how fast they can act:

1. **Immediate, automated** — the `post-deploy-verify` smoke test I built into the Q4 pipeline hits a health endpoint right after deploy; if it fails, it triggers a rollback script automatically, before a human even needs to be paged. If the deployment strategy is canary (via Argo Rollouts or similar) rather than a plain rolling update, this can go further — an `AnalysisTemplate` continuously queries a live Prometheus metric like error rate, and if it crosses a threshold during the 20%-traffic canary step, the rollout aborts and reverts automatically, so only a small slice of traffic was ever exposed.

2. **Fast, human-triggered rollback** — if something only surfaces after the smoke test already passed (a real production incident, not something the deploy gate caught), the fastest safe options, in order of how I'd actually reach for them:
   - **Revert the Git commit** in the GitOps/infra repo that bumped the image tag — ArgoCD (or Flux) detects that revert and syncs the cluster back to the previous known-good version automatically. This is the cleanest option because it's auditable — the rollback is itself a normal, reviewed git commit, not an untracked manual action.
   - **`helm rollback <release> <previous-revision>`** directly, if the situation is urgent enough that waiting on GitOps reconciliation isn't acceptable.
   - **`kubectl rollout undo deployment/<name>`** as the last-resort, break-glass option.
   In GitLab specifically, the `environment:` block also gives a one-click **"Re-deploy"** button in the Deployments UI against any previous successful deployment — a native rollback path without touching a terminal at all.

3. **Incident process, in parallel with the technical rollback** — this is the part that's specific to a bank rather than a generic outage:
   - The alert that caught it (a Prometheus `HighErrorRate`-style alert, paging on-call) starts a real incident: severity assigned, stakeholders notified, a postmortem/RCA scheduled.
   - The **change ticket** that gated the original deploy (from Q4's change-ticket gate) gets updated with the actual outcome — rolled back, root cause, fix ETA — because the CAB record has to reflect what actually happened, not just what was approved to happen.
   - There's a full **audit trail**: who triggered the rollback, exactly when, and why — this is a hard requirement in banking, not a nice-to-have, since a regulator or internal audit can ask for this history well after the fact.
   - Depending on customer impact and how the bank's compliance function is set up, a production incident touching a customer-facing banking service can trigger **formal incident/SLA-breach reporting obligations** — I don't have hands-on experience with that specific regulatory reporting step, but I know it's a real downstream consequence in this domain and I'd defer to the compliance/incident-management process already in place rather than guess at it.

---

**The honest gap, stated directly:** the automated rollback step in `post-deploy-verify` is something I've designed for this pipeline, not something running in FinBank's real Jenkins pipeline today — that pipeline currently has no automated rollback wired up at all; a rollback there would still be a manual `helm rollback` or redeploying the previous tag by hand. It's a known, already-identified gap on my own list, not something I'm discovering for the first time in this answer.

---

**Summary (what to say if time is short):**

*"It depends which of two things actually happened. If the deploy job itself fails — a bad Helm upgrade, a timeout — a correctly configured rolling update means the readiness probe never passes for the new version, traffic never routes to it, and the old version keeps serving, so users see nothing. The dangerous case is when the deploy succeeds but the new version is actually broken — there I'd want an automated smoke test to catch it and trigger a rollback immediately, and for anything caught later by monitoring, the fastest safe path is reverting the Git commit that bumped the image tag so GitOps syncs back to the last known-good version automatically, or a direct `helm rollback` if it's urgent. In parallel with the technical rollback, a banking environment also needs the process side handled: the incident gets a postmortem, the change ticket that gated the original deploy gets updated with what actually happened, and there's a full audit trail of who rolled back what and when. I'll say directly that the automated-rollback piece is something I've designed here, not something running in my real Jenkins pipeline today — that's a genuine, already-known gap on my end, not something I'm papering over."*

---

#### Q7. How do you troubleshoot a failed CI/CD pipeline?

**Answer:**

I approach this the same disciplined way regardless of tool — Jenkins console output or a GitLab job log, the debugging instinct is identical: **don't guess, read the actual error, and work outward from the exact line that failed.** The mechanics below are GitLab-specific; the muscle memory underneath them is the same one I use reading Jenkins console output on FinBank today.

---

**Step 1 — find exactly where it broke, not just that it broke**

GitLab's pipeline graph view shows the stage/job that's red at a glance — click straight into that job's log rather than starting from the top of the pipeline. The job log is the full script output for that one job; scroll to the actual non-zero exit near the bottom, since the real error is often buried under normal verbose output above it, not the first suspicious-looking line you see.

**Step 2 — classify the failure before trying to fix anything**

Almost every pipeline failure falls into one of these categories, and each has a genuinely different fix — treating all of them as "just retry" wastes time and treating all of them as "code bug" wastes time in the opposite direction:

| Category | What it looks like | How I'd confirm it | Fix |
|---|---|---|---|
| **Actual code/test failure** | A specific assertion or compile error in the log | The stack trace names a real file/line | Fix the code, push, let the pipeline rerun |
| **`.gitlab-ci.yml` config error** | Pipeline fails to even *start*, or a job errors before running any script | GitLab's **CI Lint** tool (Pipelines → Editor → Validate, or the `/-/ci/lint` page) — validates the YAML before it ever runs | Fix the YAML — common ones: a `stage:` not declared in the top-level `stages:` list, or `when:` used alongside `rules:` on the same job (GitLab rejects that combination outright) |
| **Runner/environment issue** | Job never starts, or fails immediately with something unrelated to the app (e.g. "docker: command not found") | Check the runner's status and its registered `tags:` against the job's `tags:` — a Docker-only job landing on a shell-executor runner fails this way | Fix runner tags/scoping, or check if the runner itself is offline/out of disk |
| **Missing or misscoped secret** | Auth failure that looks unrelated to the actual change (e.g. "403" pushing to the registry) | Check Settings → CI/CD → Variables — a variable marked **Protected** is only exposed to pipelines running on a protected branch; a feature-branch pipeline silently doesn't get it | Un-protect the variable if it genuinely needs to run on feature branches, or confirm the branch should be protected instead |
| **Security gate doing its job correctly** | Trivy/SAST/secret-detection job fails on purpose | Read the actual scan report, don't just see "failed" and assume it's broken tooling | Fix the real finding, or — only for genuinely unfixable CVEs — document a `.trivyignore`/`--ignore-unfixed` waiver, never silently disable the gate |
| **External dependency/service down** | Timeout pulling from a registry, npm/Maven repo, or (in the banking pipeline from Q4) the Jira/ServiceNow change-ticket API | Check the target service's status directly, not just retry blindly | Retry once to rule out a blip; if it's a genuine outage, that's now a dependency incident, not a pipeline bug |
| **Flaky/non-deterministic test** | Fails intermittently, unrelated to the actual diff | Re-run the same commit — if it passes on retry with no code change, that's the signature | Don't just keep retrying forever — a test that's flaky enough to need retries regularly needs to actually be fixed, or it quietly trains the team to stop trusting red pipelines |

**Step 3 — use GitLab's built-in tooling to go deeper when the log alone isn't enough**

- **Artifacts + the Tests tab** — if JUnit XML is uploaded as a job artifact, GitLab renders it directly as a pass/fail breakdown per test in the merge request, instead of making you grep raw log text for which of 200 tests actually failed.
- **Retry a single job** — GitLab lets you retry just the failed job without re-running everything upstream that already passed, the same time-saver as re-running a single failed stage in Jenkins instead of the whole build.
- **`CI_DEBUG_TRACE: "true"`** — a built-in verbose mode that dumps every shell command the runner actually executes, useful when the script's own output doesn't explain *why* it behaved the way it did. Worth a caution alongside it: debug trace can print secret values into the log, so it should only be enabled temporarily, and GitLab requires it to be explicitly allowed per-project for exactly that reason.
- **Interactive web terminal** — for supported executors, GitLab can open a live terminal into a *running* job, which is the closest equivalent to SSHing onto a Jenkins agent mid-build to poke around instead of only reading logs after the fact.

**Step 4 — fix, verify, and don't stop at green**

Push the fix, confirm the pipeline goes green, and if the failure was a genuine flake or a config mistake that could recur, treat it the same way I'd treat any recurring reliability issue — worth a short note on *why* it happened, not just that it's fixed now, especially if it's the kind of thing that could block a release again right before a deadline.

---

**Summary (what to say if time is short):**

*"First I go straight to the specific job that's red in the pipeline graph and read its log to find the actual line that exited non-zero — not just 'it failed.' Then I classify it: is this a real code/test bug, a `.gitlab-ci.yml` config mistake — which GitLab's CI Lint tool catches before it even runs — a runner or missing-secret issue, a security gate correctly doing its job, an external dependency being down, or a flaky test. Each of those has a different fix, so I don't just retry blindly. For anything the log doesn't fully explain, GitLab has a debug trace mode that dumps every command executed, and an interactive web terminal into a running job for real-time debugging — the same instinct as reading Jenkins console output, just different tooling. Once it's fixed, I push, confirm it's green, and if it was a flake or a config mistake likely to recur, I make sure that's actually understood and fixed, not just retried away."*

---
