# Jenkins — Real Interview Questions & Answers

> This file is a personal log of actual Jenkins questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Coforge | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. Multibranch Pipeline in Jenkins: how to configure it, what problems it solves, why it's preferred over a normal SCM pipeline](#q1-multibranch-pipeline-in-jenkins-how-to-configure-it-what-problems-it-solves-why-it-is-preferred-over-a-normal-scm-pipeline)

---

## Interview #1

**Company:** Coforge
**Date:** 22-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Senior DevOps Manager

---

### Questions Asked

#### Q1. Multibranch Pipeline in Jenkins: how to configure it, what problems it solves, why it is preferred over a normal SCM pipeline

**Answer:**

The core difference in one sentence: a **normal Pipeline job** (Pipeline script from SCM) is hardwired to **one specific branch** — someone has to manually create a new Jenkins job for every new branch that needs CI. A **Multibranch Pipeline** is a single job *definition* that automatically **discovers every branch (and optionally PRs/tags) in a repository** and creates, builds, and manages a sub-job for each one on its own — no manual job creation, ever.

---

**The problems it actually solves**

| Problem with a normal SCM Pipeline job | How Multibranch Pipeline fixes it |
|---|---|
| A new feature branch has no CI until someone remembers to manually create a Jenkins job for it | Auto-discovers any branch containing a `Jenkinsfile` and builds it automatically |
| Deleted/merged branches leave behind stale, cluttered Jenkins jobs nobody cleans up | **Orphaned Item Strategy** automatically prunes the job when the branch is deleted from source control |
| The pipeline definition lives outside the repo (pasted into Jenkins job config), so it can drift from what's actually in the branch | The `Jenkinsfile` lives **in the repository itself**, versioned per branch — a pipeline change on a feature branch is tested on that branch before merge |
| No native per-branch/PR build status visibility | Integrates with the SCM (GitHub/Bitbucket) to post build status directly as a **commit/PR check** |
| Doesn't scale with frequent branching (trunk-based development, lots of short-lived feature branches) | One job definition handles an arbitrary, changing number of branches with zero manual maintenance |

---

**How to configure it, step by step**

**1. Create the job** — New Item → **Multibranch Pipeline** (not "Pipeline").

**2. Add a Branch Source** — point it at the actual repository (GitHub, Bitbucket, or generic Git), with credentials scoped as narrowly as possible (a GitHub App or a PAT/SSH key with read access, not a broad admin credential).

**3. Configure Behaviours** — control exactly what gets discovered:
   - "Discover branches" — regular branches
   - "Discover pull requests from origin" / "from forks" — build PRs, not just branches
   - Filters to exclude branches matching a pattern (e.g., skip `dependabot/*` branches if not needed)

**4. Build Configuration → Mode: "by Jenkinsfile"** — specify the path (default is `Jenkinsfile` at the repo root). This is the key design point: Jenkins doesn't need the pipeline steps configured in its own UI at all — it reads them from the file living in whichever branch it's currently building.

**5. Scan Multibranch Pipeline Triggers** — either:
   - **Periodic scanning** — Jenkins polls the SCM every N minutes looking for new/removed branches, or
   - **Webhook-triggered** (preferred) — GitHub/Bitbucket pushes an event to Jenkins the moment a branch is created, updated, or deleted, so discovery is near-instant instead of waiting for the next poll

**6. Orphaned Item Strategy** — how long to keep a branch's job/build history after the branch itself is deleted from source control, before Jenkins prunes it — e.g., discard immediately, or retain for a few days so build logs are still available for a post-mortem shortly after a branch is deleted.

