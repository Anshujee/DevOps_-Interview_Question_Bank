# DevOps Concepts — Real Interview Questions & Answers

> This file is a personal log of actual general DevOps/SRE philosophy and concept questions asked to me by interviewers in real interviews — as opposed to questions tied to one specific tool (which live in DOCKER_INTERVIEW_QA.md, KUBERNETES_INTERVIEW_QA.md, TERRAFORM_INTERVIEW_QA.md, AWS_INTERVIEW_QA.md, PYTHON_INTERVIEW_QA.md).
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Coforge | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. What's the difference between SRE and DevOps?](#q1-whats-the-difference-between-sre-and-devops)
  - [Q2. What is SLA and SLO?](#q2-what-is-sla-and-slo)
  - [Q3. What is CI/CD? Explain briefly.](#q3-what-is-cicd-explain-briefly)
  - [Q4. A developer has written only source code — design the CI/CD pipeline to deploy it to DEV/QA/PROD using best practices](#q4-a-developer-has-written-only-source-code--design-the-cicd-pipeline-to-deploy-it-to-devqaprod-using-best-practices)

---

## Interview #1

**Company:** Coforge
**Date:** 22-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Senior DevOps Manager

---

### Questions Asked

#### Q1. What's the difference between SRE and DevOps?

**Answer:**

The cleanest way I've heard this framed — and the line I'd actually quote in an interview, because it's from Google's own SRE book and shows I've gone to the source rather than just picked up the term secondhand — is: **"class SRE implements interface DevOps."** DevOps is the philosophy — the *what* and *why*. SRE is one specific, concrete, heavily measurable way of actually *doing* it — the *how*, as Google originally built it.

---

**DevOps — a culture and a set of principles, not a fixed set of practices**

DevOps is a **movement/philosophy**, not a job title or a certification with one official rulebook. Its core goals:
- Break down the traditional silo between **Development** (build the thing) and **Operations** (keep the thing running) — shared ownership instead of "throw it over the wall"
- **Automate** as much of the build-test-deploy pipeline as possible — CI/CD
- Ship changes **frequently and safely**, in small increments, instead of big infrequent releases
- **"You build it, you run it"** — the team that writes the code is also on the hook for its production behavior, which naturally motivates writing more operable, observable code

Because it's a philosophy, there's no single prescribed mechanism for balancing "ship fast" against "keep it stable" — different organizations implement DevOps culture in different ways, with different tooling, different processes, different levels of rigor.

**SRE — a specific, quantifiable engineering discipline for reliability**

SRE (**Site Reliability Engineering**) is what Google built when they treated operations as a **software engineering problem**, with actual engineers writing code and tooling to solve operational problems instead of doing repetitive manual work by hand. It's far more prescriptive than "DevOps" as a general term — it comes with specific, named mechanisms:

| SRE concept | What it actually is |
|---|---|
| **SLI** (Service Level Indicator) | An actual measured metric — e.g., request success rate, p99 latency |
| **SLO** (Service Level Objective) | The internal target for that metric — e.g., "99.9% of requests succeed" |
| **SLA** (Service Level Agreement) | The external, often contractual promise to customers — usually looser than the internal SLO, to leave a safety margin |
| **Error budget** | `100% − SLO` — the amount of "acceptable unreliability" a team is allowed to spend before it must stop shipping new features and focus on stability |
| **Toil** | Manual, repetitive, automatable operational work that adds no lasting value — SRE explicitly tracks and caps how much time is spent on it |
| **Blameless postmortems** | A structured incident review focused on fixing systemic causes, explicitly not on assigning individual blame |

The **error budget** is the mechanism I'd highlight most, because it's the concrete thing DevOps-as-a-philosophy doesn't give you on its own: a **data-driven, non-political way to decide** whether a team should keep shipping features or pause and fix reliability. If a service's SLO is 99.9% and it's currently running at 99.95%, there's error budget left — ship freely. If an incident burns through the entire month's error budget, new feature releases to that service **stop**, by policy, not by a manager's judgment call, until reliability recovers. DevOps says "care about reliability and ship fast" as a cultural value; SRE turns that into an actual numeric gate.

---

**Side-by-side**

| | DevOps | SRE |
|---|---|---|
| **Nature** | Philosophy / cultural movement | Concrete engineering discipline / job function |
| **Origin** | Grassroots movement, ~2009, no single inventor | Google, ~2003, publicly documented in the SRE book (~2016) |
| **Prescriptiveness** | Loose — many valid implementations | Fairly prescriptive — specific named practices |
| **Balancing speed vs. stability** | Cultural/organizational — no fixed mechanism | Error budget — an explicit, quantified, automatic gate |
| **Job title?** | Not always a distinct role — often "how the whole engineering org works" | Usually an actual job title/team, with its own responsibilities |
| **Core question** | "How do we get Dev and Ops working together?" | "How do we make reliability a measurable, engineered property, not a feeling?" |

---

**Real-world example — CloudCart**

CloudCart's engineering org runs on DevOps culture as the baseline — CI/CD pipelines for every service, Terraform-managed infrastructure, and shared on-call between the people who write a service's code and the people who operate it, rather than a separate ops team taking pages for code they didn't write.

On top of that cultural baseline, we specifically adopted SRE practices for our most business-critical service — order processing — because "be reliable" as a value wasn't giving us a clear, consistent answer to "should we ship this risky change today." We defined explicit SLOs for it (99.95% success rate, p99 latency under 300ms), track the error budget monthly, and when a bad month burns through it — which happened once, after a cascading failure from a downstream payment provider outage — new feature deployments to that service were explicitly paused, by policy, in favor of reliability-focused work until the budget reset the following month. We also adopted blameless postmortems specifically for that service's incidents — the format explicitly asks "what in the system allowed this to happen," not "who did this," which measurably increased how much people were willing to write down in the postmortem versus the more guarded incident write-ups we had before.

Less critical internal tooling, by contrast, still just runs on general DevOps practices — CI/CD, automation, shared ownership — without a formally defined SLO or error budget, because the cost of an SRE-level rigor didn't match the actual business risk of that tooling being briefly down.

---

**Complete thought process — how I approach this in the interview**

```
Are they the same thing?
  → No, but they're related — not competing ideas, one implements
    the other: "class SRE implements interface DevOps"

DevOps = the philosophy
  → Break Dev/Ops silos, automate delivery, ship fast and often,
    shared ownership — the WHAT and WHY, loosely defined

SRE = one specific, engineering-driven implementation of it
  → Comes with concrete, named mechanisms DevOps doesn't mandate:
    SLIs/SLOs, error budgets (the key one — a data-driven gate
    for feature velocity vs. stability), toil limits, blameless
    postmortems

When would I reach for full SRE rigor vs. general DevOps practices?
  → Business-critical services: worth the overhead of defining
    SLOs/error budgets explicitly
  → Lower-stakes internal tooling: general DevOps practices are
    proportionate, formal SRE mechanisms would be overkill
```

---

**Summary (what to say if time is short):**

*"They're related, not interchangeable — the way I'd put it is 'SRE implements DevOps,' which is actually a line from Google's own SRE book. DevOps is the philosophy: break down the silo between development and operations, automate delivery, ship frequently, share ownership of what you build. It doesn't prescribe exactly how to balance shipping fast against staying stable — that's left to each organization's culture. SRE is Google's specific, much more measurable way of actually doing that: it defines SLIs and SLOs as concrete reliability targets, and the error budget — the gap between 100% and your SLO — becomes an explicit, data-driven rule for when a team should keep shipping features versus pause and focus on reliability, rather than that being a judgment call. SRE also brings toil reduction and blameless postmortems as named practices. In practice, most orgs — including where I've worked — run general DevOps culture as the baseline everywhere, and layer full SRE rigor, like formally defined SLOs and error budgets, specifically onto the handful of services where reliability is business-critical enough to justify that overhead."*

---

#### Q2. What is SLA and SLO?

**Answer:**

These two get mixed up constantly, and the single most important thing to say is: **they're usually not the same number, on purpose.** To explain why, I have to start one level below both of them, with **SLI**, since SLO and SLA are both built on top of it.

---

**SLI — Service Level Indicator: the actual measurement**

An SLI is a real, measured metric about how the service is behaving — not a target, just a number you're actually tracking. Examples: the percentage of HTTP requests that returned a successful status code, the p99 request latency, the percentage of successful reads from a database. It's collected from real telemetry — CloudWatch metrics, Prometheus, application logs — it's the raw data everything else is built from.

---

**SLO — Service Level Objective: the internal target for that measurement**

An SLO is the **goal** a team sets for an SLI — the internal bar the team holds itself accountable to. For example: *"99.9% of requests will succeed, measured over a rolling 30-day window."* SLOs are:
- **Internal** — set by the engineering team, for the engineering team
- **Flexible** — can be adjusted as the team learns more about the service, without needing legal or contract involvement
- **The basis for the error budget** — `100% − SLO` is the amount of acceptable unreliability the team is allowed to "spend" before pausing feature work in favor of stability work (covered in Q1)

---

**SLA — Service Level Agreement: the external, often contractual promise**

An SLA is a **commitment made to customers**, usually with real consequences attached if it's breached — service credits, refunds, contractual penalties. It's typically:
- **External** — written for customers, sometimes literally part of a signed contract
- **Legally/financially binding** — breaching it has real consequences, not just an internal conversation
- **Precisely defined** — an SLA document usually spells out exactly how uptime/downtime is measured, and what's excluded (e.g., planned maintenance windows don't count against it)

---

**The critical detail — SLA is deliberately set looser than SLO**

This is the part I always make sure to state explicitly, because it's the actual point of having two separate numbers instead of one: the **SLO should be stricter (higher) than the SLA**, giving the team a buffer.

```
SLI (measured reality)  →  actual observed success rate, e.g. 99.94% this month

SLO (internal target)   →  99.95% — the team's own bar, tracked via error budget

SLA (external promise)  →  99.9% — what's actually promised to customers,
                             with financial consequences if missed
```

The gap between the SLO (99.95%) and the SLA (99.9%) is deliberate: by the time the service is actually at risk of breaching the customer-facing, financially-binding SLA, the team has **already** been getting internal error-budget warnings for a while, because the SLO threshold is stricter and gets crossed first. Without that buffer, the first sign of trouble would be a contractual breach — with it, the first sign of trouble is an internal alert with time to actually fix things before it becomes a customer-facing, financial problem.

---

**Side-by-side**

| | SLI | SLO | SLA |
|---|---|---|---|
| **What it is** | A measurement | An internal target | An external promise |
| **Audience** | Engineering (raw data) | Engineering team | Customers / contract |
| **Binding?** | N/A — just a number | No — internal accountability only | Yes — often financial/legal consequences |
| **Typical strictness** | N/A | Stricter (higher) | Looser (lower) — gives the SLO room to catch problems first |
| **Who can change it** | Whoever owns the metric | The engineering/SRE team | Usually requires a contract/legal change |

---

**Real-world example — CloudCart**

For CloudCart's order-processing service, all three exist as actual, distinct numbers, not interchangeable ones:

- **SLI**: the real, measured request success rate, tracked via ALB/CloudWatch metrics and fed into a Grafana dashboard
- **SLO**: **99.95%** success rate over a rolling 30-day window, plus p99 latency under 300ms — this is the number the team is held to internally, and it's what the error budget policy from Q1 is actually calculated against
- **SLA**: **99.9%** uptime, written into the contract with enterprise customers, with a defined service-credit schedule if breached — measured with an explicit, contractually-defined methodology (and explicit exclusions for scheduled maintenance windows announced in advance)

That 0.05-percentage-point gap between the SLO and SLA looks small on paper, but at real request volume it represents a meaningful amount of "advance warning" time — the team sees error-budget pressure building internally well before there's ever a real risk of triggering the customer-facing SLA's financial consequences. Internal tooling and dashboards are all built against the SLO, not the SLA — the SLA is what Legal and Sales reference in customer conversations and contract renewals, not something engineers look at day to day.

---

**Complete thought process — how I approach this in the interview**

```
Three layers, not two:

SLI  → the raw, measured number (what's actually happening)
SLO  → the internal target for that number (what the team is held to)
SLA  → the external, often contractual promise (what customers are told)

The key point to state explicitly: SLO is deliberately STRICTER
than SLA, not the same number —
  → gives the team a buffer/early-warning zone
  → by the time the SLA is at real risk of breach, the stricter SLO
    has already been flagging trouble internally, via the error
    budget, for a while

Who owns each, and how easy is it to change?
  → SLI: whoever owns the metric/dashboard
  → SLO: engineering/SRE team — can adjust as understanding improves
  → SLA: usually requires a legal/contract change — much higher bar
    to change than an SLO
```

---

**Summary (what to say if time is short):**

*"SLI is the actual measured metric — like request success rate. SLO is the internal target the team sets for that metric, like 99.95%, and it's what the error budget is calculated against. SLA is the external, often contractual promise made to customers, usually with financial consequences if it's breached. The detail I'd make sure to state clearly is that the SLO is deliberately set stricter than the SLA, not the same number — so the team gets internal warning, through the error budget, well before there's any real risk of breaching the customer-facing, legally binding SLA. At CloudCart, for example, our internal SLO for order processing is 99.95%, while the SLA we actually commit to customers is 99.9% — that gap is the safety buffer, and it's why engineering dashboards are built against the SLO, while the SLA is what shows up in the actual customer contract."*

---

#### Q3. What is CI/CD? Explain briefly.

**Answer:**

Since this was asked as a "briefly" question, I'd keep the answer tight rather than over-explaining — a good sign of seniority is knowing when to give the short version.

**CI — Continuous Integration:** developers merge code into a shared repository frequently (multiple times a day), and every merge automatically triggers a build and an automated test run. The point is catching integration bugs immediately, while the change is small and fresh, instead of discovering them weeks later when many people's changes collide at once.

**CD — Continuous Delivery vs. Continuous Deployment** (the same acronym, two different things — worth clarifying, since this distinction is a common follow-up):
- **Continuous Delivery** — every change that passes CI is automatically built, tested, and packaged into a release-ready artifact, but the actual push to production is a **manual, human-triggered** step.
- **Continuous Deployment** — goes one step further: every change that passes all automated checks deploys to production **automatically, with no human step at all.**

**A typical pipeline, end to end:**
```
Code push/PR → Build → Automated tests → Package (e.g., Docker image) → Deploy to staging → (approval gate, if Delivery) → Deploy to production
```

**Common tools:** GitHub Actions, GitLab CI, Jenkins, Azure DevOps Pipelines — for the CI/build/test side; Argo CD (GitOps-style) or the same pipeline tool — for the CD/deploy side.

**Why it matters:** smaller, more frequent changes are easier to test and safer to roll back than large, infrequent releases — CI/CD is what makes shipping small and often practical instead of risky.

**Summary (what to say if time is short):**

*"CI is automatically building and testing every code change as soon as it's merged, to catch problems early. CD usually means Continuous Delivery — every passing change is automatically packaged and ready to release, but deploying to production is still a manual trigger — versus Continuous Deployment, which removes that manual step entirely and deploys automatically. Together, they're what make small, frequent releases practical and low-risk instead of infrequent, large, and scary."*

---

#### Q4. A developer has written only source code — design the CI/CD pipeline to deploy it to DEV/QA/PROD using best practices

**Answer:**

Starting from "just source code" means I get to design the whole thing deliberately rather than retrofit best practices onto something already built. I'd anchor the whole design around **one principle above all others: build the artifact exactly once, and promote that same, unmodified artifact through every environment** — never rebuild per environment. Almost every other best practice here either supports that principle or builds on top of it.

---

**Foundational decisions before writing any pipeline config**

- **Branching strategy**: trunk-based development — short-lived feature branches, frequent merges to `main` via PR — rather than long-lived GitFlow-style branches, unless there's a specific reason (like supporting multiple simultaneously-released versions) that justifies the extra complexity.
- **Immutable, traceable artifacts**: every built container image is tagged with the **git commit SHA**, never `latest` or a mutable tag, for anything that actually gets deployed. This means you can always answer "what code, exactly, is running in PROD right now" with certainty.
- **Environment parity**: DEV, QA, and PROD should differ only in **scale and specific config values** (endpoints, secrets) — not in deployment mechanism. If QA gets deployed a different way than PROD, QA never actually validates the deployment process itself — a very common, very avoidable gap.

---

**The pipeline, stage by stage**

```
PR opened
  → Lint + SAST + dependency vulnerability scan + unit tests
    (fast feedback, required checks before merge)

Merge to main
  → Build container image, tag with git commit SHA
  → Scan the built image (Trivy / ECR scanning) — fail on critical/high
  → Push to ECR

  → Deploy to DEV automatically (fastest feedback loop)
  → Run integration/E2E tests against the live DEV environment

  → Promote the SAME image (same SHA) to QA
  → Manual/exploratory QA testing happens here

  → Manual approval gate
  → Promote the SAME image (same SHA) to PROD
    (rolling update / canary, health-checked)
  → Automated smoke test + monitoring bake period
  → Rollback path: redeploy the previous known-good SHA
```

**A few of these stages deserve more explanation:**

- **PR-stage checks (lint, SAST, dependency scanning, unit tests)** run *before* merge, as required checks on the PR — this is where security scanning belongs, not bolted on later. A known-vulnerable dependency should block a merge, not get discovered after it's already in PROD.
- **The image is only built once**, right after merge to `main` — this is the moment the immutable artifact is created. Everything downstream (DEV, QA, PROD) deploys that exact image, identified by its SHA tag, never rebuilding it.
- **DEV deploys automatically**, with no approval gate — it's the fastest-feedback environment, and the cost of a bad deploy there is low.
- **QA gets the same image**, promoted forward, not rebuilt — this is what makes QA sign-off actually mean something: what QA tested is byte-for-byte what will reach PROD, not a similar-but-not-identical rebuild.
- **PROD requires an explicit, human approval gate** — a required reviewer on the deployment (e.g., GitHub Environments' protection rules), not just "tests passed, ship it." The deployment mechanism itself should still be a safe strategy — rolling update with health checks, or canary via a tool like Argo Rollouts for anything business-critical — same principle as reducing deployment downtime covered elsewhere.
- **Post-deploy verification and a bake period** — an automated smoke test immediately after deploy, plus watching key metrics for a defined window before considering the deploy "done." A rollback path (redeploy the previous SHA) needs to be a known, practiced action, not something invented under pressure during an incident.

---

**Where infrastructure and secrets fit in**

The **underlying infrastructure** (VPC, cluster, database, IAM roles) is provisioned through its **own separate Terraform pipeline**, not mixed into the application deploy pipeline — infra changes go through `terraform plan` visible in the PR, an approval, then `terraform apply`, following the same DEV→QA→PROD promotion philosophy, usually with even more caution on PROD infra changes given their blast radius.

For credentials: the CI/CD pipeline itself authenticates to AWS via **OIDC federation** — no long-lived AWS access keys stored as GitHub secrets. Application-level secrets (database passwords, API keys) are pulled from Secrets Manager at deploy or runtime, never baked into the image or passed through the pipeline as plaintext variables.

---

**Real-world example — CloudCart**

Early on, CloudCart's pipeline **rebuilt the image separately for each environment** — DEV, QA, and PROD each triggered their own `docker build`. This seemed harmless until a real incident: a feature that passed QA cleanly failed in PROD a few days later, with a confusing dependency-related error. The root cause was that a transitive dependency's patch version had been updated in the package registry between the QA build and the PROD build days later — same source code, genuinely different resolved dependency tree, because each environment ran its own independent build instead of promoting one artifact.

That incident is specifically why we moved to the single-build, promote-the-same-artifact-everywhere model described above — it's not a theoretical best practice, it's a direct fix for a real "works in QA, fails in PROD" failure we'd already had. Since then, the rule has been absolute: if it wasn't the exact image SHA that passed QA, it doesn't go to PROD, full stop.

---

**Complete thought process — how I approach this in the interview**

```
Everything anchors around ONE artifact, built once, promoted everywhere:

1. PR stage: lint, SAST, dependency scan, unit tests — required
   checks before merge, not an afterthought

2. Merge to main: build ONCE, tag with git commit SHA, scan the
   image, push to registry — this artifact is now immutable

3. DEV: auto-deploy, fastest feedback, lowest cost of failure

4. QA: promote the SAME artifact (never rebuild) — this is what
   makes QA sign-off meaningful

5. PROD: manual approval gate + safe deployment strategy (rolling/
   canary) + post-deploy smoke test + monitored bake period +
   a known, practiced rollback path

Supporting decisions:
  → Infra (Terraform) has its own pipeline, same promotion philosophy
  → CI/CD authenticates via OIDC, not stored AWS keys
  → Environment parity — QA should be deployed the same WAY as PROD,
    just at smaller scale with different config values
```

---

**Summary (what to say if time is short):**

*"The single most important principle is building the artifact exactly once and promoting that same image through DEV, QA, and PROD — never rebuilding per environment, since that's what actually guarantees what was tested in QA is what reaches PROD. Before merge, I'd run linting, SAST, dependency vulnerability scanning, and unit tests as required PR checks. On merge to main, the pipeline builds the image once, tags it with the git commit SHA, scans it for vulnerabilities, and pushes it to the registry. DEV deploys automatically for fast feedback; QA gets the identical image promoted forward, not rebuilt; and PROD requires an explicit manual approval gate, deployed with a safe rolling or canary strategy, followed by an automated smoke test and a monitored bake period, with a known rollback path ready rather than improvised. Infrastructure changes go through a separate Terraform pipeline with the same promotion philosophy, and the pipeline itself authenticates to AWS via OIDC rather than stored credentials. I'd mention this build-once-promote-everywhere rule isn't just theoretical — I've seen a 'works in QA, fails in PROD' incident caused by rebuilding per environment and picking up a different dependency version between builds, which is exactly the class of bug this principle eliminates."*

---

<!-- Add more scenario questions as Scenario 1, Scenario 2... -->

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
