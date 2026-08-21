# AWS — Real Interview Questions & Answers

> This file is a personal log of actual AWS questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Accenture | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. You have created an IAM user in AWS and configured role-based access in EKS. How do you bind the IAM user to the EKS role?](#q1-you-have-created-an-iam-user-in-aws-and-configured-role-based-access-in-eks-how-do-you-bind-the-iam-user-to-the-eks-role)
  - [Q2. Assume you have 10 AWS accounts. How will you securely log in to them, considering access keys are not used for security reasons?](#q2-assume-you-have-10-aws-accounts-how-will-you-securely-log-in-to-them-considering-access-keys-are-not-used-for-security-reasons)
  - [Q3. What are the ways to log in to an AWS account?](#q3-what-are-the-ways-to-log-in-to-an-aws-account)
  - [Q4. Does Amazon S3 require a VPC?](#q4-does-amazon-s3-require-a-vpc)
  - [Q5. If the frontend, backend, and database are all deployed in private subnets, how can an end user access the application?](#q5-if-the-frontend-backend-and-database-are-all-deployed-in-private-subnets-how-can-an-end-user-access-the-application)
  - [Q6. If secrets are created in AWS Secrets Manager, how can Amazon EKS access those secrets?](#q6-if-secrets-are-created-in-aws-secrets-manager-how-can-amazon-eks-access-those-secrets)
  - [Q7. How do you set up RBAC in Amazon EKS?](#q7-how-do-you-set-up-rbac-in-amazon-eks)

---

## Interview #1

**Company:** Accenture
**Date:** 05-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Senior DevOps Manager

---

### Questions Asked

#### Q1. You have created an IAM user in AWS and configured role-based access in EKS. How do you bind the IAM user to the EKS role?

**Answer:**

This question is really checking one thing: do you understand that **AWS IAM** and **Kubernetes RBAC** are two completely separate authorization systems, and that EKS sits in the middle translating one into the other? A lot of people who've only worked with EKS at a surface level get tripped up here because they think attaching an IAM policy to the user is enough — it isn't. Let me walk through it the way I'd actually explain it on the whiteboard.

---

**The core concept — two systems, two questions**

Every request you make to the Kubernetes API server (via `kubectl`) has to answer two separate questions, handled by two separate systems:

| Question | Who answers it | Mechanism |
|---|---|---|
| **"Who are you?"** (Authentication) | AWS IAM | The AWS IAM Authenticator built into the EKS control plane |
| **"What are you allowed to do?"** (Authorization) | Kubernetes RBAC | Native `Role` / `ClusterRole` + `RoleBinding` / `ClusterRoleBinding` |

An IAM user, on its own, has **zero permissions inside the cluster** — even if it has `AdministratorAccess` in AWS. IAM only gets you through the front door and tells Kubernetes "this is `arn:aws:iam::111122223333:user/john`." What John can actually do once he's inside — list pods, delete deployments, read secrets — is 100% controlled by Kubernetes RBAC. This is the single most important thing to say in the interview: **IAM authenticates, RBAC authorizes, and something has to sit between them to map one identity to the other.**

That "something" is either the `aws-auth` ConfigMap (the original mechanism) or **EKS Access Entries** (the modern replacement AWS introduced in 2023). I'll explain both because interviewers often ask "what about older clusters?" as a follow-up.

---

**Method 1 — The `aws-auth` ConfigMap (classic method, still very common in existing clusters)**

Every EKS cluster used to ship with a ConfigMap called `aws-auth` in the `kube-system` namespace. This ConfigMap is the mapping table between IAM identities and Kubernetes usernames/groups.

```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::111122223333:role/eks-node-group-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::111122223333:user/john
      username: john
      groups:
        - eks-readonly-group
```

What this does:
- `userarn` — the exact IAM user ARN (this must match the identity that shows up in `aws sts get-caller-identity`)
- `username` — the Kubernetes-side identity this IAM user is translated into
- `groups` — the Kubernetes RBAC group(s) this user is placed into — **this is the piece that actually connects to RBAC permissions**

You edit it directly or, more safely, through `eksctl`:

```bash
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::111122223333:user/john \
  --username john \
  --group eks-readonly-group
```

I always recommend `eksctl` over manual `kubectl edit configmap/aws-auth` — a bad edit to that ConfigMap can lock every IAM identity, including your own, out of the cluster at once. That's a genuinely painful outage to recover from since you then need to use the cluster creator's root credentials to fix it.

---

**Method 2 — EKS Access Entries (modern method, AWS's current recommendation)**

Since EKS added the **Access Entries API** (available on clusters running platform version supporting it, GA since late 2023), you no longer touch the `aws-auth` ConfigMap at all. It's a first-class EKS API, managed entirely through `aws eks` CLI commands or Terraform, and it's what I'd push for on any new cluster.

```bash
# Step 1 — Create the access entry, linking the IAM user to the cluster
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::111122223333:user/john \
  --type STANDARD \
  --kubernetes-groups eks-readonly-group
```

```bash
# Step 2 — (Optional) Attach an AWS-managed access policy directly,
# instead of relying purely on a Kubernetes RBAC group
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::111122223333:user/john \
  --access-scope type=namespace,namespaces=staging \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy
```

The advantage here is huge from an operations standpoint: this is a proper AWS API call, so it shows up in **CloudTrail**, can be provisioned through **Terraform** (`aws_eks_access_entry` resource), and a bad entry only affects that one principal — it can't accidentally wipe out access for the entire cluster the way a botched ConfigMap edit can.

---

**Step-by-step — binding the IAM user to a working EKS role, end to end**

Whichever method you use for the identity mapping, the full flow to actually give John working access looks like this:

**Step 1 — IAM side: make sure the user can reach the cluster at all**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:us-east-1:111122223333:cluster/my-cluster"
    }
  ]
}
```

Attach this to the IAM user (or, better, to a Role the user assumes). This does **not** grant Kubernetes permissions — it only allows `aws eks update-kubeconfig` to succeed and lets the IAM Authenticator verify the caller's identity via a signed `sts:GetCallerIdentity` request.

**Step 2 — Map the IAM identity to a Kubernetes group** (Method 1 or Method 2 above)

This is the step people skip and then can't figure out why `kubectl` returns `error: You must be logged in to the server (Unauthorized)`.

**Step 3 — Kubernetes side: create the Role and RoleBinding that actually grants permissions**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: readonly-role
  namespace: staging
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: staging
subjects:
  - kind: Group
    name: eks-readonly-group        # must match the "groups" value from Step 2 exactly
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: rbac.authorization.k8s.io
```

The `subjects.name` here (`eks-readonly-group`) has to be an **exact string match** to the group you mapped the IAM user into back in Step 2. This is the actual "binding" the question is asking about — it's not an AWS-side operation at all, it's a Kubernetes RBAC object. That's the part that trips people up: they go looking for the "bind" step in IAM, but it's really half in IAM (identity mapping) and half in native Kubernetes (RoleBinding).

**Step 4 — Verify**

```bash
# John configures kubectl using his own IAM credentials
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Confirm which IAM identity is being used
aws sts get-caller-identity

# Confirm what Kubernetes thinks John can do
kubectl auth can-i get pods -n staging --as=john
# yes

kubectl auth can-i delete deployments -n staging --as=john
# no
```

`kubectl auth can-i ... --as=<username>` is the fastest way to sanity-check an RBAC binding without needing the actual user's credentials in front of you.

---

**Real-world example — CloudCart**

At CloudCart (our AWS-hosted e-commerce platform), we onboarded a new QA engineer, Priya, who needed to read pod logs in the `staging` namespace to debug failing test runs — nothing else. She should not be able to touch `production`, and she should not be able to edit or delete anything even in `staging`.

Here's exactly what we did:

1. Created her IAM user (`priya`) with only `eks:DescribeCluster` attached — no EKS or EC2 admin permissions.
2. Since we were already on the newer platform version, we used an **Access Entry** instead of touching `aws-auth`:
   ```bash
   aws eks create-access-entry \
     --cluster-name cloudcart-prod \
     --principal-arn arn:aws:iam::444455556666:user/priya \
     --kubernetes-groups qa-readonly-group
   ```
3. We already had a `Role` + `RoleBinding` in the `staging` namespace scoped to `get`, `list`, `watch` on `pods`, `pods/log`, and `events` — bound to the group `qa-readonly-group`. We didn't need to create a new one; Priya just needed to join that existing group.
4. She ran `aws eks update-kubeconfig` and confirmed with `kubectl auth can-i get pods -n staging` → `yes`, and `kubectl auth can-i get pods -n production` → `no`.

The whole thing took about 10 minutes because the RBAC `Role`/`RoleBinding` already existed as a reusable "QA read-only" pattern — we only ever had to do the identity-mapping step for each new hire. That reusability is exactly why I'd recommend designing your RBAC roles as small, purpose-built, reusable groups (`qa-readonly-group`, `oncall-cluster-admin`, `ci-deployer`) rather than mapping individual users straight to permissions one at a time.

---

**Complete thought process — how I approach this in the interview**

```
"Bind IAM user to EKS role" really means two separate bindings:

1. IAM identity → Kubernetes identity (username/group)
   → Is this an older cluster?
       → Yes → edit aws-auth ConfigMap (prefer eksctl over manual edit)
       → No  → use `aws eks create-access-entry` (CloudTrail-logged, safer, Terraform-native)

2. Kubernetes group → Kubernetes permissions
   → Does a Role/RoleBinding already exist for this access level?
       → Yes → just add the user's mapped group as a subject
       → No  → create Role (namespace-scoped) or ClusterRole (cluster-scoped)
               + RoleBinding/ClusterRoleBinding

Verify with:
   aws sts get-caller-identity        (confirms WHO AWS thinks you are)
   kubectl auth can-i <verb> <resource> --as=<user>   (confirms WHAT k8s thinks you can do)
```

---

**Summary (what to say if time is short):**

*"An IAM user has no permissions inside an EKS cluster by default, even with full AWS access — IAM only handles authentication, not authorization. To bind them, I first map the IAM user's ARN to a Kubernetes username and group — either through the `aws-auth` ConfigMap on older clusters, or through the newer EKS Access Entries API, which I prefer because it's CloudTrail-logged and Terraform-manageable instead of being a shared ConfigMap everyone edits. Then, on the Kubernetes side, I create a Role or ClusterRole defining the actual permissions, and a RoleBinding or ClusterRoleBinding that ties that Role to the same group name I mapped the IAM user into. So the 'binding' is really two links in a chain — IAM identity to Kubernetes group, and Kubernetes group to Kubernetes permissions — and I verify both ends with `aws sts get-caller-identity` and `kubectl auth can-i --as=<user>`."*

---

#### Q2. Assume you have 10 AWS accounts. How will you securely log in to them, considering access keys are not used for security reasons?

**Answer:**

This is a multi-account access management question, and it's really testing whether you know the industry-standard replacement for "one IAM user with access keys per account" — which is exactly the anti-pattern the question is ruling out. The moment someone says "access keys are off the table," the answer they're fishing for is **federated, temporary, credential-based access through AWS Organizations + IAM Identity Center**, not ten separate logins. Let me build it up the way I would on a whiteboard.

---

**Why access keys are the wrong tool here in the first place**

Access keys are long-lived, static secrets. If you have 10 accounts and give every engineer an IAM user + access key pair in each one, you immediately have a few structural problems:

| Problem | Why it's dangerous |
|---|---|
| **Key sprawl** | 10 accounts × N engineers = potentially hundreds of static key pairs to rotate, track, and revoke |
| **No expiry** | Access keys don't expire on their own — a leaked key from 2 years ago still works unless someone manually rotated it |
| **No central kill switch** | To offboard one engineer, you have to hunt down and delete their key in all 10 accounts individually |
| **Hard to audit centrally** | Each account logs `AccessKeyId` usage separately — there's no single place to see "everything this person did across all 10 accounts today" |
| **Easy to leak** | Static keys get hardcoded into scripts, committed to git, baked into AMIs — this is one of the most common real-world breach vectors (see the Capital One and Uber incidents) |

The fix for all five of these at once is to stop distributing long-lived secrets entirely and instead give people **temporary, auto-expiring credentials issued on demand** from a single, central login.

---

**The architecture — AWS Organizations + IAM Identity Center**

```
                     ┌─────────────────────────┐
                     │   Identity Provider       │
                     │  (Okta / Azure AD / etc.) │
                     └────────────┬──────────────┘
                                  │ SAML 2.0 federation + MFA
                                  ▼
                     ┌─────────────────────────┐
                     │   IAM Identity Center      │
                     │  (management account)     │
                     │  — Permission Sets defined │
                     │    once, centrally         │
                     └────────────┬──────────────┘
                                  │ auto-provisions IAM roles
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
        ┌───────────┐       ┌───────────┐       ┌───────────┐
        │ Account 1 │  ...  │ Account 5 │  ...  │ Account 10│
        │  (dev)    │       │ (staging) │       │  (prod)   │
        └───────────┘       └───────────┘       └───────────┘
```

The pieces:

1. **AWS Organizations** — all 10 accounts sit under one Organization with a management account. This is a prerequisite — Identity Center needs an Organization to provision roles into every member account automatically.
2. **IAM Identity Center** (formerly AWS SSO) — a single, centralized login. Enabled once, in the management account (or a delegated admin account, which is the more secure pattern so the management account itself isn't a daily-use login target).
3. **Identity source** — either Identity Center's built-in directory, or federated to a real corporate IdP (Okta, Azure AD, Google Workspace) via SAML 2.0. In any real company with 10 accounts, you federate to the existing corporate IdP so account access rides on the same MFA and offboarding process as email/Slack.
4. **Permission Sets** — defined once, centrally (e.g., `DevOpsEngineerAccess`, `ReadOnlyAccess`, `BillingViewer`). A permission set is just a named bundle of IAM policies. Identity Center automatically provisions a matching IAM role (like `AWSReservedSSO_DevOpsEngineerAccess_a1b2c3`) into each account it's assigned to — you never manually create these roles yourself.
5. **Account assignment** — you assign (user or group) + (account) + (permission set). Example: "DevOps team gets `DevOpsEngineerAccess` in accounts 1–7 (dev/staging), but only `ReadOnlyAccess` in accounts 8–10 (prod)."

---

**What logging in actually looks like day to day**

**Console access:**
1. Engineer goes to the company's SSO portal: `https://my-company.awsapps.com/start`
2. Authenticates with corporate credentials + MFA (handled entirely by the IdP — Okta push notification, etc.)
3. Sees a tile for every (account, role) combination they've been assigned — all 10 accounts, or however many they have access to, in one screen
4. Clicks "Account 3 → DevOpsEngineerAccess" → gets an AWS Console session for that account, backed by **temporary STS credentials** valid for a limited session duration (commonly 1–12 hours, configurable per permission set)

**CLI access — no keys anywhere on disk:**
```bash
# One-time setup per profile
aws configure sso
# SSO start URL: https://my-company.awsapps.com/start
# SSO region: us-east-1
# SSO account ID: 222233334444        (pick from the list it shows you)
# SSO role name: DevOpsEngineerAccess
# CLI profile name: account3-devops
```

```bash
# Every day / whenever the cached token expires
aws sso login --profile account3-devops
# Opens a browser, you authenticate via the IdP + MFA once

aws sts get-caller-identity --profile account3-devops
# {
#   "UserId": "AROAABCDEFG:john@company.com",
#   "Account": "222233334444",
#   "Arn": "arn:aws:sts::222233334444:assumed-role/AWSReservedSSO_DevOpsEngineerAccess_.../john@company.com"
# }
```

Notice the ARN — it's `assumed-role`, not `user`. There is no static access key involved at any point. The CLI caches a short-lived SSO token under `~/.aws/sso/cache`, and it auto-refreshes temporary STS credentials from that token as needed. Switching between all 10 accounts is just switching `--profile`.

---

**Non-human / CI-CD access — the same principle, different mechanism**

The question is about human login, but interviewers almost always follow up with "what about pipelines?" — so I make sure to cover it. Pipelines shouldn't have static keys either. The equivalent for CI/CD is **OIDC federation**:

- GitHub Actions / GitLab CI has a built-in OIDC identity provider
- You register that OIDC provider as a trusted identity in each target AWS account
- The pipeline's IAM role trust policy allows `sts:AssumeRoleWithWebIdentity` only from that specific repo/branch
- Each pipeline run gets a short-lived token, scoped to exactly the one deployment it's doing — again, nothing long-lived stored as a GitHub secret

```yaml
# GitHub Actions example — no AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY anywhere
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::222233334444:role/github-actions-deploy-role
      aws-region: us-east-1
```

---

**Guardrails — enforcing this org-wide, not just relying on people to opt in**

Setting up Identity Center doesn't stop someone from also creating an IAM user with an access key the old way, unless you close that door explicitly. At the Organization level, I'd apply a **Service Control Policy (SCP)** that denies static credential creation for human roles across all 10 accounts:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAccessKeyCreationForHumans",
      "Effect": "Deny",
      "Action": [
        "iam:CreateAccessKey",
        "iam:CreateUser"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalTag/AllowIamUserCreation": "true"
        }
      }
    }
  ]
}
```

This SCP is attached once, at the Organization root (or an OU), and it applies to every one of the 10 accounts automatically — that's the real payoff of doing this through Organizations instead of account-by-account.

---

**Real-world example — CloudCart**

CloudCart grew from 2 AWS accounts to 10 over about a year — `dev`, `staging`, `prod`, `security`, `logging`, `shared-services`, and a few sandbox accounts per team. Early on, engineers genuinely did have IAM users with access keys in each account, and offboarding a contractor once meant manually checking and deleting keys across 6 different accounts — someone missed one, and that stale key sat active for another 4 months before a routine access review caught it.

After that, we:
1. Put all 10 accounts under one **AWS Organization**
2. Enabled **IAM Identity Center**, federated to our existing **Okta** tenant (same login/MFA employees already used for everything else)
3. Defined 4 permission sets: `ReadOnlyAccess`, `DevOpsEngineerAccess`, `BillingViewer`, `OrgAdmin` — and assigned them per team, per account
4. Rolled out an org-wide **SCP** denying `iam:CreateAccessKey` for anything without an explicit exception tag (used only for a couple of legacy service accounts we hadn't migrated yet)
5. Moved our GitHub Actions deploy pipelines to **OIDC role assumption**, removing every `AWS_SECRET_ACCESS_KEY` GitHub secret we had

Offboarding is now one action — disable the person in Okta — and their access to all 10 AWS accounts is revoked immediately, everywhere, at once. That single change (one action instead of ten) was the actual business justification the security team used to prioritize the migration.

---

**Complete thought process — how I approach this in the interview**

```
"No access keys, 10 accounts" → the answer is centralize the login, decentralize nothing else

Is this human interactive access?
  → Yes → AWS Organizations + IAM Identity Center
           → Federate to corporate IdP (SAML) for real companies
           → Define Permission Sets once, assign per (account, group)
           → Users get temporary STS creds via SSO portal or `aws sso login`

Is this pipeline / non-human access?
  → Yes → OIDC federation (GitHub Actions / GitLab CI / etc.)
           → IAM role trust policy scoped to specific repo/branch
           → No secrets stored in the CI system at all

Legacy environment, no Identity Center yet?
  → Central federated identity (or a single tightly-controlled IAM user with
    mandatory MFA) + per-account IAM roles with sts:AssumeRole,
    trust policy requiring aws:MultiFactorAuthPresent

How do I stop people from going around this?
  → SCP at the Organization root denying iam:CreateAccessKey / iam:CreateUser
    for human principals, enforced across all 10 accounts at once
```

---

**Summary (what to say if time is short):**

*"I wouldn't give anyone standing access keys in any of the 10 accounts. Instead I'd put all 10 under one AWS Organization and turn on IAM Identity Center as a single, central login, federated to our corporate IdP so it inherits the same MFA and offboarding process as everything else. I'd define Permission Sets once — like ReadOnlyAccess or DevOpsEngineerAccess — and Identity Center automatically provisions the matching IAM role into whichever accounts I assign them to. Day to day, engineers log into one SSO portal, pick an account and role, and get temporary STS credentials that expire on their own — nothing long-lived to leak or rotate. For CI/CD pipelines I'd use OIDC federation the same way, so even automated deployments never touch a static secret. And I'd back all of it with a Service Control Policy at the Organization root that denies `iam:CreateAccessKey` for human principals, so the secure path is the only path, not just the recommended one."*

---

#### Q3. What are the ways to log in to an AWS account?

**Answer:**

This question usually comes right after the "10 accounts, no access keys" question, as a way to check whether you actually know the full landscape or just memorized one answer. "Logging in" to AWS isn't one thing — it spans a human clicking a button in a browser all the way to a Lambda function silently picking up credentials with no human involved at all. I always structure my answer around one axis: **who or what is authenticating** — a person, or a workload — because that's what determines which method is correct.

---

**The full list, at a glance**

| # | Method | Who/what uses it | Credential type | Typical use case |
|---|---|---|---|---|
| 1 | **Root user login** | A human (rarely) | Email + password (+ MFA) | Account creation, billing emergencies, closing the account — nothing else |
| 2 | **IAM user — console password** | A human | Username + password (+ MFA) | Legacy direct login — being phased out in favor of #5 |
| 3 | **IAM user — access keys** | A human or script | Long-lived `AccessKeyId` / `SecretAccessKey` | CLI/API — the pattern most companies are actively moving away from |
| 4 | **IAM Role (AssumeRole via STS)** | Another AWS identity, EC2, ECS, Lambda | Temporary STS credentials (minutes–hours) | Cross-account access, workloads running inside AWS |
| 5 | **IAM Identity Center (AWS SSO)** | A human | Temporary STS credentials, federated | Centralized human login across one or many accounts |
| 6 | **SAML 2.0 federation (direct)** | A human, via corporate IdP | Temporary STS credentials | Enterprise login without IAM Identity Center in the middle |
| 7 | **Web Identity Federation (OIDC)** | An app end-user, or a CI/CD pipeline | Temporary STS credentials | Mobile/web app users (Cognito), GitHub Actions/GitLab CI |
| 8 | **MFA** | Layered on top of #1, #2, #5, #6 | Not a login method by itself | Additional required factor for privileged actions |

I'll go through each briefly, because that's usually exactly what the interviewer wants — a fast, structured pass through all of them, not an essay on each one.

---

**1. Root user**

The identity created the moment you sign up for AWS. It bypasses every IAM policy — there is nothing you can restrict it with. In any properly run account, the root user is used exactly three times: account creation, setting up the first IAM Identity Center admin, and (rarely) emergency billing access. It should have a hardware MFA device attached, no access keys ever generated for it, and nobody should know its password day-to-day.

**2 & 3. IAM users — password and access keys**

An IAM user is a named identity inside one account. It can have a console password (for browser login) and/or access keys (for CLI/SDK/API calls) — both are **long-lived, static credentials**. This is the classic method everyone starts with, and it's exactly the anti-pattern discussed in Q2 — no built-in expiry, hard to rotate at scale, easy to leak. Still valid for a handful of legacy or third-party integrations that genuinely can't do federation, but not the default for humans anymore.

**4. IAM Role via AssumeRole (STS)**

A role has no long-term credentials of its own — it's assumed by something else, which then receives **temporary security credentials** from AWS STS (`AccessKeyId` + `SecretAccessKey` + `SessionToken`, expiring in as little as 15 minutes or up to 12 hours). This is the backbone almost every other method on this list is built on top of. Common shapes:

```bash
# Human or another account assuming a role directly
aws sts assume-role \
  --role-arn arn:aws:iam::222233334444:role/DeployRole \
  --role-session-name my-session \
  --serial-number arn:aws:iam::111122223333:mfa/john \
  --token-code 123456
```

```yaml
# EC2 instance profile — the instance IS the identity, no keys involved
IamInstanceProfile:
  Arn: arn:aws:iam::111122223333:instance-profile/app-server-role
```

```yaml
# Lambda execution role — same idea, credentials injected automatically
Role: arn:aws:iam::111122223333:role/lambda-execution-role
```

Anything running *inside* AWS — EC2, ECS tasks, Lambda, CodeBuild — should be using an attached role, never keys baked into the environment.

**5. IAM Identity Center (AWS SSO)**

Covered in depth in Q2 — a single, centralized login for humans, federated to a corporate IdP, that issues temporary STS credentials per session across as many accounts as the person is assigned to. This is AWS's current recommended default for human access.

**6. SAML 2.0 federation (direct, without Identity Center)**

Before Identity Center existed, companies federated directly: the corporate IdP (ADFS, Okta, PingFederate) authenticates the user and hands back a SAML assertion, which is exchanged for temporary credentials via `sts:AssumeRoleWithSAML`. You still see this in older environments or where a company runs its own custom federation broker instead of using Identity Center.

```bash
aws sts assume-role-with-saml \
  --role-arn arn:aws:iam::111122223333:role/SAMLFederatedRole \
  --principal-arn arn:aws:iam::111122223333:saml-provider/CorpADFS \
  --saml-assertion file://saml-assertion.xml
```

**7. Web Identity Federation (OIDC)**

Same underlying idea as SAML federation, but for OIDC/OAuth2 tokens instead of SAML assertions. Two very different audiences use this in practice:
- **App end-users** — a mobile or web app lets users sign in with Google/Facebook/Apple, and uses **Amazon Cognito Identity Pools** to exchange that token for temporary, scoped AWS credentials — so the app never embeds an AWS secret at all.
- **CI/CD pipelines** — GitHub Actions and GitLab CI expose a built-in OIDC token for each pipeline run. The AWS IAM role trust policy is scoped to a specific repo/branch, and `sts:AssumeRoleWithWebIdentity` issues a short-lived credential for that one run only. This is the method from Q2's CI/CD section.

```yaml
# GitHub Actions — no stored AWS secrets, token is minted per run
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::111122223333:role/github-actions-deploy-role
      aws-region: us-east-1
```

**8. MFA**

Not a login method on its own — a second factor layered on top of #1, #2, #5, or #6. Can be a hardware key (YubiKey), a virtual authenticator app (TOTP), or an IdP-driven push (Okta Verify, Duo). AWS also supports conditionally *requiring* MFA before certain actions are allowed, regardless of how the session started:

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
  }
}
```

---

**Real-world example — CloudCart**

CloudCart's account access, laid out by identity type, looks like this — I think this is the cleanest way to show an interviewer you understand when each method actually applies:

| Identity | Method used | Why |
|---|---|---|
| Root user (each of the 10 accounts) | Root login, hardware MFA, password in a vault | Emergency-only, essentially never used |
| Engineers, QA, support staff | IAM Identity Center, federated to Okta | Centralized, temporary, easy offboarding (Q2) |
| A partner agency's contractors | Direct SAML federation from the partner's own IdP | They're not in our Okta tenant — federated separately with a scoped role and a short session duration |
| EC2 app servers, ECS tasks | Instance profiles / task roles | Workload identity, zero stored credentials |
| Lambda functions | Execution roles | Same — injected automatically by the Lambda service |
| GitHub Actions deploy pipelines | OIDC web identity federation | No `AWS_SECRET_ACCESS_KEY` GitHub secret exists anywhere in the org |
| Mobile app users | Cognito Identity Pool (Google/Apple sign-in) | End users never see or touch an AWS credential |
| One legacy billing-export script | IAM user with an access key | The one exception — flagged for migration, key is rotated every 90 days by a Lambda automation until it's replaced |

That last row is deliberate — I like including it in the answer because it shows the interviewer I'm being realistic, not reciting a textbook. Real environments almost always have one or two legacy exceptions; the point is knowing they're exceptions, tracking them, and having a plan to remove them — not pretending they don't exist.

---

**Complete thought process — how I approach this in the interview**

```
Who or what is trying to authenticate?

Is it a human?
  → Everyday engineering access?      → IAM Identity Center (federated, temporary)
  → Enterprise federation without
    Identity Center in the middle?    → Direct SAML 2.0 federation
  → Break-glass / account setup only? → Root user, hardware MFA, locked away
  → Legacy exception that can't
    federate yet?                     → IAM user password/keys — track it, plan to remove it

Is it a workload running inside AWS?
  → EC2 / ECS / Lambda / CodeBuild    → Instance profile / task role / execution role
  → Another AWS account               → Cross-account IAM role + sts:AssumeRole

Is it an external app's end user?
  → Web/mobile app sign-in            → Web Identity Federation via Cognito

Is it a CI/CD pipeline?
  → GitHub Actions / GitLab CI        → OIDC web identity federation, scoped to repo/branch

Whatever the answer — is MFA enforced on top of it where a human is involved?
```

---

**Summary (what to say if time is short):**

*"Broadly, there are eight ways to authenticate into AWS, and I group them by who or what is logging in. For humans, there's the root user — locked away, emergency-only — an IAM user with a console password or access keys, which is the legacy path, and the modern path, which is IAM Identity Center giving federated, temporary credentials, or direct SAML federation if there's no Identity Center in the middle. For workloads running inside AWS — EC2, ECS, Lambda — it's always an attached IAM role, assumed automatically, no stored keys. For an external app's end users, it's Web Identity Federation through Cognito. And for CI/CD pipelines, it's OIDC federation, the same idea as SAML but for tools like GitHub Actions. Every one of those, other than the legacy IAM user path, produces short-lived STS credentials instead of something static — and MFA sits on top wherever a human is involved."*

---

#### Q4. Does Amazon S3 require a VPC?

**Answer:**

Short answer first, because that's what a good interviewer wants to hear immediately: **No — S3 does not require a VPC, and it doesn't live inside one.** Then I explain the "why," because that's where the real signal is — this question is checking whether you understand the distinction between services AWS deploys *into* your VPC and services that sit *outside* it as public, regional endpoints.

---

**The core distinction — VPC-resident services vs. public AWS services**

AWS services fall into two broad categories from a networking standpoint:

| Category | Examples | How you reach it |
|---|---|---|
| **VPC-resident services** | EC2, RDS, ElastiCache, EKS worker nodes, Lambda-in-VPC | Deployed with an ENI inside your subnets — has a private IP, lives in an AZ, needs security groups/NACLs |
| **Public/regional AWS services** | **S3**, DynamoDB, SNS, SQS, IAM, CloudFront, Route 53 | Reached via a public HTTPS endpoint (`s3.<region>.amazonaws.com`) — not deployed into any customer's VPC at all |

S3 is firmly in the second category. There's no "launch S3 into this VPC" option because S3 was never a VPC construct to begin with — it predates VPC by several years and was built as a global-namespace, region-redundant object store reached over the public internet (or over AWS's private backbone), the same way any external client reaches it. An EC2 instance in a public subnet with an internet gateway can hit S3 today with zero extra configuration — same as your laptop can.

---

**So why does this question even come up? — because of *private* workloads**

The nuance the interviewer is actually testing is: what happens when your workload is inside a **private subnet with no internet access** (no NAT gateway, no internet gateway) and it still needs to read/write S3? This is genuinely common — private EKS worker nodes, private RDS-adjacent app servers, Lambda functions in a VPC — and it's where people get confused into thinking "S3 must need a VPC" when really the VPC is what's missing the *path out*, not S3 requiring a *path in*.

There are two ways to solve this, and the difference between them is a very common follow-up question:

**Option 1 — NAT Gateway**
Traffic from the private subnet routes out through a NAT Gateway in a public subnet, over the internet gateway, and hits S3's public endpoint. This works, but:
- Costs money — NAT Gateway has an hourly charge *plus* a per-GB data processing charge
- Traffic technically leaves your VPC and traverses the public internet (still encrypted over TLS, but it's not staying inside the AWS private network)

**Option 2 — VPC Gateway Endpoint for S3 (the better answer)**
A **Gateway Endpoint** adds a route in your subnet's route table that sends S3-bound traffic directly to S3 over AWS's internal network — never touching the public internet, and it's **free** (no hourly charge, no per-GB charge). This is the AWS-recommended pattern for any private subnet that needs S3 access.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-0abc123def456 \
  --vpc-endpoint-type Gateway
```

This just adds an entry to the specified route table:
```
Destination                    Target
pl-63a5400a (S3 prefix list)   vpce-0a1b2c3d4e5f
```
No ENI, no security group, no per-hour cost — it's purely a routing construct.

*(There's also an **Interface Endpoint** for S3, backed by AWS PrivateLink with an ENI and a private IP — used specifically for hybrid scenarios, like reaching S3 privately from an on-premises network over Direct Connect/VPN, since a Gateway Endpoint is only reachable from within the VPC itself.)*

---

**The other reason people reach for VPC endpoints — locking S3 down, not just reaching it**

Beyond cost and routing, there's a security angle: even with a Gateway Endpoint in place, S3 is still, by default, reachable from the public internet by anyone with valid credentials. If the requirement is "this bucket must ONLY ever be reachable from inside our VPC — not even with a leaked access key from someone's laptop" — that's enforced with a **bucket policy condition on `aws:sourceVpce`**, not by the existence of the VPC endpoint alone.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAccessOutsideThisVpcEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::cloudcart-internal-data",
        "arn:aws:s3:::cloudcart-internal-data/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0a1b2c3d4e5f"
        }
      }
    }
  ]
}
```

This is the piece that actually answers "does S3 need a VPC" in the security sense — S3 doesn't need one to function, but you can choose to make a *specific bucket* effectively unreachable from anywhere except a specific VPC, which is a deliberate design choice, not a default.

---

**Real-world example — CloudCart**

CloudCart's EKS worker nodes for the `payments` service run in **private subnets with no NAT Gateway at all** — a deliberate cost and security decision, since that service processes sensitive data and we didn't want it to have any outbound path to the public internet, full stop. But the pods still need to write transaction logs to an S3 bucket for compliance archiving.

We added a **Gateway VPC Endpoint for S3** to those private subnets' route tables. That alone solved reachability — zero NAT cost, and the traffic never left AWS's internal network. Then, because this was a compliance-sensitive bucket, we layered the `aws:sourceVpce` bucket policy condition on top, so even if a `payments` service credential somehow leaked outside the company, it couldn't be used to read that bucket from anywhere except through that specific VPC endpoint. Two separate problems — "can I reach it privately" and "can it be reached from anywhere else" — solved by two different mechanisms, which is exactly the distinction I'd walk an interviewer through.

---

**Complete thought process — how I approach this in the interview**

```
Does S3 require a VPC to function?
  → No. S3 is a public, regional AWS service reached over a public
    endpoint — it is never deployed "into" a VPC like EC2 or RDS.

Then why does the question exist?
  → Because workloads that ARE inside a VPC (especially private subnets
    with no internet route) need a way to reach S3.

  Is the subnet private, with no NAT/IGW?
    → Add a Gateway VPC Endpoint for S3 — free, stays on AWS's private
      network, just a route table entry.

  Need to reach S3 privately from on-prem (Direct Connect/VPN)?
    → Use an Interface Endpoint (PrivateLink) instead — Gateway
      Endpoints only work from inside the VPC itself.

  Need to guarantee the bucket is UNREACHABLE from outside that VPC,
  even with valid credentials?
    → Bucket policy with a Deny + aws:sourceVpce condition — this is
      a deliberate lockdown, separate from the endpoint itself.
```

---

**Summary (what to say if time is short):**

*"No — S3 doesn't require a VPC and isn't deployed inside one, unlike EC2 or RDS. It's a public, regional service reached over its own HTTPS endpoint, so any client with valid credentials and internet access can reach it by default. The question usually comes up because of private subnets with no NAT or internet gateway — in that case, the fix isn't giving S3 a VPC, it's giving the private subnet a route to S3 via a Gateway VPC Endpoint, which is free and keeps traffic entirely on AWS's internal network instead of going through a NAT Gateway. And if the goal is actually to lock a bucket down so it's only reachable from inside that VPC — even with a valid leaked credential — that's a separate step: a bucket policy that denies access unless the request comes through that specific VPC endpoint."*

---

#### Q5. If the frontend, backend, and database are all deployed in private subnets, how can an end user access the application?

**Answer:**

This question is checking whether you understand that "private subnet" doesn't mean "unreachable" — it means "not directly reachable from the internet." There's a very important distinction between a resource having no route in from the internet, and an application being completely inaccessible to end users. The answer is: you never expose the frontend, backend, or database directly — you put a **public-facing entry point in a public subnet in front of them**, and that entry point forwards traffic into the private subnets over the VPC's internal networking. Let me build the full picture.

---

**The core concept — a public subnet doesn't need to hold your application, just the door to it**

A private subnet, by definition, has no route to an **Internet Gateway** in its route table — instances there have no public IP and nothing on the internet can initiate a connection to them. That's exactly the security posture you want for a frontend, backend, and database: none of them should be directly reachable from `0.0.0.0/0`.

But a **public subnet** in the same VPC does have a route to the Internet Gateway. So the standard pattern is: put something in the public subnet whose entire job is to accept internet traffic and hand it off internally — a **Load Balancer** — while everything that actually does work (frontend, backend, database) stays private.

```
                         End User
                            │
                            ▼
                     Route 53 (DNS)
                            │
                            ▼
              CloudFront (CDN / WAF / TLS) — optional
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │           PUBLIC SUBNET                │
        │   Internet-facing ALB (has public IP)  │
        └────────────────┬────────────────────────┘
                          │  VPC-internal traffic only
                          ▼
        ┌───────────────────────────────────────┐
        │        PRIVATE SUBNET — App tier        │
        │   Frontend (ECS/EC2/EKS pods)           │
        │   Backend (ECS/EC2/EKS pods)             │
        └────────────────┬────────────────────────┘
                          │  SG-restricted, DB port only
                          ▼
        ┌───────────────────────────────────────┐
        │       PRIVATE SUBNET — Data tier         │
        │   RDS / Database — no internet route at all │
        └───────────────────────────────────────┘
```

The Load Balancer is the only thing in this entire diagram that has a public IP. Everything past it is reached over the VPC's private network, using private IPs, and is filtered by security groups — not by whether the subnet happens to be "public" or "private."

---

**Step by step — how the request actually flows**

1. **DNS** — The user's browser resolves your domain via **Route 53** to either a CloudFront distribution or directly to the ALB's DNS name.
2. **(Optional) CloudFront** — A CDN sits in front of the ALB for caching static frontend assets, TLS termination at the edge, and pairing with **AWS WAF** to filter malicious traffic before it even reaches your VPC.
3. **Internet-facing ALB (public subnet)** — This is the actual entry point into your VPC. It's deployed across public subnets in multiple AZs, has a public DNS name, and its security group allows inbound `443` from `0.0.0.0/0`. This is the **only** thing in the architecture allowed to accept unsolicited internet traffic.
4. **ALB → Frontend target group (private subnet)** — The ALB forwards the request over the VPC's internal network to frontend targets (EC2 instances, ECS tasks, or EKS pods) sitting in a private subnet. Their security group only allows inbound traffic **from the ALB's security group** — not from the internet directly.
5. **Frontend → Backend** — If the frontend needs to call backend APIs, that's either routed through the same ALB on a different path/listener rule (`/api/*` → backend target group), or through a **second, internal-only ALB** that isn't internet-facing at all — both frontend and backend stay in private subnets, only talking to each other over private IPs.
6. **Backend → Database (private subnet, most restricted tier)** — The database's security group allows inbound only on its DB port (5432/3306/etc.) **from the backend's security group specifically** — nothing else, not even the frontend, and never the internet. The database subnet often doesn't even have a route to a NAT Gateway, since a database has no legitimate reason to reach the internet at all.

---

**A very common point of confusion — NAT Gateway does NOT solve this**

Interviewers love this follow-up: "So do we need a NAT Gateway for users to reach the private backend?" The answer is **no** — and knowing why shows real understanding, not memorization.

| | Direction | Purpose |
|---|---|---|
| **NAT Gateway** | **Outbound only** — private subnet → internet | Lets private resources initiate connections out (pulling container images, calling third-party APIs, OS patching) |
| **Internet-facing Load Balancer** | **Inbound** — internet → private subnet | Lets external users initiate connections in |

A NAT Gateway is asymmetric by design — it lets something *inside* the private subnet reach *out*, but there is no way to route inbound traffic *in* through a NAT Gateway. If your backend needs to call an external payment API, that's NAT Gateway. If a user needs to reach your backend, that's the Load Balancer. They solve opposite problems and people mix them up constantly in interviews.

---

**Security group chaining — the real enforcement mechanism**

Subnets being "public" or "private" is about routing (is there a route to an Internet Gateway or not). The actual access control that makes this safe is **security groups**, chained tier by tier:

```
ALB Security Group:
  Inbound:  443 from 0.0.0.0/0                    ← only tier open to the internet

Frontend Security Group:
  Inbound:  80/443 from ALB-SecurityGroup only     ← not from 0.0.0.0/0

Backend Security Group:
  Inbound:  8080 from Frontend-SecurityGroup only

Database Security Group:
  Inbound:  5432 from Backend-SecurityGroup only   ← nothing else, ever
```

Each tier only accepts traffic from the security group of the tier immediately in front of it — not by IP range, by security group reference. This is the pattern I'd actually write into Terraform, and it's what actually enforces the "private means private" guarantee — the subnet's lack of an internet route is the first layer, security groups are the layer that really matters.

---

**On EKS specifically**

If frontend/backend are running as pods in a private EKS cluster, this same pattern is what the **AWS Load Balancer Controller** automates: you create a Kubernetes `Ingress` resource, and the controller provisions an actual internet-facing ALB in the public subnets, with target groups pointing at pod IPs in the private subnets — you never give a pod a public IP, ever. The Ingress annotation `alb.ingress.kubernetes.io/scheme: internet-facing` is what tells it to place the ALB in the public subnet; `internal` would keep it entirely private, for backend-to-backend traffic that should never reach the internet.

---

**Real-world example — CloudCart**

CloudCart runs a classic 3-tier VPC: a public subnet holding only the internet-facing ALB (and a NAT Gateway for outbound needs), a private "app" subnet holding the frontend and backend as ECS Fargate tasks, and a fully isolated private "data" subnet holding RDS Postgres — the data subnet's route table has **no route to anything**, not even the NAT Gateway, since the database never needs to reach out.

End users hit `shop.cloudcart.com`, resolved via Route 53 to a **CloudFront** distribution (for static asset caching and WAF protection), which forwards dynamic requests to the internet-facing **ALB**. The ALB has two listener rules on the same domain: `/` routes to the frontend target group, `/api/*` routes to the backend target group — both target groups point at Fargate tasks in the private app subnet. The backend's security group is the only one allowed to reach RDS on port 5432.

The NAT Gateway in the public subnet exists purely so the frontend and backend tasks can pull container images from ECR and call the external payment gateway's API — it has nothing to do with how end users reach the app. That distinction — NAT for outbound app needs, ALB for inbound user access — is something we specifically call out in our architecture diagram's legend, because new engineers on the team ask about it constantly.

---

**Complete thought process — how I approach this in the interview**

```
"Private subnet" means no route IN from the internet — it does NOT mean
"unreachable." The fix is never "make it public," it's "put a public-facing
entry point in front of it."

Who needs to reach this tier, and from where?

  End users, from the internet
    → Frontend needs a public-facing ALB (+ optionally CloudFront/WAF)
      in a public subnet, forwarding to the frontend in a private subnet

  Frontend needing to call Backend APIs
    → Same ALB with path-based routing, OR a second internal-only ALB —
      still entirely inside private subnets, never exposed publicly

  Backend needing to reach the Database
    → Direct private connection, security-group-restricted to the
      backend's SG only — the DB is never internet-facing, ever

  Private resources needing to reach OUT to the internet
    → That's a NAT Gateway — a completely separate, outbound-only concern,
      not how end users get in
```

---

**Summary (what to say if time is short):**

*"Private subnets aren't unreachable, they just have no route in from the internet — so end users never talk to the frontend, backend, or database directly. Instead, I'd put an internet-facing Application Load Balancer in a public subnet, optionally behind CloudFront and WAF, and that's the only thing in the whole architecture with a public IP. The ALB forwards traffic over the VPC's internal network to the frontend in a private subnet, the frontend's API calls go to the backend either through the same ALB with path-based routing or a second internal ALB, and the backend reaches the database directly over a private connection restricted by security groups to just the backend's security group — never the internet, never even the frontend directly. The part I'd make sure to call out explicitly is that a NAT Gateway doesn't solve this — NAT is outbound-only, for private resources reaching out to the internet, like pulling container images or calling a third-party API. Inbound user access is always the Load Balancer's job, and confusing the two is a common mistake."*

---

#### Q6. If secrets are created in AWS Secrets Manager, how can Amazon EKS access those secrets?

**Answer:**

There are really two separate problems buried inside this question, and I always split them apart before answering: **how does a pod authenticate to AWS without static credentials**, and **how does the actual secret value get from Secrets Manager into a running pod**. Kubernetes has no native concept of AWS Secrets Manager — it's an AWS service, entirely outside the cluster — so something has to bridge the two, and there are three real ways to do it, each with different trade-offs.

---

**Problem 1 — Authentication: how does the pod prove who it is to AWS?**

This is exactly the "no access keys" principle from earlier — a pod should never have an `AWS_SECRET_ACCESS_KEY` baked into its image or injected as a plain env var. The answer is **IRSA (IAM Roles for Service Accounts)**, or its newer, simpler replacement, **EKS Pod Identity**.

**IRSA — the original, OIDC-based mechanism:**

1. Every EKS cluster has its own **OIDC issuer URL**. You register that as an IAM OIDC Identity Provider.
2. You create an IAM role whose trust policy allows `sts:AssumeRoleWithWebIdentity`, scoped down to one specific Kubernetes namespace + ServiceAccount using a `sub` condition:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:production:backend-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

3. You annotate a Kubernetes **ServiceAccount** with that role's ARN:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/eks-secrets-reader-role
```

4. Any pod using `serviceAccountName: backend-sa` automatically gets a short-lived, auto-rotated web identity token projected into it by EKS's built-in mutating webhook — the AWS SDK inside the pod picks this up transparently and exchanges it for temporary STS credentials. No secret is stored anywhere; this is the same STS-based, temporary-credential pattern from the multi-account login question — just applied at the pod level instead of the human level.

**EKS Pod Identity — the newer, simpler alternative (2023+):**
Instead of manually crafting OIDC trust policy conditions per namespace/ServiceAccount, you run the **EKS Pod Identity Agent** (an AWS-managed add-on) and create the mapping with one API call:

```bash
aws eks create-pod-identity-association \
  --cluster-name cloudcart-prod \
  --namespace production \
  --service-account backend-sa \
  --role-arn arn:aws:iam::111122223333:role/eks-secrets-reader-role
```

Same underlying idea — temporary, no-static-key credentials scoped to a specific ServiceAccount — just less manual OIDC/trust-policy wiring. I'd use this on any newer cluster; IRSA still shows up constantly in existing environments, so it's worth knowing both.

Either way, the IAM role itself is scoped tightly:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "arn:aws:secretsmanager:us-east-1:111122223333:secret:cloudcart/prod/db-credentials-*"
    }
  ]
}
```

---

**Problem 2 — Retrieval: how does the secret value get into the pod?**

Once the pod can authenticate, there are three ways to actually get the secret contents in.

| Method | Where the secret ends up | Auto-refresh | Native k8s Secret created? | Best for |
|---|---|---|---|---|
| **Secrets Store CSI Driver** | Mounted as a file in the pod (tmpfs) | Optional, on a poll interval | Optional — only if you opt in | Maximum security; avoids storing the value in etcd at all |
| **External Secrets Operator (ESO)** | Synced into a real Kubernetes `Secret` object | Yes, on a `refreshInterval` | Yes, always | Apps that already expect a normal k8s Secret/env var, minimal app changes |
| **Direct AWS SDK call in app code** | Fetched at runtime, held in memory by the app | Fully app-controlled | No | Maximum control, at the cost of AWS-specific code in the app |

**Option A — Secrets Store CSI Driver + AWS provider (my default recommendation)**

This is a CSI (Container Storage Interface) volume driver, paired with an AWS-specific provider plugin, that mounts secrets directly as files — the secret never has to exist as a Kubernetes `Secret` object in etcd unless you explicitly ask it to.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: cloudcart-db-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "cloudcart/prod/db-credentials"
        objectType: "secretsmanager"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: production
spec:
  serviceAccountName: backend-sa       # IRSA-annotated ServiceAccount from above
  containers:
    - name: backend
      image: cloudcartacr/backend:v1
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "cloudcart-db-secrets"
```

The app reads the credential from `/mnt/secrets/cloudcart/prod/db-credentials` on disk at startup. If you also want it available as an env var (some apps don't support reading from a file), the CSI driver can optionally **sync** it into a native k8s `Secret` too, via a `secretObjects` block — that's an explicit opt-in, not the default behavior.

**Option B — External Secrets Operator**

Different philosophy: instead of mounting on-demand, ESO continuously watches Secrets Manager and keeps a real Kubernetes `Secret` object in sync automatically.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials-k8s-secret
  data:
    - secretKey: password
      remoteRef:
        key: cloudcart/prod/db-credentials
        property: password
```

This is genuinely simpler to consume — any app or Helm chart that already expects a standard k8s `Secret` just works, no code changes — but the trade-off is the value now permanently exists as an actual `Secret` object in etcd (encrypted at rest if you have envelope encryption with KMS turned on, but still present), which some compliance requirements explicitly want to avoid.

**Option C — Direct AWS SDK call in application code**

The app itself calls `secretsmanager:GetSecretValue` using the IRSA-issued credentials, with no CSI driver or operator involved at all:

```python
import boto3
client = boto3.client("secretsmanager", region_name="us-east-1")
response = client.get_secret_value(SecretId="cloudcart/prod/db-credentials")
```

Maximum control (the app decides exactly when to fetch and re-fetch), but it means AWS-specific code inside the application, which hurts portability if you ever need the same app to run outside EKS.

---

**Real-world example — CloudCart**

CloudCart's compliance requirements explicitly stated that database credentials must never be stored as a native Kubernetes `Secret` object — even encrypted at rest, the security team didn't want them sitting in etcd at all. So for the backend's database password, we used the **Secrets Store CSI Driver**, mounted as a file only, with no `secretObjects` sync — the app reads the password from disk at container startup and never touches a k8s `Secret`.

Authentication was via **IRSA**, with the IAM role scoped down with a resource ARN that only matched secrets under the `cloudcart/prod/*` prefix — so even if that role were somehow assumed outside its intended context, it couldn't read secrets belonging to other services.

For rotation, AWS Secrets Manager automatically rotates the DB password every 30 days via its built-in Lambda rotation function. We set the CSI driver's `rotationPollInterval` so it periodically re-mounts the updated file, and paired it with a lightweight controller (`stakater/reloader`) watching for the CSI driver's optional synced Secret to change, which then triggers a rolling restart of the backend deployment — so the new credential actually gets picked up, rather than sitting unused on disk while the running process still holds the old one in memory.

For a different, lower-sensitivity service — a third-party analytics API key that a Helm chart expected as a plain env var and we didn't control the chart's code — we used **External Secrets Operator** instead, since fighting the chart to read from a mounted file wasn't worth it for a non-compliance-sensitive secret.

---

**Complete thought process — how I approach this in the interview**

```
Two separate problems:

1. How does the pod AUTHENTICATE to AWS without static keys?
   → IRSA (OIDC federation, ServiceAccount annotation) — the established method
   → EKS Pod Identity — newer, simpler association API, same underlying idea

2. How does the SECRET VALUE get from Secrets Manager into the pod?

   Must the secret NEVER exist as a native k8s Secret (compliance)?
     → Yes → Secrets Store CSI Driver, mount as a file only, no secretObjects sync

   Does the app/Helm chart expect a normal k8s Secret or env var,
   and I don't control the app code?
     → Yes → External Secrets Operator, syncs into a real k8s Secret automatically

   Do I want full control and don't mind AWS-specific code in the app?
     → Direct AWS SDK call using the IRSA-issued credentials

   Does the secret rotate?
     → Set a rotation poll interval on the CSI driver / ESO refreshInterval,
       and pair it with something that restarts pods on change if the app
       doesn't reload credentials on its own
```

---

**Summary (what to say if time is short):**

*"There are two parts to this — authentication and retrieval. For authentication, a pod should never carry static AWS credentials — I'd use IRSA, or the newer EKS Pod Identity, so the pod's ServiceAccount is mapped to an IAM role and gets short-lived, auto-rotated STS credentials with no secret stored anywhere. For retrieval, my default is the Secrets Store CSI Driver with the AWS provider, which mounts the secret directly as a file in the pod — that avoids ever creating a native Kubernetes Secret object in etcd, which matters for compliance-sensitive data like database credentials. If the application or a Helm chart I don't control expects a standard Kubernetes Secret instead of a mounted file, I'd use External Secrets Operator, which keeps a real k8s Secret continuously synced from Secrets Manager. And for rotation, since Secrets Manager can auto-rotate credentials on a schedule, I'd make sure the CSI driver's poll interval picks up the new value and pair it with something like Reloader to actually restart the pods, since a rotated secret on disk doesn't do anything until the running process re-reads it."*

---

#### Q7. How do you set up RBAC in Amazon EKS?

**Answer:**

This question is broader than Q1's IAM-binding question — that one was specifically about connecting an IAM user to the cluster; this one is asking for the whole picture, end to end: designing the actual permission model, writing the native Kubernetes RBAC objects, and only then connecting AWS identities into it. I always start by making the same point I made in Q1: **RBAC itself is 100% native Kubernetes** — it works identically on EKS, self-managed clusters, or minikube. The only EKS-specific piece is the identity mapping layer, which I'll only touch briefly here since Q1 covers it in depth.

---

**The four RBAC building blocks**

| Object | Scope | Purpose |
|---|---|---|
| **Role** | Single namespace | Defines a set of permissions (verbs on resources) within one namespace |
| **ClusterRole** | Cluster-wide (or reusable) | Defines permissions that apply cluster-wide, OR a reusable rule set you bind per-namespace |
| **RoleBinding** | Single namespace | Grants a Role (or a ClusterRole, used in a namespaced way) to a subject, within one namespace |
| **ClusterRoleBinding** | Cluster-wide | Grants a ClusterRole to a subject across the entire cluster |

A **Role**/**ClusterRole** is just a list of rules — `apiGroups`, `resources`, `verbs` — it doesn't grant anything to anyone by itself. A **RoleBinding**/**ClusterRoleBinding** is what actually connects that rule set to a `User`, `Group`, or `ServiceAccount`. People frequently forget the Role/Binding split is two separate objects — you always need both.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                       # WHAT is allowed
metadata:
  name: developer-role
  namespace: staging
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps", "pods/log"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]       # even developers only get read access to secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                # WHO gets it
metadata:
  name: developer-binding
  namespace: staging
subjects:
  - kind: Group
    name: developers-group       # a Kubernetes group — not tied to any one person
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

---

**Step-by-step — how I actually set this up**

**Step 1 — Design the access model first, before writing any YAML**

I always start with a table, not a manifest — figure out who needs what, at what scope:

| Group | Scope | Access level | Bound to |
|---|---|---|---|
| Platform team | Cluster-wide | `cluster-admin` | `platform-admin-group` |
| Developers | Own team's namespace only | Read/write on workloads, read-only on secrets | `developers-group` |
| QA | `staging` namespace only | Read-only (pods, logs, services) | `qa-readonly-group` |
| CI/CD pipeline | `production` namespace, specific resources only | Update deployments/configmaps — nothing else | `ci-deployer-group` |
| Auditors/Security | Cluster-wide | Read-only, everywhere | `security-audit-group` |

Getting this table right is most of the actual work — the YAML afterward is mechanical.

**Step 2 — Write the ClusterRoles/Roles**

For access that should apply the same way across every namespace (like the auditor role), use a **ClusterRole**, bound with a **ClusterRoleBinding**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-audit-role
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["pods", "deployments", "services", "jobs", "networkpolicies", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-audit-binding
subjects:
  - kind: Group
    name: security-audit-group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: security-audit-role
  apiGroup: rbac.authorization.k8s.io
```

For access that should be tightly scoped to one namespace — like the CI/CD deployer, which should never be able to touch anything outside `production` — use a namespaced **Role** + **RoleBinding**, deliberately narrow:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]     # no create/delete — deployments already exist
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: Group
    name: ci-deployer-group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: ci-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

Notice the CI role has no `secrets`, no `delete`, no `apiGroups: ["*"]` — a compromised CI pipeline should only be able to do exactly the one job it's meant to do.

**Step 3 — Use Kubernetes' built-in ClusterRoles where they fit**

EKS ships with four default ClusterRoles you rarely need to recreate yourself:

| Built-in ClusterRole | What it grants |
|---|---|
| `view` | Read-only on most objects — explicitly excludes Secrets |
| `edit` | Read/write on most objects in a namespace — not RBAC objects themselves |
| `admin` | Full access within a namespace, including managing RoleBindings there |
| `cluster-admin` | Unrestricted access to everything, cluster-wide — the most dangerous one |

For a lot of real teams, `edit` and `view` bound via a namespaced RoleBinding cover 80% of what you'd otherwise hand-write.

**Step 4 — Map AWS IAM identities into these Kubernetes groups**

This is the part covered in depth in Q1 — I won't repeat the full walkthrough here, but the short version: either add entries to the `aws-auth` ConfigMap (legacy clusters) or create **EKS Access Entries** (modern clusters), mapping each IAM user or role's ARN to the exact group name used in the RoleBindings above — `developers-group`, `qa-readonly-group`, `ci-deployer-group`, etc.

```bash
aws eks create-access-entry \
  --cluster-name cloudcart-prod \
  --principal-arn arn:aws:iam::111122223333:role/developers-sso-role \
  --kubernetes-groups developers-group
```

**The one rule I'd stress here that Q1 didn't need to:** always bind RBAC to **Groups**, never directly to individual IAM usernames. If you bind permissions straight to `arn:aws:iam::...:user/john`, then offboarding John means going and editing every RoleBinding he was ever added to. If everything is bound to `developers-group` instead, offboarding is just removing John from that one group mapping — the RBAC objects themselves never change.

**Step 5 — Verify**

```bash
kubectl auth can-i update deployments -n production --as=john
# no    (john is in developers-group, not ci-deployer-group)

kubectl auth can-i --list --as=john -n staging
# shows the full permission set for that identity in that namespace

aws eks list-access-entries --cluster-name cloudcart-prod
```

---

**What RBAC does NOT cover — worth mentioning proactively**

RBAC only answers "can this identity call this verb on this resource type." It has no concept of things like "deployments must have resource limits set," "images must come from an approved registry," or "no pod may run as root" — those are enforced by an **admission controller** (OPA Gatekeeper, Kyverno), a completely different, complementary layer. I always mention this distinction unprompted — it shows I understand RBAC's actual boundary rather than treating it as a catch-all security tool.

---

**Real-world example — CloudCart**

CloudCart's RBAC model maps almost exactly to the table in Step 1. A few details from how we actually built it:

- We deliberately kept `cluster-admin` down to **3 people** on the platform team, bound through `platform-admin-group` — every other team, no matter how senior, gets namespace-scoped access only.
- Each product team got its own namespace and its own `Role` (not the built-in `edit`, because we wanted to explicitly exclude `delete` on `namespaces` and `secrets` create/delete even for team leads).
- The `ci-deployer-role` shown above is the actual role our GitHub Actions pipeline uses — mapped via **OIDC web identity federation** into an IAM role, which is in turn mapped into `ci-deployer-group` via an EKS Access Entry. No human and no static credential is involved in production deployments at all.
- We run a scheduled job that dumps `kubectl auth can-i --list --as=<group>` for every group monthly and diffs it against the previous month — mostly to catch RBAC drift where someone added a broad rule "just to unblock a deploy" and forgot to narrow it back down afterward.

---

**Complete thought process — how I approach this in the interview**

```
RBAC setup on EKS = pure Kubernetes RBAC + one AWS-specific mapping step

1. Design the access model first — who needs what, at what scope
   (namespace-level vs cluster-wide)

2. Namespace-scoped access → Role + RoleBinding
   Cluster-wide access      → ClusterRole + ClusterRoleBinding

3. Reuse built-in ClusterRoles (view/edit/admin/cluster-admin) where
   they genuinely fit — don't hand-write what already exists

4. ALWAYS bind to Groups, never directly to individual IAM
   usernames/roles — makes onboarding/offboarding a one-line change

5. Map IAM identities → Kubernetes groups via aws-auth ConfigMap
   (legacy) or EKS Access Entries (modern) — this is the only
   EKS-specific step, everything else is portable Kubernetes RBAC

6. Verify with `kubectl auth can-i --list --as=<user/group>`

7. Remember RBAC's boundary — it doesn't enforce pod security or
   resource limits, that's OPA/Kyverno's job, a separate layer
```

---

**Summary (what to say if time is short):**

*"RBAC in EKS is entirely native Kubernetes — Roles and ClusterRoles define what's allowed, RoleBindings and ClusterRoleBindings define who gets it, and I always design the access model as a table first — who needs what, at namespace scope or cluster scope — before writing any YAML. I use namespace-scoped Roles for anything that should stay contained to one team or environment, ClusterRoles for things that genuinely need to apply everywhere, and I lean on the built-in `view`/`edit`/`admin`/`cluster-admin` ClusterRoles where they fit instead of reinventing them. The one rule I never break is binding everything to Kubernetes Groups, never to individual IAM identities directly — that's what makes onboarding and offboarding a one-line change instead of hunting through every RoleBinding in the cluster. The only EKS-specific step is mapping IAM identities into those group names, either through the `aws-auth` ConfigMap or EKS Access Entries, which is exactly what Q1 covers in depth. And I always mention that RBAC only controls API-level permissions — it doesn't enforce things like pod security standards or resource limits, that's a separate admission-control layer like OPA Gatekeeper or Kyverno."*

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