**7. Write the `Jenkinsfile` with branch-conditional stages** — since one pipeline definition now runs across every branch, `when` conditions differentiate behavior by branch:

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'docker build -t myapp:${GIT_COMMIT} .' }
        }
        stage('Test') {
            steps { sh 'run-tests.sh' }
        }
        stage('Deploy to DEV') {
            when { not { branch 'main' } }
            steps { sh 'deploy.sh dev' }
        }
        stage('Deploy to PROD') {
            when { branch 'main' }
            steps { sh 'deploy.sh prod' }
        }
    }
}
```

Every branch runs Build and Test identically; only `main` reaches the Deploy-to-PROD stage — one file, consistent logic, branch-aware behavior.

---

**Why it's preferred over a normal SCM Pipeline — the direct comparison**

| | Normal Pipeline (Pipeline script from SCM) | Multibranch Pipeline |
|---|---|---|
| **Branches covered** | One, fixed at job creation | All (or a filtered subset), auto-discovered |
| **New branch needs CI** | Manually create a new Jenkins job | Automatic — no action needed |
| **Deleted branch cleanup** | Manual | Automatic (Orphaned Item Strategy) |
| **Pipeline definition source** | Can be pasted into Jenkins UI, or from SCM for the one tracked branch | Always from the `Jenkinsfile` in each branch itself |
| **PR build status** | Not native | Native — posts directly as a commit/PR check |
| **Scales with trunk-based dev (many short-lived branches)** | Poorly | Naturally |

---

**Real-world example — CloudCart**

CloudCart originally ran individual Jenkins Pipeline jobs, one per long-lived branch, back when the team followed a more GitFlow-style workflow. Two real problems came out of that as the team moved toward trunk-based development with many short-lived feature branches: first, an engineer opened a feature branch, nobody created a corresponding Jenkins job for it, and the branch got merged to `main` having **never run CI at all** — caught only because a teammate happened to notice the missing build history during review, not because anything actually blocked it. Second, the Jenkins dashboard had accumulated **dozens of stale jobs** for branches that had been merged and deleted months earlier, making it genuinely hard for new team members to tell which jobs were still relevant.

Switching to a **Multibranch Pipeline** fixed both directly: every new branch with a `Jenkinsfile` gets CI automatically, with zero chance of "forgot to create the job," and the Orphaned Item Strategy prunes a branch's job automatically a few days after it's deleted from GitHub — the dashboard now only ever shows branches that actually still exist.

---

**Complete thought process — how I approach this in the interview**

```
Normal Pipeline job = 1 job, hardwired to 1 branch, manually created
Multibranch Pipeline = 1 job DEFINITION, auto-discovers N branches

Problems it solves:
  → No manual job creation per new branch
  → No manual cleanup of dead branches' jobs (Orphaned Item Strategy)
  → Pipeline definition lives IN the repo (Jenkinsfile), versioned
    per branch, not pasted into Jenkins UI
  → Native PR/commit status integration
  → Scales naturally with trunk-based development / frequent branching

Configuration, in order:
  1. New Item → Multibranch Pipeline
  2. Branch Source (repo + scoped credentials)
  3. Behaviours (which branches/PRs/tags to discover)
  4. Build Configuration → by Jenkinsfile
  5. Scan trigger — webhook preferred over periodic polling
  6. Orphaned Item Strategy — how long to retain a deleted branch's job
  7. Jenkinsfile itself uses `when { branch '...' }` for branch-aware stages
```

---

**Summary (what to say if time is short):**

*"A normal Pipeline job is tied to one specific branch — someone has to manually create a new job for every branch that needs CI, and manually clean up jobs for branches that get deleted. A Multibranch Pipeline is one job definition that automatically discovers every branch, or PR, in a repository containing a Jenkinsfile, builds each one, and automatically prunes the job once its branch is deleted, via the Orphaned Item Strategy. To configure it: create a Multibranch Pipeline job, point it at the repo with scoped credentials, configure which branches/PRs to discover, set the build configuration to read from the Jenkinsfile in each branch rather than a UI-configured pipeline, and set up a webhook so new branches are picked up almost instantly instead of waiting for a periodic scan. It's preferred because it eliminates manual job sprawl entirely, keeps the pipeline definition versioned alongside the code it builds, and scales naturally with a trunk-based workflow that has a lot of short-lived feature branches — which is exactly the kind of gap that bit us directly at CloudCart, when a feature branch got merged having never run CI at all, simply because nobody had manually created a job for it."*

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
