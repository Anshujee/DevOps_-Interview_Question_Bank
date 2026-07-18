# Real Interview Asked Questions & Answers

> This file is a personal log of actual questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1 — Capgemini | Azure DevOps | Technical Round 1](#interview-1)
  - [Q1. Rate yourself in Terraform](#q1-rate-yourself-in-terraform)
  - [Q2. Which Terraform commands do you use daily](#q2-which-terraform-commands-do-you-use-daily)
  - [Q3. Can terraform apply run without terraform plan](#q3-can-terraform-apply-run-without-terraform-plan)
  - [Q4. Difference between Provider and Provisioner](#q4-what-is-the-difference-between-a-provider-and-a-provisioner)
  - [Q5. Types of Provisioners](#q5-what-are-the-different-types-of-provisioners-in-terraform)
  - [Q6. What is Drift Detection](#q6-what-is-drift-detection-in-terraform)
  - [Q7. How do you resolve Terraform Drift](#q7-how-do-you-resolve-terraform-drift)
  - [Q8. Will you update the Terraform state file manually](#q8-will-you-update-the-terraform-state-file-manually)
  - [Q9. Implicit vs Explicit Dependency](#q9-what-is-the-difference-between-implicit-dependency-and-explicit-dependency-in-terraform)
  - [Q10. depends_on keyword and real usage](#q10-which-keyword-is-used-for-explicit-dependency-where-have-you-used-depends_on-in-your-project)
  - [Q11. Why should infrastructure changes always be done through Terraform instead of Azure Portal?](#q11-why-should-infrastructure-changes-always-be-done-through-terraform-instead-of-azure-portal)
  - [Q12. Difference between Terraform Resource and Data Source](#q12-difference-between-terraform-resource-and-data-source)
  - [Q13. Why is using a Data Source better than hardcoding the Resource Group name?](#q13-why-is-using-a-data-source-better-than-hardcoding-the-resource-group-name)
  - [Q14. You are reviewing a Terraform Pull Request. How would you review it?](#q14-youre-reviewing-a-terraform-pull-request-how-would-you-review-it)
  - [Q15. Which tools would you use to validate Terraform code before approval?](#q15-which-tools-would-you-use-to-validate-terraform-code-before-approval)
  - [Scenario 1. EC2 instance type manually changed in AWS Console](#scenario-1-someone-manually-changed-an-ec2-instance-type-in-the-aws-console-terraform-detects-drift-what-will-you-do)
  - [Scenario 2. Fix only one resource without applying complete infrastructure](#scenario-2-your-terraform-deployment-takes-around-one-hour-for-both-plan-and-apply-after-deployment-a-critical-production-issue-is-found-in-only-one-resource-how-will-you-fix-only-that-resource-without-applying-the-complete-infrastructure)
  - [Scenario 3. Will terraform apply -target update the state file?](#scenario-3-if-you-use-terraform-apply--target-will-the-terraform-state-file-also-be-updated)
  - [Scenario 4. Have you used terraform -target in your production project?](#scenario-4-have-you-used-terraform--target-in-your-production-project)
  - [Scenario 5. Customer wants Read-Only access to an existing Resource Group](#scenario-5-you-have-an-existing-terraform-managed-azure-environment-a-customer-wants-read-only-access-to-an-existing-resource-group-how-would-you-implement-this)
  - [Scenario 6. Terraform Plan detects Drift — Customer confirms valid vs not valid](#scenario-6-suppose-terraform-plan-detects-drift-how-would-you-proceed-and-customer-confirms-the-drift-is-valid-how-would-you-proceed-what-if-the-customer-says-the-drift-is-not-valid)

---

## Interview #1

**Company:** Capgemini  
**Date:** 18-07-2026  
**Role Applied For:** Azure DevOps  
**Round:** Technical Round 1  
**Interviewer Level:** Tech Lead

---

### Questions Asked

#### Q1. Rate yourself in Terraform.

**Answer:**

I would rate myself **8 out of 10** in Terraform.

I have been working with Terraform for a significant part of my DevOps career. I have used it to build real production infrastructure on Azure — things like AKS clusters, Virtual Networks, Subnets, NSGs, Azure Container Registry, Key Vault, Storage Accounts, and more. I am very comfortable with the core concepts, writing modules, managing state, and working in team environments where multiple people use Terraform together.

The reason I say 8 and not 10 is because Terraform is a vast tool — there are always new providers, advanced features like CDK for Terraform, and deep internals of the state engine that I am still exploring. I believe in being honest about where I stand.

---

**Let me explain what I know well:**

---

**1. What is Terraform and why do we use it?**

Terraform is an **Infrastructure as Code (IaC)** tool made by HashiCorp. It lets you write your cloud infrastructure — servers, networks, databases, everything — as code in files, instead of clicking around in a portal manually.

Think of it like this: instead of going to the Azure Portal and creating a Virtual Network by clicking buttons, you write a `.tf` file that says *"I want a Virtual Network with this address space"* — and Terraform creates it for you automatically.

**Why this is powerful:**
- You can create the same infrastructure 100 times without mistakes
- You can version control your infrastructure in Git just like application code
- You can destroy and recreate environments easily (dev, staging, prod)
- Teams can review infrastructure changes before they are applied, just like a code review

---

**2. Core Concepts I work with daily:**

**Provider**
A provider tells Terraform which cloud or platform to talk to. For Azure, we use the `azurerm` provider.

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

This is the first thing in every Terraform project — it tells Terraform "we are working on Azure."

---

**Resource**
A resource is the actual infrastructure you want to create. For example, a Resource Group:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "azureshop-rg"
  location = "East US"
}
```

This creates a Resource Group called `azureshop-rg` in East US. Simple and readable.

---

**Variables**
Instead of hardcoding values, we use variables so the same code can be reused for different environments.

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

Then use it like:
```hcl
name = "azureshop-${var.environment}-rg"
```

For dev it becomes `azureshop-dev-rg`, for prod it becomes `azureshop-prod-rg`. Same code, different values.

---

**Outputs**
Outputs are used to display or share values after Terraform runs. For example, after creating an AKS cluster, we output the cluster name so it can be used in pipelines.

```hcl
output "aks_cluster_name" {
  value = azurerm_kubernetes_cluster.main.name
}
```

---

**Data Sources**
Data sources let you **read** existing infrastructure without managing it. For example, if a Virtual Network already exists and I just need its ID:

```hcl
data "azurerm_virtual_network" "existing" {
  name                = "azureshop-vnet"
  resource_group_name = "azureshop-rg"
}
```

I use this often when some resources are managed by a different team and I just need to reference them.

---

**3. Terraform State — The most important concept**

Terraform keeps track of everything it has created in a file called **terraform.tfstate**. This is Terraform's memory — it knows what exists, what needs to be changed, and what needs to be deleted.

**The problem with local state:**
If you store state on your laptop, your teammates cannot see it. If your laptop crashes, the state is lost and Terraform loses track of what it created.

**The solution — Remote Backend:**
We store state in **Azure Blob Storage** so the entire team shares the same state file.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "azureshop-tfstate-rg"
    storage_account_name = "azureshoptfstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

This means when I run `terraform apply` and my colleague runs it later, we are both working from the same state. No conflicts, no duplicates.

**State Locking:**
When one person is running Terraform, it automatically locks the state file so no one else can run it at the same time. This prevents two people from making conflicting changes simultaneously.

---

**4. Modules — Writing reusable Terraform code**

A module is a reusable block of Terraform code. Instead of writing the same AKS cluster code in every project, I write it once as a module and call it wherever needed.

```hcl
module "aks" {
  source              = "./modules/aks"
  cluster_name        = "azureshop-aks"
  resource_group_name = azurerm_resource_group.main.name
  node_count          = 3
  vm_size             = "Standard_D2s_v3"
}
```

This is like a function in programming — write once, use many times. In AzureShop, we have separate modules for networking, AKS, ACR, Key Vault, and monitoring.

---

**5. Key Terraform Commands I use daily:**

| Command | What it does |
|---|---|
| `terraform init` | Downloads providers and sets up the backend. First command always. |
| `terraform plan` | Shows what Terraform WILL do — creates, updates, or destroys. No actual changes yet. |
| `terraform apply` | Actually creates or changes the infrastructure. |
| `terraform destroy` | Destroys everything Terraform created. Used for cleanup. |
| `terraform fmt` | Formats `.tf` files to standard style. |
| `terraform validate` | Checks if the code is syntactically correct. |
| `terraform state list` | Lists all resources tracked in the state file. |
| `terraform import` | Imports existing infrastructure into Terraform state. |

---

**6. Workspaces — Managing multiple environments**

Terraform workspaces let you use the same code for multiple environments with separate state files.

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod
```

Each workspace has its own state file, so dev and prod are completely isolated even though they use the same Terraform code.

---

**7. Real Example from AzureShop Project:**

In our AzureShop project, I wrote Terraform code to provision the entire Azure infrastructure:

- **Resource Groups** — separate RGs for each service (networking, AKS, monitoring)
- **Virtual Network & Subnets** — with proper CIDR ranges and NSG rules
- **AKS Cluster** — with system and user node pools, managed identity, and RBAC enabled
- **Azure Container Registry (ACR)** — with geo-replication for high availability
- **Key Vault** — for storing secrets like database passwords and API keys
- **Azure Monitor & Log Analytics Workspace** — for centralized logging and alerting
- **Storage Account** — for Terraform remote state

All of this was version-controlled in Git, and our Azure DevOps pipeline ran `terraform plan` on every Pull Request so the team could review infrastructure changes before merging. On merge to main, `terraform apply` ran automatically.

---

**What I am still growing in (the honest 2 points):**

- **Terraform CDK (Cloud Development Kit)** — writing Terraform in Python/TypeScript instead of HCL. I have read about it but not used it in production yet.
- **Deep internals of the state engine** — things like manual state surgery using `terraform state mv` or `terraform state rm` for complex migrations. I have done it a few times but it still requires careful thought.

---

**Summary (what to say if time is short):**

*"I rate myself 8 out of 10. I have hands-on experience writing Terraform from scratch for Azure infrastructure — VNets, AKS, ACR, Key Vault, and more. I work with modules, remote state in Azure Blob Storage, workspaces for environment separation, and I have integrated Terraform into Azure DevOps pipelines for automated infrastructure deployments. The areas I am still deepening are Terraform CDK and advanced state management operations."*

---

#### Q2. Which Terraform commands do you use daily?

**Answer:**

As a DevOps Engineer working with Terraform daily, these are the commands I use the most — and I will explain each one with what it does and when exactly I use it.

---

**1. `terraform init`**

This is always the **first command** you run in any Terraform project.

What it does:
- Downloads the required **provider plugins** (like `azurerm` for Azure)
- Sets up the **backend** (connects to Azure Blob Storage for remote state)
- Downloads any **modules** referenced in the code

```bash
terraform init
```

**When I use it:**
- When I clone a Terraform repo for the first time
- When I add a new provider or module
- When the backend configuration changes

Think of it like `npm install` — you run it once to set everything up before you can do anything else.

---

**2. `terraform plan`**

This is the **most important command** in my daily work.

What it does:
- Reads your `.tf` files
- Compares with the current state file
- Shows you exactly what WILL be created, changed, or destroyed — **without actually doing anything**

```bash
terraform plan
```

Or save the plan to a file (used in pipelines):
```bash
terraform plan -out=tfplan
```

**Output example:**
```
+ azurerm_resource_group.main will be created
~ azurerm_kubernetes_cluster.main will be updated
- azurerm_storage_account.old will be destroyed
```

- `+` means **create**
- `~` means **update/modify**
- `-` means **destroy**

**When I use it:**
- Every single time before I apply any change
- In Azure DevOps pipelines on every Pull Request so the team can review infrastructure changes before merging
- To verify my code is correct without risking anything in production

**AzureShop Example:**
Before adding a new node pool to our AKS cluster, I always run `terraform plan` first. It shows me exactly which fields will change and whether it will destroy and recreate the node pool or just update it in place. This is critical — a destroy and recreate means downtime.

---

**3. `terraform apply`**

This actually **executes the changes** shown in the plan.

```bash
terraform apply
```

It first shows the plan again and asks for confirmation:
```
Do you want to perform these actions? yes
```

Or to skip the prompt (used in automated pipelines):
```bash
terraform apply -auto-approve
```

Or apply a saved plan file:
```bash
terraform apply tfplan
```

**When I use it:**
- After reviewing the plan and confirming everything looks correct
- In the CD pipeline after a PR is merged to main — `terraform apply` runs automatically to provision infrastructure

---

**4. `terraform destroy`**

Destroys **everything** that Terraform created and is tracked in the state file.

```bash
terraform destroy
```

**When I use it:**
- To tear down a dev or staging environment to save cloud costs over weekends
- To clean up after testing
- To rebuild an environment from scratch

> I am always careful with this command in production. I never run it without double-checking the workspace and state file first.

---

**5. `terraform fmt`**

Automatically **formats** your `.tf` files to the standard Terraform style — correct indentation, alignment, spacing.

```bash
terraform fmt -recursive
```

**When I use it:**
- Before every commit to Git
- In our pipeline as a check — if code is not formatted, the pipeline fails and the developer must fix it

Think of it like `prettier` for JavaScript — it keeps the codebase clean and consistent across the whole team.

---

**6. `terraform validate`**

Checks if your Terraform code is **syntactically correct** — no typos, missing arguments, or wrong resource types.

```bash
terraform validate
```

**When I use it:**
- After writing new code, before running plan
- As the first step in the CI pipeline — catches basic errors fast without connecting to Azure

> `validate` only checks syntax. It does NOT check if the resources actually exist in Azure or if your values are correct. That is what `plan` does.

---

**7. `terraform state list`**

Lists all resources currently tracked in the state file.

```bash
terraform state list
```

**Example output:**
```
azurerm_resource_group.main
azurerm_kubernetes_cluster.aks
azurerm_container_registry.acr
azurerm_key_vault.kv
```

**When I use it:**
- To verify what Terraform is managing
- When troubleshooting — to check if a resource is in state or not

---

**8. `terraform import`**

Imports **existing infrastructure** into Terraform state. Used when infrastructure was created manually in the portal and you now want Terraform to manage it.

```bash
terraform import azurerm_resource_group.main /subscriptions/<sub-id>/resourceGroups/azureshop-rg
```

**When I use it:**
- When a resource was created manually before Terraform was introduced
- When migrating from manual infrastructure to IaC

> After import, you still need to write the matching `.tf` code manually. Import only updates the state file, not your code.

---

**9. `terraform workspace` commands**

Used to manage multiple environments (dev, staging, prod) with the same code but separate state files.

```bash
terraform workspace list           # Show all workspaces
terraform workspace new dev        # Create a new workspace
terraform workspace select prod    # Switch to prod workspace
terraform workspace show           # Show current workspace
```

**When I use it:**
- When switching between dev and prod to apply changes to a specific environment
- To verify I am in the correct workspace before running `apply`

---

**Full daily workflow in order:**

```bash
terraform init        # 1. Set up (first time or after changes)
terraform fmt         # 2. Format the code
terraform validate    # 3. Check for syntax errors
terraform plan        # 4. Preview what will change
terraform apply       # 5. Apply the changes
```

---

**Summary (what to say if time is short):**

*"The commands I use daily are — `init` to set up, `fmt` and `validate` to check code quality, `plan` to preview changes, and `apply` to execute them. I also use `state list` for troubleshooting, `import` when bringing existing resources under Terraform management, and `workspace` commands to switch between environments. In our Azure DevOps pipeline, `plan` runs on every PR and `apply` runs automatically on merge to main."*

---

#### Q3. Can `terraform apply` run without `terraform plan`?

**Answer:**

**Yes, technically it can. But in real production environments, we never do it that way.**

Let me explain both sides clearly.

---

**Technically — Yes it can:**

When you run `terraform apply` directly without running `terraform plan` first, Terraform automatically generates the plan internally and then shows it to you and asks for confirmation before applying.

```bash
terraform apply
```

Output:
```
Plan: 3 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

So Terraform does run a plan behind the scenes — it just does not save it as a separate step. You see the plan and then immediately confirm it.

---

**The right way — always run plan separately first:**

In professional environments, we always run `terraform plan` first and save the output to a plan file. Then we apply that exact saved plan.

```bash
# Step 1 — Generate and save the plan
terraform plan -out=tfplan

# Step 2 — Apply exactly that saved plan
terraform apply tfplan
```

**Why this is the correct approach:**

| | `terraform apply` directly | `terraform plan` then `apply tfplan` |
|---|---|---|
| Plan reviewed separately? | No — you see and apply in one step | Yes — plan is reviewed first, then applied |
| What gets applied? | A fresh plan generated at apply time | Exactly the saved plan — no surprises |
| Safe for production? | Risky | Yes — this is the standard |
| Used in pipelines? | Never | Always |

---

**Why the saved plan approach is safer:**

Imagine this scenario:
1. You run `terraform apply` and see the plan — it looks good
2. You type `yes`
3. But between the moment you saw the plan and the moment `yes` was processed, someone else on the team changed the state file

In this case, the plan Terraform applies may be **different** from what you reviewed.

With the saved plan approach (`-out=tfplan`), Terraform locks in exactly what was planned. It applies **that exact plan** — nothing can change between review and apply.

---

**In automated pipelines — plan and apply are always separate stages:**

In our AzureShop Azure DevOps pipeline, we have two separate pipeline stages:

**Stage 1 — Plan (runs on every PR):**
```bash
terraform plan -out=tfplan
```
The plan output is saved and published as an artifact. The team reviews it in the PR comments.

**Stage 2 — Apply (runs only after PR is merged to main):**
```bash
terraform apply tfplan
```
Applies exactly the reviewed and approved plan. No surprises.

This gives full visibility and control — infrastructure changes go through the same review process as code changes.

---

**One more thing — `-auto-approve` flag:**

You can also skip the confirmation prompt entirely:

```bash
terraform apply -auto-approve
```

This is only used in automated pipelines where human confirmation is not possible. **Never use this manually on production** — it will apply changes immediately without asking you.

---

**Summary (what to say if time is short):**

*"Yes, `terraform apply` can run without a separate `terraform plan` — it generates the plan internally and asks for confirmation before applying. But in production and in pipelines, we always run `terraform plan -out=tfplan` first and then `terraform apply tfplan`. This way the plan is reviewed separately and we apply exactly what was approved — no surprises between review and execution."*

---

#### Q4. What is the difference between a Provider and a Provisioner?

**Answer:**

This is a very common Terraform interview question and many people confuse the two. Let me explain both clearly with examples.

---

**In simple words:**

| | Provider | Provisioner |
|---|---|---|
| **What it is** | A plugin that connects Terraform to a cloud or platform | A way to run scripts or commands on a resource after it is created |
| **Purpose** | To **create, update, delete** cloud resources | To **configure** a resource after it is created |
| **Examples** | Azure, AWS, GCP, Kubernetes, GitHub | Run a shell script, copy a file, install software |
| **Used how often?** | Every Terraform project — always | Rarely — only when no better option exists |

---

**Provider — Explained:**

A **Provider** is a plugin that tells Terraform how to talk to a specific platform or cloud. Without a provider, Terraform does not know how to create any resource.

Think of it like a **translator** — Azure speaks its own API language, AWS speaks its own. The provider is the translator between your Terraform code and the cloud's API.

**Example — Azure Provider:**

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

Once this is defined, you can create any Azure resource:

```hcl
resource "azurerm_resource_group" "main" {
  name     = "azureshop-rg"
  location = "East US"
}
```

The `azurerm` provider handles the API call to Azure and creates the Resource Group.

**Common Providers I have used:**
- `azurerm` — for all Azure resources (AKS, ACR, Key Vault, VNet, etc.)
- `kubernetes` — to deploy Kubernetes manifests using Terraform
- `helm` — to install Helm charts via Terraform
- `github` — to manage GitHub repos and teams as code

---

**Provisioner — Explained:**

A **Provisioner** is used to execute scripts or commands **on a resource after it is created**. It is used for post-creation configuration — things like installing software, copying files, or running setup scripts.

Think of it like this — the Provider creates a virtual machine, and then the Provisioner goes inside that VM and installs Nginx on it.

**Types of Provisioners:**

**1. `remote-exec` — Run commands on the remote resource (via SSH or WinRM):**

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name = "azureshop-vm"
  # ... other config ...

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx"
    ]

    connection {
      type     = "ssh"
      user     = "adminuser"
      password = var.admin_password
      host     = self.public_ip_address
    }
  }
}
```

This creates a VM and then SSHes into it and installs Nginx.

---

**2. `local-exec` — Run commands on the machine where Terraform is running (your laptop or pipeline agent):**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name = "azureshop-aks"
  # ... other config ...

  provisioner "local-exec" {
    command = "az aks get-credentials --resource-group azureshop-rg --name azureshop-aks"
  }
}
```

After the AKS cluster is created, this runs an `az` command locally to update your kubeconfig file automatically.

---

**3. `file` — Copy a file from local machine to the remote resource:**

```hcl
provisioner "file" {
  source      = "configs/app.conf"
  destination = "/etc/app/app.conf"

  connection {
    type = "ssh"
    user = "adminuser"
    host = self.public_ip_address
  }
}
```

---

**The key difference with a real analogy:**

Imagine you are building a house:

- The **Provider** is the **construction company** — they build the house (walls, roof, windows). They create the actual structure.
- The **Provisioner** is the **interior designer** — after the house is built, they come in and set it up (put furniture, hang curtains, install appliances).

The Provider builds the resource. The Provisioner configures it after it exists.

---

**Important — Why Provisioners should be avoided when possible:**

HashiCorp themselves say provisioners are a **last resort**. Here is why:

1. **They break idempotency** — If you run `terraform apply` twice, the provisioner runs again even if nothing changed. This can cause errors.
2. **They are hard to debug** — If a provisioner script fails halfway, Terraform marks the resource as tainted and tries to destroy and recreate it on the next apply.
3. **Better alternatives exist:**
   - For VM configuration → use **cloud-init** or **Azure Custom Script Extension**
   - For Kubernetes setup → use **Helm provider** or **Kubernetes provider**
   - For software installation → use **Ansible** after Terraform creates the infrastructure

**In AzureShop**, we never used provisioners. For any post-creation VM configuration we used Azure Custom Script Extension, and for Kubernetes we used the Helm and Kubernetes Terraform providers. Provisioners were not needed.

---

**Summary (what to say if time is short):**

*"A Provider is a plugin that connects Terraform to a cloud platform like Azure — it is what allows Terraform to create, update, and delete resources. A Provisioner is used to run scripts or commands on a resource after it is created — like installing software on a VM via SSH. The key difference is — Provider creates infrastructure, Provisioner configures it. However, in practice we avoid provisioners because they break idempotency and have better alternatives like cloud-init, Ansible, or Azure Custom Script Extension."*

---

#### Q5. What are the different types of Provisioners in Terraform?

**Answer:**

Terraform has **3 main types of provisioners**. Each one is used for a different purpose. Let me explain all three in detail with real examples.

---

**Overview:**

| Provisioner | What it does | Where it runs |
|---|---|---|
| `remote-exec` | Runs commands on the **remote resource** (VM/server) | On the resource itself — via SSH or WinRM |
| `local-exec` | Runs commands on the **local machine** where Terraform is running | On your laptop or pipeline agent |
| `file` | **Copies a file** from local machine to the remote resource | Transfers over SSH or WinRM |

---

**1. `remote-exec` Provisioner**

This provisioner connects to the remote resource (usually a Virtual Machine) and runs commands directly on it.

**How it connects:**
- Linux VMs → SSH
- Windows VMs → WinRM (Windows Remote Management)

**Use case:** Install software, start services, run setup scripts on a VM after it is created.

**Example — Install Nginx on a Linux VM:**

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "azureshop-web-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = "East US"
  size                = "Standard_B2s"
  admin_username      = "adminuser"

  # ... other config ...

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx",
      "sudo systemctl enable nginx",
      "sudo systemctl start nginx"
    ]

    connection {
      type        = "ssh"
      user        = "adminuser"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip_address
    }
  }
}
```

**What happens here step by step:**
1. Terraform creates the Azure Linux VM
2. Once the VM is running, Terraform SSHes into it using the private key
3. Runs the 4 commands one by one — update packages, install nginx, enable it, start it
4. If all commands succeed, Terraform marks the resource as created successfully

**`inline` vs `script` in remote-exec:**

You can run commands in two ways:

```hcl
# Option 1 — inline: list of commands directly in the code
provisioner "remote-exec" {
  inline = [
    "sudo apt update",
    "sudo apt install -y nginx"
  ]
}

# Option 2 — script: path to a shell script file
provisioner "remote-exec" {
  script = "scripts/install_nginx.sh"
}

# Option 3 — scripts: multiple script files in order
provisioner "remote-exec" {
  scripts = [
    "scripts/setup_base.sh",
    "scripts/install_app.sh",
    "scripts/configure_app.sh"
  ]
}
```

---

**2. `local-exec` Provisioner**

This provisioner runs a command on the **machine where Terraform is running** — your laptop, or the Azure DevOps pipeline agent. It does NOT connect to the remote resource.

**Use case:** Run local CLI commands after a resource is created — like updating your kubeconfig, triggering an Ansible playbook, or calling an API.

**Example 1 — Get AKS credentials after cluster is created:**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  resource_group_name = azurerm_resource_group.main.name
  location            = "East US"
  dns_prefix          = "azureshop"

  # ... other config ...

  provisioner "local-exec" {
    command = "az aks get-credentials --resource-group azureshop-rg --name azureshop-aks --overwrite-existing"
  }
}
```

After AKS is created, this runs `az aks get-credentials` on the local machine — so your `kubectl` is automatically connected to the new cluster.

**Example 2 — Trigger an Ansible playbook after VM is created:**

```hcl
provisioner "local-exec" {
  command = "ansible-playbook -i '${self.public_ip_address},' playbooks/setup.yml"
}
```

**Example 3 — Write the output to a local file:**

```hcl
provisioner "local-exec" {
  command = "echo ${azurerm_public_ip.main.ip_address} >> deployed_ips.txt"
}
```

**You can also specify the interpreter:**

```hcl
provisioner "local-exec" {
  command     = "Get-Date"
  interpreter = ["PowerShell", "-Command"]
}
```

This is useful in Windows pipelines where the default shell is PowerShell instead of bash.

---

**3. `file` Provisioner**

This provisioner copies a file or directory from your local machine to the remote resource over SSH or WinRM.

**Use case:** Copy a config file, certificate, or script to a VM after it is created.

**Example 1 — Copy a single config file:**

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  name = "azureshop-app-vm"

  # ... other config ...

  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/etc/myapp/app.conf"

    connection {
      type        = "ssh"
      user        = "adminuser"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip_address
    }
  }
}
```

**Example 2 — Copy an entire directory:**

```hcl
provisioner "file" {
  source      = "scripts/"
  destination = "/home/adminuser/scripts"

  connection {
    type        = "ssh"
    user        = "adminuser"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip_address
  }
}
```

**Combining `file` and `remote-exec` together:**

A very common pattern is to first copy a script using `file`, then execute it using `remote-exec`:

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  name = "azureshop-app-vm"

  # ... other config ...

  # Step 1 — Copy the script to the VM
  provisioner "file" {
    source      = "scripts/install_app.sh"
    destination = "/tmp/install_app.sh"

    connection {
      type        = "ssh"
      user        = "adminuser"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip_address
    }
  }

  # Step 2 — Execute the script on the VM
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/install_app.sh",
      "/tmp/install_app.sh"
    ]

    connection {
      type        = "ssh"
      user        = "adminuser"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip_address
    }
  }
}
```

Provisioners run in the order they are defined — so `file` copies the script first, then `remote-exec` runs it.

---

**`on_failure` — What happens when a provisioner fails:**

By default, if a provisioner fails, Terraform marks the resource as **tainted** — meaning on the next `terraform apply`, it will destroy and recreate the resource from scratch.

You can change this behaviour:

```hcl
provisioner "remote-exec" {
  inline = ["sudo apt-get install -y nginx"]

  on_failure = continue   # Ignore the error and move on
  # on_failure = fail     # Default — taint the resource and stop
}
```

Use `on_failure = continue` carefully — only when the provisioner step is optional and failure is acceptable.

---

**`when` — When to run the provisioner:**

By default, provisioners run when a resource is **created**. But you can also run them when a resource is **destroyed**:

```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo 'Resource ${self.name} is being destroyed' >> destroy.log"
}
```

This is useful for cleanup tasks — like deregistering a server from a load balancer before it is destroyed.

---

**Why provisioners are considered a last resort:**

| Problem | Explanation |
|---|---|
| **Breaks idempotency** | Provisioners run every time the resource is recreated, even if the script result is already in place |
| **Hard to debug** | If a script fails, the resource gets tainted and recreated on next apply |
| **Not visible in plan** | `terraform plan` does not show what the provisioner will do — no preview |
| **Tight coupling** | Your infrastructure code becomes dependent on SSH access, script paths, and network connectivity |

**Better alternatives:**
- For VM configuration → **cloud-init** (passes a startup script to the VM at boot time)
- For software install → **Ansible** (run after Terraform creates the VMs)
- For Kubernetes resources → **Helm provider** or **Kubernetes provider**
- For Azure VMs specifically → **Azure Custom Script Extension**

---

**Summary (what to say if time is short):**

*"Terraform has three types of provisioners. `remote-exec` connects to a remote resource via SSH or WinRM and runs commands on it — like installing software on a VM. `local-exec` runs commands on the local machine where Terraform is running — like updating kubeconfig after an AKS cluster is created. `file` copies files or directories from local machine to the remote resource over SSH. All three provisioners support `on_failure` to control what happens if they fail, and a `when = destroy` option to run cleanup tasks on resource deletion. However, in practice we treat provisioners as a last resort and prefer cloud-init, Ansible, or Azure Custom Script Extension instead."*

---

#### Q6. What is Drift Detection in Terraform?

**Answer:**

This is a very important concept in real production environments. Let me explain it clearly.

---

**What is Drift?**

**Drift** means your actual cloud infrastructure no longer matches what is defined in your Terraform code and state file.

In simple words — someone changed something in the cloud **without going through Terraform**, and now Terraform's record of the world is out of sync with reality.

**Simple analogy:**

Imagine you have a blueprint of your house (Terraform code) and a record of what was built (state file). Now someone adds a room to the house without updating the blueprint. The house (actual infrastructure) is now different from the blueprint (Terraform code). That gap is called **drift**.

---

**How does drift happen in real life?**

| Scenario | What happened |
|---|---|
| A developer goes to the Azure Portal and manually increases the VM size | Terraform still thinks the old size is in place |
| Someone manually adds a firewall rule to an NSG | Terraform has no record of that rule |
| A team member deletes a resource group directly from Azure CLI | Terraform still thinks it exists |
| An Azure policy auto-applies a tag to a resource | Terraform did not add that tag |
| Someone manually scales the AKS node pool | Terraform's state shows the old node count |

All of the above cases create drift — the actual infrastructure no longer matches what Terraform expects.

---

**How does Terraform detect drift?**

Terraform detects drift using the **`terraform plan`** command.

When you run `terraform plan`, Terraform does three things:
1. Reads your `.tf` code (desired state)
2. Reads the state file (what Terraform last created)
3. Calls the Azure API to check the **actual current state** of every resource

If the actual state is different from the desired state in code, Terraform highlights it in the plan output as a change.

```bash
terraform plan
```

**Example output showing drift:**

```
~ azurerm_linux_virtual_machine.web will be updated in-place
  ~ size = "Standard_B2s" -> "Standard_D4s_v3"
```

This tells you: Terraform expects `Standard_B2s` but the actual VM in Azure is `Standard_D4s_v3`. Someone manually resized the VM. Terraform will change it back to `Standard_B2s` when you apply.

---

**`terraform refresh` — The dedicated drift detection command:**

There is also a dedicated command called `terraform refresh` which updates the state file to match the current actual state of infrastructure — without making any changes.

```bash
terraform refresh
```

What it does:
- Talks to the Azure API
- Updates the state file with the current real values
- Does NOT create, modify, or destroy anything

> Note: In modern Terraform (v0.15+), `terraform refresh` is deprecated as a standalone command. The same behaviour is now built into `terraform plan` automatically. You can also use `terraform apply -refresh-only` which is the modern replacement.

**`terraform apply -refresh-only`:**

```bash
terraform apply -refresh-only
```

This is the modern way. It:
- Reads the actual state from Azure
- Updates the state file to match reality
- Does NOT apply any changes from your `.tf` code

This is useful when you want to **accept the drift** — you want Terraform to acknowledge what exists in Azure without reverting the manual changes.

---

**Two ways to handle drift — once detected:**

**Option 1 — Fix the drift (revert manual changes):**

If someone made an unauthorized change and you want to go back to the Terraform-defined state, simply run:

```bash
terraform apply
```

Terraform will revert the manually changed resource back to what is defined in your code. This is the most common approach — it enforces infrastructure as code discipline.

**Option 2 — Accept the drift (update the code):**

If the manual change was intentional and correct — for example, someone had to increase VM size urgently during an incident — then you accept the drift by:

1. Updating the `.tf` code to match the change
2. Running `terraform plan` to confirm no more drift
3. Running `terraform apply` to sync state

```hcl
# Update the code to match what was done manually
resource "azurerm_linux_virtual_machine" "web" {
  size = "Standard_D4s_v3"   # Updated to match the manual change
}
```

---

**How we handle drift in a real team environment:**

In professional teams, drift is prevented through process and automation:

**1. No manual changes rule:**
Nobody touches the Azure Portal or CLI to change infrastructure. All changes go through Terraform and a PR review.

**2. Scheduled drift detection:**
We set up a pipeline that runs `terraform plan` every night (or every few hours). If it finds any drift (i.e., the plan is not empty), it sends an alert to the team on Slack or email.

**AzureShop Example:**

In AzureShop, we had a scheduled Azure DevOps pipeline that ran `terraform plan` every night at midnight. If the plan output showed any changes — meaning drift was detected — the pipeline would:
1. Fail with a non-zero exit code
2. Send an alert notification to the DevOps team
3. The team would investigate in the morning — find who made the manual change and either revert it or update the code

This ensured our infrastructure always stayed in sync with Terraform code.

**The pipeline command for automated drift detection:**

```bash
terraform plan -detailed-exitcode
```

The `-detailed-exitcode` flag returns:
- Exit code `0` — no changes (no drift)
- Exit code `1` — error
- Exit code `2` — changes detected (drift found)

So in the pipeline:
```bash
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  echo "DRIFT DETECTED — Infrastructure has changed outside of Terraform!"
  exit 1
fi
```

---

**Why drift detection matters:**

| Without drift detection | With drift detection |
|---|---|
| Someone makes a manual change — nobody knows | Any manual change is caught immediately |
| Terraform apply might fail or cause unexpected changes | You always know the real state before applying |
| Audit and compliance is impossible | Full audit trail — every change goes through Git and pipeline |
| Production incidents from unexpected infrastructure state | Infrastructure always matches the approved code |

---

**Summary (what to say if time is short):**

*"Drift in Terraform means the actual cloud infrastructure no longer matches what is defined in the Terraform code and state file — usually because someone made a manual change directly in the portal or CLI. Terraform detects drift automatically when you run `terraform plan` — it compares your code, the state file, and the actual Azure resources and highlights any differences. To handle drift, you either run `terraform apply` to revert unauthorized changes back to the code-defined state, or update the code to accept the change. In production, we prevent drift by enforcing a no-manual-changes policy and running scheduled `terraform plan` pipelines every night that alert the team if any drift is detected."*

---

#### Q7. How do you resolve Terraform Drift?

**Answer:**

In Q6 we talked about what drift is and how to detect it. This question goes one step further — once drift is detected, how do you actually fix it? Let me walk through the complete resolution process.

---

**First — always detect before you resolve:**

Before resolving anything, always run `terraform plan` to get a clear picture of what has drifted.

```bash
terraform plan
```

Read the output carefully:
- `~` means a resource will be **updated** — a property changed
- `-/+` means a resource will be **destroyed and recreated** — a critical change
- `-` means a resource will be **destroyed** — it was deleted manually
- `+` means a resource will be **created** — it is missing from actual infrastructure

Once you understand what drifted and why, you choose the right resolution approach.

---

**There are 3 ways to resolve drift:**

---

**Approach 1 — Revert the drift (bring infrastructure back to Terraform state)**

Use this when: Someone made an unauthorized manual change and you want to undo it and go back to what the code says.

This is the most common and recommended approach. You simply run:

```bash
terraform apply
```

Terraform compares the current actual state with your code and reverts any manual changes back to the code-defined state.

**Real example:**

Someone went to the Azure Portal and manually changed the AKS node count from 3 to 5.

Terraform plan shows:
```
~ azurerm_kubernetes_cluster.aks will be updated in-place
  ~ default_node_pool[0].node_count = 5 -> 3
```

Running `terraform apply` sets it back to 3 — exactly what the code says.

---

**Approach 2 — Accept the drift (update code to match reality)**

Use this when: The manual change was intentional, correct, and should be kept. You want Terraform to accept the new state instead of reverting it.

**Step by step:**

**Step 1 — Run `terraform apply -refresh-only`**

This updates the Terraform state file to match the actual current state of infrastructure — without changing anything in Azure.

```bash
terraform apply -refresh-only
```

Output:
```
~ azurerm_linux_virtual_machine.web
  ~ size = "Standard_B2s" -> "Standard_D4s_v3"

This plan will update the Terraform state to match the real infrastructure.
No changes to actual cloud resources will be made.

Do you want to update the Terraform state? yes
```

After this, the state file now reflects the actual current infrastructure.

**Step 2 — Update the `.tf` code to match**

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name = "azureshop-web-vm"
  size = "Standard_D4s_v3"   # Updated from Standard_B2s to match the manual change
  # ... rest of config ...
}
```

**Step 3 — Run `terraform plan` to confirm zero drift**

```bash
terraform plan
```

Expected output:
```
No changes. Your infrastructure matches the configuration.
```

This confirms the code, state, and actual infrastructure are all in sync again.

**Step 4 — Commit the updated code to Git**

```bash
git add main.tf
git commit -m "Accept VM size change from Standard_B2s to Standard_D4s_v3"
git push
```

This ensures the change is tracked, reviewed, and visible to the whole team.

---

**Approach 3 — Force recreate a drifted resource**

Use this when: A resource is so badly drifted or corrupted that the best fix is to destroy and recreate it cleanly.

**Modern way — `terraform apply -replace`:**

```bash
terraform apply -replace="azurerm_linux_virtual_machine.web"
```

This tells Terraform to destroy and recreate that specific resource, even if the plan would normally just update it. Terraform will provision a fresh, clean resource exactly as defined in your code.

**Old way (deprecated) — `terraform taint`:**

```bash
terraform taint azurerm_linux_virtual_machine.web
terraform apply
```

`terraform taint` marked a resource for forced recreation on the next apply. It is now replaced by the `-replace` flag above, but you will still see it mentioned in older documentation and projects.

---

**Handling a resource that was manually deleted:**

If someone manually deleted a resource in Azure and Terraform still has it in the state file, `terraform plan` shows:

```
+ azurerm_resource_group.secondary will be created
```

Terraform thinks it needs to create it because it no longer exists in Azure. Running `terraform apply` will recreate it. This is usually the correct action.

However, if the deletion was intentional and you do NOT want to recreate it, you remove it from the state file and the code:

**Step 1 — Remove from state:**
```bash
terraform state rm azurerm_resource_group.secondary
```

**Step 2 — Remove from `.tf` code:**
Delete the resource block from your code.

**Step 3 — Verify:**
```bash
terraform plan
# Should show: No changes.
```

---

**Handling a resource that exists in Azure but not in Terraform:**

If someone created a resource manually in Azure and it is not in Terraform at all, you use `terraform import` to bring it under Terraform management.

```bash
terraform import azurerm_resource_group.new /subscriptions/<sub-id>/resourceGroups/manually-created-rg
```

Then write the matching `.tf` code for it:

```hcl
resource "azurerm_resource_group" "new" {
  name     = "manually-created-rg"
  location = "East US"
}
```

Run `terraform plan` — it should show no changes, confirming the resource is now fully managed by Terraform.

---

**Complete drift resolution decision flow:**

```
Drift Detected (terraform plan shows changes)
         |
         |
   Was the manual change intentional?
         |
    _____|_____
   |           |
  NO           YES
   |           |
Revert it    Accept it
terraform    terraform apply -refresh-only
apply        → update .tf code
             → terraform plan (verify no drift)
             → git commit the code change
```

---

**AzureShop Real Example:**

During one of our AzureShop incidents, our on-call engineer manually increased the AKS system node pool from `Standard_D2s_v3` to `Standard_D4s_v3` to handle a traffic spike at 2 AM. This was the right call for the incident.

Next morning, our nightly drift detection pipeline flagged the change. Here is how we resolved it:

1. The team discussed — the larger VM size was actually better for production, so we decided to **accept the drift**
2. Ran `terraform apply -refresh-only` to update the state file
3. Updated `main.tf` to set `vm_size = "Standard_D4s_v3"`
4. Ran `terraform plan` — confirmed no drift
5. Raised a PR, got it reviewed and merged
6. The infrastructure change was now properly tracked in Git with full history

This is exactly how mature DevOps teams handle drift — detect it, discuss it, resolve it cleanly, and record it in version control.

---

**Summary (what to say if time is short):**

*"There are three ways to resolve Terraform drift. First — revert the drift by running `terraform apply`, which brings the infrastructure back to the code-defined state. Second — accept the drift by running `terraform apply -refresh-only` to update the state file to match reality, then updating the `.tf` code to match, and committing it to Git. Third — force recreate the drifted resource using `terraform apply -replace` when it needs to be rebuilt cleanly. If a resource was manually deleted, `terraform apply` recreates it. If a resource exists in Azure but is not in Terraform, we use `terraform import` to bring it under management. The choice depends on whether the manual change was intentional or unauthorized."*

---

#### Q8. Will you update the Terraform state file manually?

**Answer:**

**The short answer is — almost never. And when we do, we never edit the file directly.**

This is a very important question because the state file is the most sensitive file in any Terraform project. Let me explain why, and what the correct approach is.

---

**What is the state file and why is it so sensitive?**

The Terraform state file (`terraform.tfstate`) is Terraform's single source of truth. It tracks every resource Terraform has created — the resource ID, all its properties, its relationships with other resources, and metadata.

If this file gets corrupted, wrong, or out of sync — Terraform loses track of what it created. This can lead to:
- Duplicate resources being created
- Resources being destroyed that should not be
- `terraform apply` failing completely
- In the worst case — production infrastructure going down

Think of the state file like the **engine control unit (ECU) of a car**. You would never open it up and manually edit values inside it. If something is wrong, you use the right tools to fix it — not a screwdriver directly in the ECU.

---

**Why you should NEVER directly edit the state file:**

The state file is a JSON file, so technically you can open it in a text editor and change values. But you should never do this because:

1. **It is extremely easy to corrupt it** — one wrong character, a missing comma, or an incorrect resource ID and the entire state breaks
2. **No validation** — Terraform does not check if what you wrote is valid. It just reads it and acts on it. Wrong values lead to wrong actions.
3. **No audit trail** — A direct edit leaves no record of what changed, who changed it, or why
4. **State locking is bypassed** — If you edit the file directly while someone else is running Terraform, you create a conflict that corrupts the state
5. **Better tools exist** — Terraform provides proper `state` commands specifically for state management. Always use them.

---

**The right way — use `terraform state` commands:**

Terraform provides safe, purpose-built commands for every state operation you might need. These commands:
- Validate your input before making changes
- Maintain the correct JSON structure
- Work with state locking
- Leave an operation log

Here are the commands and when to use each one:

---

**1. `terraform state list` — View what is in state**

```bash
terraform state list
```

Output:
```
azurerm_resource_group.main
azurerm_kubernetes_cluster.aks
azurerm_container_registry.acr
azurerm_key_vault.kv
```

Use this to see what resources Terraform is currently tracking before making any state changes.

---

**2. `terraform state show` — View details of a specific resource**

```bash
terraform state show azurerm_kubernetes_cluster.aks
```

Shows all attributes of that resource as Terraform knows them — location, node count, resource group, etc.

Use this to verify the current values in state before and after any operation.

---

**3. `terraform state rm` — Remove a resource from state**

This removes a resource from the state file without destroying it in Azure. The resource continues to exist in Azure — Terraform just stops managing it.

```bash
terraform state rm azurerm_resource_group.secondary
```

**When to use it:**
- When a resource was intentionally deleted manually and you do not want Terraform to recreate it
- When you are moving a resource to a different Terraform project
- When refactoring — breaking one large Terraform config into smaller modules

---

**4. `terraform state mv` — Move or rename a resource in state**

This is used when you rename a resource in your `.tf` code. Without this, Terraform would destroy the old resource and create a new one — which means downtime.

**Example scenario:**
You renamed a resource block in code from `azurerm_resource_group.main` to `azurerm_resource_group.primary`.

Without `state mv`, Terraform would destroy the old RG and create a new one — disaster.

With `state mv`:
```bash
terraform state mv azurerm_resource_group.main azurerm_resource_group.primary
```

Now the state is updated to the new name, and when you run `terraform plan` it shows no changes — the resource is just renamed in Terraform, not in Azure.

**Another use case — moving a resource into a module:**
```bash
terraform state mv azurerm_kubernetes_cluster.aks module.aks.azurerm_kubernetes_cluster.main
```

---

**5. `terraform state pull` and `terraform state push` — Download and upload state**

```bash
# Download the current remote state to local stdout
terraform state pull > backup.tfstate

# Upload a local state file to the remote backend
terraform state push backup.tfstate
```

**When to use `state pull`:**
- To take a backup of the state file before doing risky operations
- To inspect the full state file in JSON format

**When to use `state push`:**
- In very rare, controlled situations — for example, restoring a known-good backup after a state corruption
- This is the closest you will ever get to "manually updating" state — but even here you are pushing a previously-valid file, not hand-editing JSON

---

**When is directly touching the state file acceptable?**

There is exactly one scenario where senior engineers do carefully edit the state file directly — **emergency state recovery after corruption**, and only after:

1. Taking a full backup first (`terraform state pull > backup.tfstate`)
2. Locking the state manually to prevent anyone else from running Terraform
3. Making the minimal possible change
4. Validating with `terraform plan` immediately after

Even in this case, most experienced engineers prefer to restore from a previous known-good backup rather than hand-editing.

---

**How we protect the state file in AzureShop:**

In our AzureShop project, the state file was stored in Azure Blob Storage with these protections:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "azureshop-tfstate-rg"
    storage_account_name = "azureshoptfstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

**Protections in place:**
- **Blob versioning enabled** — Every change to the state file creates a new version. If something goes wrong, we restore a previous version.
- **Soft delete enabled** — Accidentally deleted state file can be recovered within 30 days
- **State locking** — Azure Blob Storage uses lease-based locking. Only one `terraform apply` can run at a time.
- **RBAC access control** — Only the service principal used by the Azure DevOps pipeline has write access to the storage account. No developer can directly access or edit the state file.
- **Regular backups** — A scheduled pipeline ran `terraform state pull` every Sunday and stored the backup in a separate storage account

---

**What to do if the state file gets corrupted:**

1. **Do not panic and do not run terraform apply**
2. Pull the latest state for inspection: `terraform state pull`
3. Check Azure Blob Storage for previous versions — restore the last known-good version
4. Run `terraform plan` after restoring to verify the state is correct
5. If no backup exists, use `terraform import` to reimport each resource one by one

---

**Summary (what to say if time is short):**

*"No, I would never edit the Terraform state file directly. The state file is Terraform's single source of truth and directly editing it risks corruption, data loss, and production incidents. Instead, Terraform provides purpose-built commands — `state rm` to remove a resource from tracking, `state mv` to rename or move resources without destroying them, `state pull` to download a backup, and `state push` to restore from a backup. In AzureShop, we protected the state file by storing it in Azure Blob Storage with versioning, soft delete, state locking, and strict RBAC so only the pipeline service principal had write access — no developer could directly touch the state file."*

---

#### Q9. What is the difference between Implicit Dependency and Explicit Dependency in Terraform?

**Answer:**

This is a very good question about how Terraform understands the order in which resources should be created. Let me explain both clearly.

---

**Why does dependency matter in Terraform?**

When you create infrastructure, resources depend on each other. For example:
- A Subnet must be created **after** the Virtual Network
- An AKS cluster must be created **after** the Resource Group
- A Key Vault access policy must be created **after** the Key Vault

Terraform needs to know this order so it creates resources in the right sequence. If it tries to create a Subnet before the VNet exists, the creation will fail.

Terraform handles this in two ways — **Implicit** and **Explicit** dependencies.

---

**Implicit Dependency — Terraform figures it out automatically**

An implicit dependency is when Terraform **automatically detects** the dependency because one resource references another resource's attribute directly in the code using an expression like `resource_type.resource_name.attribute`.

You do not need to tell Terraform about the dependency — it reads the reference and understands the order by itself.

**Example:**

```hcl
# Step 1 — Create a Resource Group
resource "azurerm_resource_group" "main" {
  name     = "azureshop-rg"
  location = "East US"
}

# Step 2 — Create a Virtual Network inside that Resource Group
resource "azurerm_virtual_network" "main" {
  name                = "azureshop-vnet"
  location            = azurerm_resource_group.main.location        # Reference
  resource_group_name = azurerm_resource_group.main.name            # Reference
  address_space       = ["10.0.0.0/16"]
}
```

Here, the VNet is referencing `azurerm_resource_group.main.location` and `azurerm_resource_group.main.name`.

Terraform sees these references and automatically understands:
- "The VNet depends on the Resource Group"
- "I must create the Resource Group first, then the VNet"

No extra instruction needed — the reference itself creates the dependency. This is an **implicit dependency**.

**AzureShop Example:**

In AzureShop, almost every resource had implicit dependencies:

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  location            = azurerm_resource_group.main.location        # implicit — RG must exist first
  resource_group_name = azurerm_resource_group.main.name            # implicit — RG must exist first
  dns_prefix          = "azureshop"

  default_node_pool {
    name           = "system"
    node_count     = 3
    vm_size        = "Standard_D2s_v3"
    vnet_subnet_id = azurerm_subnet.aks.id                          # implicit — Subnet must exist first
  }
}
```

Terraform reads these references and automatically builds the correct creation order:
```
Resource Group → Virtual Network → Subnet → AKS Cluster
```

---

**Explicit Dependency — You tell Terraform manually using `depends_on`**

An explicit dependency is when Terraform **cannot automatically detect** the dependency because one resource does not directly reference another in its code — but it still needs to be created after that resource.

In this case, you manually tell Terraform about the dependency using the `depends_on` argument.

**When does this happen?**

This happens when a resource depends on the **side effect** of another resource — not its attributes. The dependency is real but not visible in the code through a reference.

**Example:**

```hcl
# Step 1 — Create a Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "azureshopstore"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# Step 2 — Run a local script that uses the storage account
# This resource does NOT reference any attribute of the storage account
# But it NEEDS the storage account to exist before it runs
resource "null_resource" "upload_config" {
  provisioner "local-exec" {
    command = "az storage blob upload --account-name azureshopstore --container-name configs --file app.conf"
  }

  depends_on = [azurerm_storage_account.main]   # Explicit dependency
}
```

Here, the `null_resource` does not reference `azurerm_storage_account.main` anywhere in its attributes. So Terraform would not know it depends on the storage account. Without `depends_on`, Terraform might try to run the script before the storage account is created — causing it to fail.

`depends_on` explicitly tells Terraform: *"Do not start this resource until the storage account is fully created."*

---

**Another real example — Role Assignment depending on a resource being ready:**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name = "azureshop-aks"
  # ... config ...
}

resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.acr.id

  depends_on = [
    azurerm_kubernetes_cluster.aks,
    azurerm_container_registry.acr
  ]
}
```

Even though there are some references here, the `depends_on` ensures both the AKS cluster and ACR are fully provisioned and ready before the role assignment is attempted — because role assignments sometimes fail if the underlying resource is not in a fully ready state yet.

---

**Side by side comparison:**

| | Implicit Dependency | Explicit Dependency |
|---|---|---|
| **How it works** | Terraform detects it automatically from references in the code | You tell Terraform manually using `depends_on` |
| **When to use** | When one resource uses an attribute of another resource | When a dependency exists but there is no direct attribute reference |
| **Code needed** | No extra code — the reference itself is enough | Must add `depends_on = [resource]` |
| **Most common?** | Yes — used in almost every Terraform config | No — used only when implicit dependency cannot be detected |
| **Example** | VNet referencing RG name and location | `null_resource` depending on a storage account it does not reference |

---

**Important rule — prefer implicit over explicit:**

HashiCorp recommends always using implicit dependencies wherever possible. Use `depends_on` only when you have no other choice.

**Why?**
- Implicit dependencies are clearer — anyone reading the code can see the relationship through the reference
- `depends_on` is a blunt tool — it blocks the entire resource creation until the dependency is complete, even if only a small part of it is needed
- Overusing `depends_on` can slow down your Terraform runs because it prevents parallel creation of resources

**Terraform creates independent resources in parallel by default.** For example, if you are creating a Storage Account and a Key Vault that do not depend on each other, Terraform creates both at the same time. Adding unnecessary `depends_on` breaks this parallelism.

---

**How Terraform visualizes the dependency graph:**

You can actually see the dependency graph Terraform builds using:

```bash
terraform graph
```

This outputs a graph in DOT format that you can visualize. It shows all implicit and explicit dependencies as arrows between resources — very useful for understanding complex infrastructure and debugging creation order issues.

---

**Summary (what to say if time is short):**

*"Implicit dependency is when Terraform automatically detects the relationship between resources because one resource directly references another's attribute in the code — like a VNet referencing a Resource Group's name and location. Terraform builds a dependency graph from these references and creates resources in the correct order automatically. Explicit dependency is when you manually tell Terraform about a dependency using `depends_on` — used when a resource depends on another but does not directly reference any of its attributes, such as a script that must run after a storage account is created but does not use any of its attributes in its own config. The best practice is to always prefer implicit dependencies and use `depends_on` only when implicit detection is not possible."*

---

#### Q10. Which keyword is used for Explicit Dependency? Where have you used `depends_on` in your project?

**Answer:**

The keyword used for explicit dependency in Terraform is **`depends_on`**.

It is a meta-argument — meaning it is not specific to any one resource type. You can use it in any resource, module, or data source block in Terraform.

---

**Basic syntax:**

```hcl
resource "resource_type" "resource_name" {
  # ... configuration ...

  depends_on = [
    resource_type.resource_name,
    resource_type.another_resource_name
  ]
}
```

`depends_on` takes a **list** — so you can declare multiple dependencies in one place. Terraform will wait for all of them to be fully created before starting this resource.

---

**Where I used `depends_on` in the AzureShop project:**

In AzureShop, I used `depends_on` in four specific situations. Let me walk through each one.

---

**Use Case 1 — ACR Pull Role Assignment for AKS**

After creating the AKS cluster and Azure Container Registry, we needed to assign the `AcrPull` role to the AKS kubelet identity so that AKS nodes could pull Docker images from ACR without needing credentials.

The problem was — even though the AKS cluster was created, the kubelet identity sometimes took a few extra seconds to be fully registered in Azure Active Directory. If the role assignment was attempted immediately, it would fail with an identity not found error.

We used `depends_on` to make sure Terraform fully completed the AKS cluster creation before attempting the role assignment:

```hcl
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.acr.id

  depends_on = [
    azurerm_kubernetes_cluster.aks,
    azurerm_container_registry.acr
  ]
}
```

Without `depends_on` here, Terraform would see that `azurerm_kubernetes_cluster.aks` is already referenced via `kubelet_identity[0].object_id` — but that implicit reference alone was not enough to prevent the race condition. The explicit `depends_on` gave the full resource enough time to stabilize.

---

**Use Case 2 — Key Vault Access Policy depending on AKS Managed Identity**

In AzureShop, our application pods needed to read secrets from Azure Key Vault. We used the AKS managed identity (pod identity) to grant access.

The Key Vault access policy needed to be created after both the Key Vault and the AKS cluster were fully ready. The access policy referenced the AKS identity's object ID — but we added `depends_on` as an extra safety guard because Azure's identity propagation across services can have a slight delay:

```hcl
resource "azurerm_key_vault_access_policy" "aks_identity" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_kubernetes_cluster.aks.identity[0].principal_id

  secret_permissions = [
    "Get",
    "List"
  ]

  depends_on = [
    azurerm_key_vault.main,
    azurerm_kubernetes_cluster.aks
  ]
}
```

---

**Use Case 3 — Diagnostic Settings depending on Log Analytics Workspace**

We configured diagnostic settings on all our Azure resources to send logs to a central Log Analytics Workspace for monitoring. The diagnostic settings resource does not always directly reference the workspace in a way Terraform can detect implicitly for creation ordering.

```hcl
resource "azurerm_monitor_diagnostic_setting" "aks_diagnostics" {
  name                       = "azureshop-aks-diagnostics"
  target_resource_id         = azurerm_kubernetes_cluster.aks.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  log {
    category = "kube-apiserver"
    enabled  = true
  }

  metric {
    category = "AllMetrics"
    enabled  = true
  }

  depends_on = [
    azurerm_kubernetes_cluster.aks,
    azurerm_log_analytics_workspace.main
  ]
}
```

This ensured the Log Analytics Workspace was fully ready to accept data before the diagnostic setting was created.

---

**Use Case 4 — Module level `depends_on`**

In AzureShop, we split our Terraform code into modules — one module for networking, one for AKS, one for monitoring. The AKS module depended on the networking module completing first.

You can use `depends_on` at the module level too:

```hcl
module "aks" {
  source              = "./modules/aks"
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = module.networking.aks_subnet_id
  acr_id              = module.acr.acr_id

  depends_on = [
    module.networking,
    module.acr
  ]
}
```

This told Terraform — do not start creating any resource inside the AKS module until both the networking module and the ACR module have fully completed.

---

**Important things to remember about `depends_on`:**

**1. It applies to the whole resource, not just one attribute:**

When you add `depends_on`, Terraform waits for the entire referenced resource to be fully created and ready — not just a specific attribute. This is more strict than an implicit dependency which only waits for the specific attribute it references.

**2. It prevents parallel creation:**

Terraform is smart — it creates independent resources in parallel to save time. Adding `depends_on` tells Terraform to stop and wait, which slows things down. So only use it when genuinely needed.

**3. `depends_on` inside a `data` source:**

You can also use `depends_on` in data sources. For example, if you are reading a resource that was just created in the same run, you need `depends_on` to make sure it exists before you try to read it:

```hcl
data "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  resource_group_name = azurerm_resource_group.main.name

  depends_on = [azurerm_kubernetes_cluster.aks]
}
```

**4. Available in resources, modules, and data sources — not in providers:**

`depends_on` works in:
- `resource` blocks ✅
- `module` blocks ✅
- `data` blocks ✅
- `provider` blocks ❌ — not supported here

---

**Summary (what to say if time is short):**

*"The keyword for explicit dependency in Terraform is `depends_on`. It is a meta-argument that takes a list of resources or modules and tells Terraform to wait for all of them to be fully created before starting this resource. In AzureShop, I used `depends_on` in four places — for the ACR Pull role assignment on AKS to handle Azure identity propagation delay, for the Key Vault access policy to ensure the AKS managed identity was fully registered, for diagnostic settings to ensure the Log Analytics Workspace was ready, and at the module level to make sure the networking module completed before the AKS module started. The key rule is — use `depends_on` only when implicit dependency cannot detect the relationship, because it prevents Terraform's parallel resource creation and slows down your runs."*

---

#### Q11. Why should infrastructure changes always be done through Terraform instead of Azure Portal?

**Answer:**

This is one of the most fundamental questions in DevOps and Infrastructure as Code. Let me explain every reason clearly with real examples.

---

**The core problem with using the Azure Portal:**

The Azure Portal is a graphical interface where you click buttons to create and manage resources. It is great for exploring and learning — but it is a terrible way to manage production infrastructure. Here is why.

Think of it like this — imagine 5 developers all decorating the same room by walking in and moving furniture around manually, at different times, with no written plan. The room ends up looking different every time and nobody knows who moved what or why. That is exactly what happens when a team manages infrastructure through the portal.

Terraform is the written plan — everyone follows the same plan, every change is recorded, and the result is always predictable.

---

**Reason 1 — No Audit Trail in the Portal**

When you make a change in the Azure Portal, you click a button and the change happens. There is no automatic record of:
- Who made the change
- What exactly was changed
- Why it was changed
- When it was changed

Yes, Azure Activity Log captures basic operation history — but it does not capture the intent, the reasoning, or the full context of a change.

**With Terraform:**

Every change goes through a Git commit and a Pull Request. The Git history gives you a complete, permanent audit trail:

```bash
git log --oneline

a3f9c21 Scale AKS node pool to 6 — peak traffic handling (approved by team lead)
b2e1d04 Add Key Vault access policy for payment service
c9d7f88 Update NSG rules — restrict port 22 to VPN CIDR only
```

Six months later, anyone on the team can look at this history and understand exactly what changed, who approved it, and why. The portal gives you none of this.

---

**Reason 2 — No Consistency — Every Environment Becomes Different**

When you create dev, staging, and production environments manually through the portal, small differences creep in over time. Someone forgets a tag here, sets a slightly different SKU there, uses a different naming convention. The environments slowly become inconsistent.

This leads to the classic problem: *"It works in staging but breaks in production"* — because staging and production are not actually identical.

**With Terraform:**

The same `.tf` code creates every environment. You just change the variables:

```bash
terraform apply -var="environment=dev"
terraform apply -var="environment=staging"
terraform apply -var="environment=prod"
```

The code is identical. The infrastructure is identical. No surprises.

**AzureShop Example:**

In AzureShop, our dev, staging, and prod environments were created from the exact same Terraform modules with different variable files (`dev.tfvars`, `staging.tfvars`, `prod.tfvars`). Every NSG rule, every subnet CIDR, every AKS configuration was identical across environments except for size and scale. This is why bugs caught in staging were always real bugs — not environment differences.

---

**Reason 3 — No Repeatability — You Cannot Recreate What You Built**

If you built your entire infrastructure by clicking through the Azure Portal, and tomorrow something catastrophic happens — a region goes down, someone deletes the resource group, a misconfiguration corrupts everything — can you recreate it?

With a portal-built environment: you have to remember every step, every setting, every configuration you clicked. It is nearly impossible to recreate it exactly. You will miss things, make mistakes, and spend days rebuilding.

**With Terraform:**

```bash
terraform apply
```

One command. Everything is recreated exactly as it was — same configuration, same settings, same dependencies, same order. Disaster recovery becomes a matter of minutes, not days.

---

**Reason 4 — No Version Control — You Cannot Roll Back**

In the Azure Portal, if you make a change and something breaks, can you go back to the previous state with a single command? No. You have to manually undo every change you made, hoping you remember exactly what you changed.

**With Terraform:**

Every change is a Git commit. Rolling back is simply checking out the previous version of the code and running `terraform apply`:

```bash
git revert HEAD
terraform apply
```

Or even more surgically:

```bash
git checkout abc123 -- main.tf    # Restore a specific file to a previous version
terraform apply
```

Infrastructure rollback becomes as simple as code rollback.

---

**Reason 5 — No Peer Review — Changes Go in Without Approval**

In the Azure Portal, anyone with Contributor access can make changes directly — no review, no approval, no second pair of eyes. A junior engineer can accidentally delete a production database with a few clicks.

**With Terraform:**

Every infrastructure change goes through a Pull Request. The team sees exactly what will change (`terraform plan` output is posted in the PR), discusses it, and approves it before it is applied. This catches mistakes before they reach production.

```
Developer writes code → Raises PR → Terraform plan runs automatically
→ Team reviews the plan → Approves → Merges to main → Terraform apply runs
```

No infrastructure change reaches production without at least one reviewer seeing it.

---

**Reason 6 — Drift is Guaranteed with Portal Changes**

As we discussed in earlier questions — any change made through the portal creates drift. Terraform no longer knows the real state of your infrastructure. The next `terraform apply` can produce unexpected results — potentially destroying or modifying things you did not intend.

**With Terraform only:**

No portal changes → No drift → `terraform plan` always produces a predictable, expected output → Infrastructure always matches code.

---

**Reason 7 — Cannot Automate Portal Changes**

A click in the Azure Portal is a manual action. You cannot automate it. You cannot schedule it. You cannot put it in a pipeline.

**With Terraform:**

Infrastructure changes are fully automated in the CI/CD pipeline:

```yaml
# Azure DevOps pipeline
- stage: TerraformPlan
  steps:
    - script: terraform plan -out=tfplan

- stage: TerraformApply
  dependsOn: TerraformPlan
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  steps:
    - script: terraform apply tfplan
```

Every merge to main automatically applies the latest infrastructure changes. No human needs to log into the portal. No human can forget a step.

---

**Reason 8 — Compliance and Security**

In regulated industries — banking, healthcare, government — you must prove that infrastructure changes went through an approved process. With portal changes, there is no proof. With Terraform:

- Every change has a PR with approvers listed
- Every change is tied to a Git commit with a timestamp and author
- Every `terraform apply` run is logged in the pipeline with who triggered it
- The code itself is the compliance evidence — auditors can see exactly what the infrastructure looks like and how it got there

---

**Reason 9 — Documentation is Always Up to Date**

With portal-managed infrastructure, documentation is always a separate task that nobody does — or does and then forgets to update. The actual infrastructure and the documentation diverge immediately.

**With Terraform:**

The code IS the documentation. The `.tf` files describe exactly what exists. They are always up to date because Terraform will not work if they are not. Any new engineer joining the team can read the Terraform code and understand the entire infrastructure in an hour.

---

**The one exception — Portal is fine for exploration:**

The Azure Portal is perfect for **learning and exploring** — trying out new services, understanding how a resource works, quick prototyping. But anything that needs to go into a real environment — dev, staging, or production — must go through Terraform.

Our rule in AzureShop was simple:
> **"Portal for learning. Terraform for everything else."**

---

**Summary table — Portal vs Terraform:**

| | Azure Portal | Terraform |
|---|---|---|
| Audit trail | Partial (Activity Log only) | Full (Git history + PR + pipeline logs) |
| Consistency across environments | ❌ Differences creep in | ✅ Identical every time |
| Repeatability | ❌ Cannot recreate exactly | ✅ One command recreates everything |
| Version control and rollback | ❌ No | ✅ Git revert and re-apply |
| Peer review before changes | ❌ No | ✅ Pull Request process |
| Drift risk | ❌ Every portal change creates drift | ✅ No drift if all changes go through Terraform |
| Automation in pipelines | ❌ Cannot automate clicks | ✅ Fully automated in CI/CD |
| Compliance evidence | ❌ Hard to prove process | ✅ PR approvals + Git history = complete evidence |
| Documentation | ❌ Always out of date | ✅ Code is always up to date |

---

**Summary (what to say if time is short):**

*"Infrastructure changes should always go through Terraform instead of the Azure Portal for several reasons. First, every Terraform change goes through Git and a Pull Request — giving a full audit trail of who changed what, when, and why. Second, the same code creates every environment identically — no configuration drift between dev, staging, and prod. Third, if something goes wrong, you can roll back with a simple git revert and terraform apply. Fourth, changes go through peer review before being applied — no one person can make unauthorized changes. Fifth, Terraform changes can be fully automated in a CI/CD pipeline — no manual steps, no human errors. And finally, portal changes create drift — Terraform loses track of reality and the next apply can produce unexpected results. Our rule in AzureShop was simple — Portal for learning and exploration, Terraform for everything that goes into a real environment."*

---

#### Q12. Difference between Terraform Resource and Data Source

**Answer:**

This is one of the most frequently asked Terraform interview questions. Many beginners confuse the two because they look similar in syntax. Let me explain both clearly with real examples.

---

**In one line:**

- A **Resource** — tells Terraform to **CREATE and MANAGE** something in the cloud
- A **Data Source** — tells Terraform to **READ and FETCH** information about something that already exists

---

**Simple real-world analogy:**

Think of it like this:

- A **Resource** is like **building a new house** — you are creating something from scratch and you are responsible for it
- A **Data Source** is like **looking up the address of an existing house** — you are not building anything, you are just getting information about something that is already there

---

**Resource — Creating and Managing Infrastructure**

When you define a `resource` block in Terraform, you are telling Terraform:

> "Create this thing, own it, track it in the state file, and manage its full lifecycle — create, update, and destroy."

Terraform creates the resource when you run `terraform apply`, tracks it in the state file, and will destroy it when you run `terraform destroy`.

**Syntax:**
```hcl
resource "<provider>_<resource_type>" "<local_name>" {
  # configuration
}
```

**Example — Create a Resource Group:**

```hcl
resource "azurerm_resource_group" "main" {
  name     = "azureshop-rg"
  location = "East US"
}
```

What Terraform does here:
- Calls the Azure API to **create** a new Resource Group called `azureshop-rg`
- Writes its details (ID, name, location) into the state file
- Owns it — `terraform destroy` will **delete** this Resource Group

**Example — Create an AKS Cluster:**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "azureshop"

  default_node_pool {
    name       = "system"
    node_count = 3
    vm_size    = "Standard_D2s_v3"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

Terraform creates this AKS cluster, manages all updates to it, and will destroy it on `terraform destroy`.

---

**Data Source — Reading Existing Infrastructure**

When you define a `data` block in Terraform, you are telling Terraform:

> "Go look up this thing that already exists somewhere, fetch its details, and make those details available for me to use in my code."

Terraform does NOT create, modify, or destroy anything with a data source. It only reads.

**Syntax:**
```hcl
data "<provider>_<resource_type>" "<local_name>" {
  # filter/lookup criteria
}
```

**Example — Read an existing Resource Group:**

```hcl
data "azurerm_resource_group" "existing" {
  name = "azureshop-rg"
}
```

What Terraform does here:
- Calls the Azure API to **look up** the Resource Group named `azureshop-rg`
- Fetches all its details (ID, location, tags)
- Makes those details available to use in your code
- Does NOT create it, does NOT track it in state, will NOT destroy it

You then use the data source output elsewhere in your code:

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location
  # ...
}
```

---

**When do you use a Data Source instead of a Resource?**

Use a data source when the infrastructure already exists and is managed by someone or something else — you just need to reference it.

**Real situations where I use data sources:**

| Situation | Why data source |
|---|---|
| Networking team manages the VNet, you manage AKS | Read the VNet/Subnet IDs to put AKS in the right network |
| Key Vault was created by the security team | Read its ID to assign access policies |
| Shared ACR is used by multiple teams | Read its ID to assign AcrPull role |
| Azure subscription details needed in code | Read the current subscription ID and tenant ID |
| An existing resource you want to reference but not own | Read it, use its attributes, leave it alone |

---

**Real examples from AzureShop:**

**Example 1 — Reading the current Azure subscription and tenant:**

```hcl
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "kv" {
  name                = "azureshop-kv"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  tenant_id           = data.azurerm_client_config.current.tenant_id   # Read from data source
  sku_name            = "standard"
}
```

`data.azurerm_client_config.current` reads the current Azure login context — tenant ID, subscription ID, object ID of the currently authenticated principal. No Azure resource is created. It is purely a lookup.

---

**Example 2 — Reading a shared VNet owned by the networking team:**

In AzureShop, our networking team managed a shared Hub VNet. Our team needed to deploy AKS into a Spoke subnet connected to that Hub. We used a data source to reference their VNet without taking ownership of it:

```hcl
# Read the shared VNet managed by the networking team
data "azurerm_virtual_network" "hub" {
  name                = "azureshop-hub-vnet"
  resource_group_name = "azureshop-networking-rg"
}

# Read the specific subnet inside that VNet
data "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  virtual_network_name = data.azurerm_virtual_network.hub.name
  resource_group_name  = "azureshop-networking-rg"
}

# Use the subnet ID when creating AKS
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  dns_prefix          = "azureshop"

  default_node_pool {
    name           = "system"
    node_count     = 3
    vm_size        = "Standard_D2s_v3"
    vnet_subnet_id = data.azurerm_subnet.aks.id   # Pulled from data source
  }
}
```

The VNet and Subnet belong to the networking team — we only reference them. If we used a `resource` block instead of `data`, our `terraform destroy` would delete their VNet — a disaster.

---

**Example 3 — Reading an existing ACR owned by a shared platform team:**

```hcl
data "azurerm_container_registry" "shared_acr" {
  name                = "azureshopacr"
  resource_group_name = "azureshop-platform-rg"
}

resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = data.azurerm_container_registry.shared_acr.id   # From data source
}
```

---

**How they appear in `terraform plan` output:**

This is a very useful thing to know:

- **Resources** appear in the plan output as changes to be made — `+` create, `~` update, `-` destroy
- **Data sources** appear in the plan output with a `data.` prefix and a `(read)` annotation — they are refreshed/read but never shown as changes

```
data.azurerm_resource_group.existing: Reading...
data.azurerm_resource_group.existing: Read complete

  # azurerm_kubernetes_cluster.aks will be created
  + resource "azurerm_kubernetes_cluster" "aks" {
      + name = "azureshop-aks"
      ...
    }
```

---

**Are data sources tracked in the state file?**

**Partially — yes, but differently from resources.**

Data sources are stored in the state file so Terraform can detect if they have changed between runs. But the key difference is:

- **Resources** in state → Terraform manages their lifecycle (create/update/destroy)
- **Data sources** in state → Terraform only caches the last-read values. On the next `terraform plan`, Terraform re-reads the data source from the cloud to get fresh values.

Terraform will never destroy a data source through `terraform destroy` — because it does not own it.

---

**Complete side by side comparison:**

| | Resource | Data Source |
|---|---|---|
| **Keyword** | `resource` | `data` |
| **Purpose** | Create and manage infrastructure | Read existing infrastructure |
| **Creates anything?** | ✅ Yes | ❌ No |
| **Modifies anything?** | ✅ Yes | ❌ No |
| **Destroys on `terraform destroy`?** | ✅ Yes | ❌ No — never owns or destroys |
| **Tracked in state file?** | ✅ Yes — full lifecycle tracking | Partially — cached for reference only |
| **Shows in `terraform plan`?** | ✅ Yes — as `+`, `~`, `-` changes | ✅ Yes — as `(read)` only |
| **When to use** | When you need to create something | When you need info about something that already exists |

---

**Summary (what to say if time is short):**

*"A Resource tells Terraform to create, manage, and own a piece of infrastructure — it creates it on apply, updates it when the code changes, and destroys it on terraform destroy. It is fully tracked in the state file. A Data Source tells Terraform to read information about something that already exists — it makes no changes, creates nothing, and destroys nothing. It is just a lookup. In AzureShop, I used data sources extensively to reference infrastructure owned by other teams — like reading the Hub VNet managed by the networking team to get the Subnet ID for AKS, or reading the shared ACR to get its ID for the AcrPull role assignment. The rule is simple — if Terraform should own it, use a resource. If it already exists and you just need to reference it, use a data source."*

---

#### Q13. Why is using a Data Source better than hardcoding the Resource Group name?

**Answer:**

This is a very practical question that tests whether you write clean, maintainable, real-world Terraform code. Let me explain the problem with hardcoding first, and then show exactly why a data source is the better approach.

---

**What does hardcoding mean?**

Hardcoding means writing a fixed, literal value directly in multiple places in your code — instead of reading it from a single source of truth.

**Example of hardcoded Resource Group name:**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  resource_group_name = "azureshop-rg"       # hardcoded
  location            = "East US"             # hardcoded
}

resource "azurerm_container_registry" "acr" {
  name                = "azureshopacr"
  resource_group_name = "azureshop-rg"       # hardcoded again
  location            = "East US"             # hardcoded again
}

resource "azurerm_key_vault" "kv" {
  name                = "azureshop-kv"
  resource_group_name = "azureshop-rg"       # hardcoded again
  location            = "East US"             # hardcoded again
}
```

The string `"azureshop-rg"` and `"East US"` are repeated everywhere. This looks simple at first — but it creates serious problems.

---

**Problem 1 — One name change breaks everything**

Imagine tomorrow your team decides to rename the Resource Group from `azureshop-rg` to `azureshop-prod-rg` to follow a new naming convention.

With hardcoded values, you have to find every single place in every single `.tf` file where `"azureshop-rg"` appears and update it manually. In a large project with 20+ `.tf` files, this is tedious and error-prone.

Miss even one — and `terraform plan` shows an error or tries to create a resource in a Resource Group that does not exist.

**With a data source — you change the name in exactly one place:**

```hcl
data "azurerm_resource_group" "main" {
  name = "azureshop-prod-rg"    # Change here ONCE
}
```

Every resource that references `data.azurerm_resource_group.main.name` and `data.azurerm_resource_group.main.location` picks up the new value automatically. No hunting through files.

---

**Problem 2 — Hardcoded values give you only the name — data sources give you everything**

When you hardcode `"azureshop-rg"`, all you have is the name string.

But a Resource Group has many useful attributes — its ID, its location, its tags. To reference the location you have to hardcode that separately too:

```hcl
# Hardcoded — you have to maintain BOTH values separately
resource_group_name = "azureshop-rg"
location            = "East US"
```

If someone changes the location of the Resource Group in Azure, your hardcoded `"East US"` is now wrong — and Terraform will try to fix the mismatch.

**With a data source — you get all attributes for free:**

```hcl
data "azurerm_resource_group" "main" {
  name = "azureshop-rg"
}

resource "azurerm_kubernetes_cluster" "aks" {
  resource_group_name = data.azurerm_resource_group.main.name       # "azureshop-rg"
  location            = data.azurerm_resource_group.main.location   # "eastus" — read live from Azure
}
```

The location is read directly from Azure — whatever the Resource Group's actual location is. You never have to hardcode or manually maintain it.

---

**Problem 3 — Hardcoding breaks reusability across environments**

If you have multiple environments — dev, staging, prod — each has its own Resource Group:

```
azureshop-dev-rg
azureshop-staging-rg
azureshop-prod-rg
```

With hardcoded values, you either need separate `.tf` files for each environment (duplicated code) or you constantly edit the hardcoded name before every deployment. Both are bad.

**With a data source and a variable — one codebase, all environments:**

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

data "azurerm_resource_group" "main" {
  name = "azureshop-${var.environment}-rg"
}
```

Now the same code works for all three environments:

```bash
terraform apply -var="environment=dev"      # reads azureshop-dev-rg
terraform apply -var="environment=staging"  # reads azureshop-staging-rg
terraform apply -var="environment=prod"     # reads azureshop-prod-rg
```

One codebase. Zero duplication. Fully reusable.

---

**Problem 4 — Hardcoding is a security and compliance risk**

In real projects, Terraform code is stored in Git and reviewed by the team. Hardcoded resource names, IDs, or sensitive values scattered through dozens of files make it harder to audit and easier to miss a stale or wrong value.

When values come from data sources and variables, there is a single place to look — your `variables.tf` and `data.tf` files. Clean, reviewable, auditable.

---

**Problem 5 — Hardcoded location can mismatch Azure's actual format**

Azure uses specific internal location identifiers. For example:

| What you type | What Azure actually uses |
|---|---|
| `"East US"` | `"eastus"` |
| `"West Europe"` | `"westeurope"` |
| `"UK South"` | `"uksouth"` |

If you hardcode `"East US"` and the Resource Group was created with `"eastus"`, Terraform may detect a mismatch and try to update the location — or fail entirely. Azure does not allow location changes on existing resources.

**With a data source — you always get the exact format Azure uses:**

```hcl
data "azurerm_resource_group" "main" {
  name = "azureshop-rg"
}

# data.azurerm_resource_group.main.location returns "eastus" — exactly as Azure stores it
# No mismatch, no format guessing
```

---

**The right way — full clean example:**

```hcl
# data.tf — single file for all data sources
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

# variables.tf
variable "resource_group_name" {
  description = "Name of the existing Resource Group"
  type        = string
}

# main.tf — all resources reference the data source
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "azureshop-aks"
  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location
}

resource "azurerm_container_registry" "acr" {
  name                = "azureshopacr"
  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location
}

resource "azurerm_key_vault" "kv" {
  name                = "azureshop-kv"
  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location
}
```

Now:
- The Resource Group name is defined in one variable
- Every resource reads the name and location from the data source
- Change the variable value once → all 10 resources update automatically
- Location is always in Azure's exact format — no mismatches

---

**AzureShop Real Example:**

In AzureShop we had over 15 resources spread across 8 `.tf` files — AKS, ACR, Key Vault, VNet, Subnets, NSGs, monitoring, diagnostic settings. Every single one needed the Resource Group name and location.

Early in the project, a team member had hardcoded `"azureshop-rg"` in several files. When we renamed the RG to `"azureshop-prod-rg"` during our environment structuring sprint, we spent 30 minutes hunting through files to fix every hardcoded reference — and still missed one, which caused a pipeline failure.

After that, we moved everything to a single data source in `data.tf` and a variable in `variables.tf`. From that point on, the Resource Group name existed in exactly one place. The same issue would have taken 10 seconds to fix — just update the variable value.

---

**Summary — 5 reasons data source beats hardcoding:**

| Reason | Hardcoded | Data Source |
|---|---|---|
| Name change | Update in 20+ places | Update in 1 place |
| Get location automatically | Must hardcode separately | Comes free from the data source |
| Reuse across environments | Impossible without duplication | Pass as a variable — one codebase |
| Location format mismatch | Risk of `"East US"` vs `"eastus"` | Always returns exact Azure format |
| Code clarity and maintainability | Scattered magic strings | Single source of truth |

---

**Summary (what to say if time is short):**

*"Using a data source is better than hardcoding the Resource Group name for several reasons. First — maintainability. If the name changes, you update it in one place — the data source definition — and every resource that references it picks up the change automatically. With hardcoding, you hunt through every file and risk missing one. Second — you get all attributes for free. The data source gives you the name, location, ID, and tags without hardcoding each one separately. Third — the location returned by the data source is always in Azure's exact internal format like `eastus`, avoiding mismatches that can cause Terraform to fail. Fourth — combined with a variable, the same codebase works across all environments by just passing a different resource group name. In AzureShop, we learned this the hard way when a rename caused a pipeline failure because of scattered hardcoded values — after that, everything went through a data source and a variable."*

---

#### Q14. You're reviewing a Terraform Pull Request. How would you review it?

**Answer:**

Reviewing a Terraform PR is very different from reviewing application code. You are not just checking if the code looks clean — you are checking if the infrastructure change is safe, secure, correct, and will not cause production issues. A wrong approval here can take down a production environment.

Let me walk through exactly how I review a Terraform PR — step by step, in the same order I actually follow.

---

**Step 1 — Read the PR description first**

Before looking at a single line of code, I read the PR description carefully. A good Terraform PR description should answer:

- **What** is being changed and why
- **Which environment** this targets — dev, staging, or prod
- **Is there a `terraform plan` output** attached — so I can see what will actually happen in Azure
- **Is there a ticket or requirement** that this change is linked to
- **Any risks or dependencies** the author has flagged

If the PR description is empty or vague, that is already a red flag. I ask the author to fill it in before I review the code.

---

**Step 2 — Review the `terraform plan` output — this is the most important step**

The plan output tells me exactly what will happen in Azure when this PR is merged and applied. This is more important than the code itself.

I look for:

**`+` Create — new resources being added:**
```
+ azurerm_subnet.app_gateway will be created
```
Is this expected? Does it make sense based on the PR description?

**`~` Update — existing resources being modified:**
```
~ azurerm_kubernetes_cluster.aks will be updated in-place
  ~ default_node_pool[0].node_count = 3 -> 5
```
Is this in-place or will it force a destroy and recreate? An in-place update is safe. A destroy-and-recreate on AKS means downtime.

**`-/+` Destroy and recreate — the most dangerous:**
```
-/+ azurerm_kubernetes_cluster.aks must be replaced
  ~ default_node_pool[0].vm_size = "Standard_D2s_v3" -> "Standard_D4s_v3" # forces replacement
```

This means the AKS cluster will be **destroyed and recreated**. That is major downtime. I stop here, flag it in the PR, and ask the author:
- Is this intentional?
- Have you considered the downtime impact?
- Is there a safer way to do this change — like adding a new node pool first and removing the old one?

**`-` Destroy — resources being deleted:**
```
- azurerm_storage_account.old will be destroyed
```
Is this intentional? Is the data in this storage account backed up? Once destroyed, the data is gone.

---

**Step 3 — Check for hardcoded values**

Hardcoded values are one of the most common issues in Terraform PRs. I look for:

**Bad — hardcoded everywhere:**
```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  resource_group_name = "azureshop-rg"    # hardcoded
  location            = "East US"          # hardcoded
  node_count          = 3                  # hardcoded
}
```

**Good — using variables and data sources:**
```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  resource_group_name = data.azurerm_resource_group.main.name
  location            = data.azurerm_resource_group.main.location
  node_count          = var.node_count
}
```

Hardcoded values break reusability, make the code fragile, and create maintenance nightmares. I always flag these in the review.

---

**Step 4 — Check for secrets and sensitive values**

This is a security-critical check. I scan the PR for any secrets, passwords, keys, or tokens hardcoded in the code:

**Immediate rejection — never allowed:**
```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  windows_profile {
    admin_password = "MySecretPassword123!"    # NEVER hardcode passwords
  }
}
```

Secrets in Terraform code end up in:
1. The Git repository — visible to anyone with repo access
2. The Terraform state file — stored in plain text in the state file

**The correct approach:**
```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  windows_profile {
    admin_password = var.windows_admin_password    # Use a variable
  }
}

# And in variables.tf:
variable "windows_admin_password" {
  type      = string
  sensitive = true    # Marks this variable as sensitive — hides value in plan output
}
```

The value should be passed through a pipeline secret variable or fetched from Azure Key Vault — never hardcoded.

I also check that sensitive outputs are marked correctly:

```hcl
output "db_connection_string" {
  value     = azurerm_postgresql_server.main.connection_string
  sensitive = true    # Must be marked sensitive — otherwise printed in plain text in logs
}
```

---

**Step 5 — Check resource naming conventions**

Every team has a naming convention. In AzureShop ours was:

```
{project}-{resource-type}-{environment}
azureshop-aks-prod
azureshop-acr-prod
azureshop-kv-prod
```

I check if the new resources follow the agreed naming convention. Inconsistent names cause confusion and make it hard to identify resources in the Azure Portal.

---

**Step 6 — Check if required tags are present**

In AzureShop, we had a mandatory tagging policy — every resource must have these tags:

```hcl
tags = {
  environment = var.environment
  project     = "azureshop"
  owner       = "devops-team"
  cost-center = var.cost_center
}
```

Tags are used for:
- Cost management — identifying which team's resources are costing what
- Compliance — proving resources are properly labelled for audit
- Operations — finding resources quickly in the portal

If a new resource block is missing required tags, I flag it.

---

**Step 7 — Check provider version pinning**

Provider versions should always be pinned to a specific version or a constrained range. Unpinned providers can auto-upgrade and introduce breaking changes:

**Bad — unpinned:**
```hcl
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
    }
  }
}
```

**Good — pinned to a range:**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.75"    # Allows patch updates within 3.x, never jumps to 4.x
    }
  }
}
```

---

**Step 8 — Check for `depends_on` usage**

If the PR adds `depends_on`, I ask:
- Is this explicit dependency really needed, or can it be replaced by an implicit reference?
- If it is needed, is the reason documented in a comment?

Unnecessary `depends_on` slows down Terraform by breaking parallelism.

---

**Step 9 — Check if data sources are used where they should be**

If a resource already exists in Azure and is managed by another team, it should be a `data` source — not a `resource` block. A `resource` block on something that already exists will either fail or try to replace it.

```hcl
# Wrong — if this VNet already exists, Terraform will try to create a duplicate
resource "azurerm_virtual_network" "hub" {
  name = "existing-hub-vnet"
}

# Correct — read it, do not own it
data "azurerm_virtual_network" "hub" {
  name                = "existing-hub-vnet"
  resource_group_name = "networking-rg"
}
```

---

**Step 10 — Check the scope of role assignments**

Role assignments are a common source of security issues. I always check:
- Is the scope the **minimum necessary** — resource level, not subscription level?
- Is the role the **least privileged** — `Reader` instead of `Contributor` wherever possible?

```hcl
# Bad — too broad scope
resource "azurerm_role_assignment" "aks_acr" {
  scope                = "/subscriptions/${var.subscription_id}"   # Entire subscription — too broad
  role_definition_name = "Contributor"                              # Too powerful
}

# Good — minimal scope, minimal privilege
resource "azurerm_role_assignment" "aks_acr" {
  scope                = azurerm_container_registry.acr.id         # Only the ACR
  role_definition_name = "AcrPull"                                  # Only pull permission
}
```

---

**Step 11 — Check `terraform fmt` compliance**

The code should be properly formatted. In our pipeline, `terraform fmt -check` runs automatically and fails the PR if the code is not formatted. But I also eyeball it during review — inconsistent indentation and spacing are a sign of rushed work.

---

**Step 12 — Check if `terraform validate` passed in the pipeline**

A properly set up pipeline should run `terraform validate` automatically on every PR. I check the pipeline run attached to the PR to confirm it passed. If it did not, the PR should not even be reviewed until the author fixes the validation error.

---

**My complete Terraform PR review checklist:**

```
PR Description
  [ ] PR description explains what is changing and why
  [ ] Environment is clearly stated (dev / staging / prod)
  [ ] terraform plan output is attached

Plan Output
  [ ] No unexpected destroys (-) or destroy-recreate (-/+)
  [ ] All changes match what the PR description says
  [ ] No resources affected that are not mentioned in the PR

Code Quality
  [ ] No hardcoded values — variables and data sources used
  [ ] No secrets or passwords in code or variables without sensitive = true
  [ ] Naming convention followed for all new resources
  [ ] Required tags present on all new resources
  [ ] Provider version pinned correctly

Security
  [ ] No credentials hardcoded
  [ ] Sensitive outputs marked with sensitive = true
  [ ] Role assignments use minimum scope and minimum privilege
  [ ] No overly broad permissions (Contributor at subscription scope)

Terraform Best Practices
  [ ] Data sources used for existing resources (not resource blocks)
  [ ] depends_on used only when truly necessary
  [ ] terraform fmt passes — code is properly formatted
  [ ] terraform validate passes in pipeline
  [ ] State file changes are safe — no risky state operations without discussion
```

---

**AzureShop Example — a real PR I rejected:**

In AzureShop, a developer raised a PR to add a new service's identity to Key Vault. The plan output looked fine at first glance. But during my review I caught three issues:

1. The role assignment scope was set to the entire Resource Group instead of just the Key Vault
2. A new `azurerm_virtual_network` resource block was added for a VNet that already existed — this would have tried to create a duplicate VNet and failed
3. The new resources had no tags

I rejected the PR with clear inline comments on each issue. The developer fixed all three and re-raised. The second version was approved and merged cleanly.

This is exactly why PR reviews matter — the plan output alone would not have caught the scoping issue or the wrong resource vs data source mistake.

---

**Summary (what to say if time is short):**

*"When reviewing a Terraform PR, I follow a structured checklist. First I read the PR description to understand what is changing and why, and check that a `terraform plan` output is attached. Then I review the plan carefully — any `-/+` destroy-and-recreate or unexpected `-` destroy operations are a red flag that I investigate before approving. In the code, I check for hardcoded values that should be variables or data sources, secrets that should be marked `sensitive`, correct naming conventions, required tags on all new resources, and pinned provider versions. For security, I verify role assignments use the minimum scope and least privilege. I also check that existing resources are referenced as data sources and not accidentally recreated as resource blocks. Finally, I confirm the pipeline shows `terraform fmt` and `terraform validate` passed. In AzureShop, this checklist caught a scoping bug in a role assignment and a duplicate VNet resource block before they reached production."*

---

#### Q15. Which tools would you use to validate Terraform code before approval?

**Answer:**

Validating Terraform code before it is approved and applied is critical. A mistake in Terraform code does not just fail a build — it can destroy production infrastructure. That is why we use multiple layers of validation tools, each catching a different type of problem.

I think of it as a **defence-in-depth approach** — each tool acts as one layer of protection. If one layer misses something, the next catches it.

Let me explain each tool, what it catches, and how we used it in AzureShop.

---

**Layer 1 — `terraform fmt` — Code Formatting**

**What it is:** Built into Terraform. Checks that all `.tf` files follow the standard Terraform formatting style — correct indentation, alignment, spacing.

**What it catches:** Formatting inconsistencies, messy indentation, misaligned equals signs.

**What it does NOT catch:** Logic errors, security issues, wrong resource configurations.

```bash
# Check if formatting is correct — exits with error code 1 if not formatted
terraform fmt -check -recursive

# Auto-fix formatting
terraform fmt -recursive
```

**In our pipeline:**

```yaml
- script: terraform fmt -check -recursive
  displayName: "Terraform Format Check"
```

If this fails, the PR pipeline fails immediately. The developer must run `terraform fmt -recursive` locally and push the fix before the pipeline will proceed. No exceptions.

**Why it matters:** Consistent formatting makes code reviews faster. Reviewers focus on logic, not style.

---

**Layer 2 — `terraform validate` — Syntax and Logic Validation**

**What it is:** Built into Terraform. Validates the syntax and internal logic of your `.tf` files without connecting to any cloud provider.

**What it catches:**
- Syntax errors — missing brackets, wrong argument names
- Invalid resource configurations — wrong argument types
- References to variables or resources that do not exist in the code
- Missing required arguments

**What it does NOT catch:** Whether the resources actually exist in Azure, security misconfigurations, wrong values.

```bash
terraform init -backend=false    # Init without connecting to backend
terraform validate
```

**In our pipeline:**

```yaml
- script: |
    terraform init -backend=false
    terraform validate
  displayName: "Terraform Validate"
```

`-backend=false` skips the remote state connection — useful in PR validation where you do not want to touch the real state file.

---

**Layer 3 — `terraform plan` — The Most Important Validation**

**What it is:** Built into Terraform. Connects to Azure, reads the actual current state, and shows exactly what will be created, changed, or destroyed.

**What it catches:**
- Everything that will actually happen in Azure
- Unexpected destroys or destroy-and-recreates
- Drift between code and actual infrastructure
- Missing permissions (if the service principal cannot create a resource, plan fails here)

```bash
terraform plan -out=tfplan
```

**In our pipeline — plan on every PR:**

```yaml
- stage: Plan
  jobs:
    - job: TerraformPlan
      steps:
        - script: terraform init
        - script: terraform plan -out=tfplan -detailed-exitcode
        - task: PublishPipelineArtifact@1
          inputs:
            targetPath: tfplan
            artifact: terraform-plan
```

The plan output is published as a pipeline artifact and posted as a comment on the PR — so every reviewer can see exactly what will change in Azure before approving.

This is the single most important validation step. The first three layers check the code. This layer checks what the code will actually do to real infrastructure.

---

**Layer 4 — `tflint` — Linting and Best Practices**

**What it is:** An open-source Terraform linter. Goes beyond `terraform validate` to catch issues that are syntactically valid but logically wrong or against best practices.

**What it catches:**
- Deprecated resource arguments that still work but will be removed in future provider versions
- Invalid values for arguments — like an invalid Azure VM size that Terraform will accept but Azure will reject
- Unused variables and outputs
- Missing required tags (with custom rules)
- Naming convention violations (with custom rules)
- Azure-specific best practice violations (with the `tflint-ruleset-azurerm` plugin)

```bash
# Install tflint
brew install tflint

# Initialize with Azure ruleset plugin
tflint --init

# Run
tflint --recursive
```

**`.tflint.hcl` configuration in AzureShop:**

```hcl
plugin "azurerm" {
  enabled = true
  version = "0.25.1"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}

rule "azurerm_resource_group_missing_tags" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
}
```

**In our pipeline:**

```yaml
- script: |
    tflint --init
    tflint --recursive
  displayName: "TFLint"
```

---

**Layer 5 — `checkov` — Security and Compliance Scanning**

**What it is:** An open-source static analysis security tool for Terraform (and other IaC tools). Made by Bridgecrew. It scans your Terraform code against hundreds of security best practice rules.

**What it catches:**
- Storage accounts with public access enabled
- Key Vaults without soft delete or purge protection
- NSGs with port 22 or 3389 open to `0.0.0.0/0` (the entire internet)
- AKS clusters without RBAC enabled
- Resources without required tags
- Unencrypted disks or databases
- Missing audit logging on sensitive services
- Overly permissive role assignments

```bash
# Install
pip install checkov

# Run against Terraform directory
checkov -d . --framework terraform
```

**Example output:**

```
Check: CKV_AZURE_35: "Ensure default action is deny in azurerm_storage_account"
  PASSED for resource: azurerm_storage_account.tfstate

Check: CKV_AZURE_33: "Ensure storage account is using the latest minimum TLS version"
  FAILED for resource: azurerm_storage_account.logs
  File: /main.tf:45

Check: CKV_AZURE_9: "Ensure that RDP access is restricted from the internet"
  PASSED for resource: azurerm_network_security_group.main
```

**In our pipeline:**

```yaml
- script: checkov -d . --framework terraform --output cli --output junitxml --output-file-path . --soft-fail
  displayName: "Checkov Security Scan"

- task: PublishTestResults@2
  inputs:
    testResultsFormat: "JUnit"
    testResultsFiles: "results_junitxml.xml"
```

`--soft-fail` means the pipeline does not hard-fail on checkov findings — instead they are published as test results for the reviewer to see. In mature teams, specific high-severity checks are set to hard-fail.

**AzureShop example — Checkov caught this in a PR:**

A developer added an NSG rule that opened port 443 to `0.0.0.0/0`. This was intentional for a public-facing load balancer. Checkov flagged it. The reviewer confirmed it was intentional and added a `#checkov:skip=CKV_AZURE_10:Public HTTPS access intentional for ALB` comment to suppress the specific rule for that resource.

---

**Layer 6 — `terrascan` — Another Security Scanner**

**What it is:** Another open-source security scanner for Terraform. Similar to Checkov but with a different rule set. We sometimes used both together because they catch slightly different issues.

```bash
# Install
brew install terrascan

# Run
terrascan scan -i terraform -d .
```

Some teams use Checkov, some use Terrascan, some use both. The choice depends on which ruleset better matches your compliance requirements.

---

**Layer 7 — `infracost` — Cost Estimation**

**What it is:** An open-source tool that estimates the monthly cost of your infrastructure changes based on the `terraform plan` output.

**What it catches:** Expensive resources being added — like accidentally choosing a very large VM size or enabling geo-replication on a storage account when it is not needed.

```bash
# Install
brew install infracost

# Generate cost estimate from plan
infracost breakdown --path .
```

**Example output posted as a PR comment:**

```
Monthly cost estimate

Project: azureshop

+ azurerm_kubernetes_cluster.aks
  + System node pool (3 × Standard_D4s_v3)    +$380.00/month

  Monthly total: +$380.00
  Previous total: $210.00
  Difference: +$170.00/month
```

This tells reviewers: this PR increases our monthly Azure bill by $170. Is that expected? In AzureShop, we had infracost post a comment on every PR automatically — it made cost-awareness part of the review process, not an afterthought.

---

**Layer 8 — `Sentinel` — Policy as Code (for Enterprise teams)**

**What it is:** HashiCorp's policy-as-code framework. Available in Terraform Cloud and Terraform Enterprise. Lets you write policies in code that the Terraform plan must pass before `apply` is allowed.

**What it catches:** Any policy violation you define — like "no resource can be created outside the approved Azure regions" or "all storage accounts must have encryption enabled" or "AKS clusters must have at least 3 nodes in production."

```hcl
# Example Sentinel policy — enforce minimum AKS node count in prod
import "tfplan/v2" as tfplan

aks_clusters = filter tfplan.resource_changes as _, rc {
  rc.type is "azurerm_kubernetes_cluster" and
  rc.change.actions contains "create"
}

main = rule {
  all aks_clusters as _, cluster {
    cluster.change.after.default_node_pool[0].node_count >= 3
  }
}
```

If this policy fails, `terraform apply` is blocked — no human can override it. This is enterprise-grade guardrail enforcement.

---

**How all these tools fit together in the AzureShop pipeline:**

```
Developer pushes code to feature branch
          ↓
PR is raised
          ↓
Pipeline triggers automatically:

  Stage 1 — Code Quality
    ├── terraform fmt -check       (formatting)
    ├── terraform validate         (syntax)
    └── tflint                     (linting + best practices)

  Stage 2 — Security
    ├── checkov                    (security scan)
    └── terrascan                  (additional security rules)

  Stage 3 — Plan
    ├── terraform plan -out=tfplan (what will change in Azure)
    └── infracost breakdown        (cost impact)

  All results posted as PR comments
          ↓
Reviewer sees: fmt ✅ validate ✅ tflint ✅ checkov ✅ plan output ✅ cost impact ✅
          ↓
Reviewer approves → PR merges to main
          ↓
  Stage 4 — Apply (only on main branch)
    └── terraform apply tfplan
```

Every stage must pass before the next starts. If any check fails, the PR is blocked and the developer must fix the issue.

---

**Quick summary of all tools:**

| Tool | Type | What it catches | Built-in? |
|---|---|---|---|
| `terraform fmt` | Formatter | Code style and formatting | ✅ Yes |
| `terraform validate` | Validator | Syntax and logic errors | ✅ Yes |
| `terraform plan` | Planner | Real impact on infrastructure | ✅ Yes |
| `tflint` | Linter | Best practices, invalid values, deprecated args | ❌ Third-party |
| `checkov` | Security scanner | Security misconfigurations | ❌ Third-party |
| `terrascan` | Security scanner | Additional security rules | ❌ Third-party |
| `infracost` | Cost estimator | Cost impact of changes | ❌ Third-party |
| `Sentinel` | Policy engine | Enterprise policy enforcement | ❌ Terraform Cloud/Enterprise |

---

**Summary (what to say if time is short):**

*"I use multiple layers of tools to validate Terraform code before approval. The built-in tools run first — `terraform fmt` checks formatting, `terraform validate` catches syntax errors, and `terraform plan` shows exactly what will change in real infrastructure. On top of those, I use `tflint` with the AzureRM ruleset to catch best practice violations and deprecated configurations. For security, I use `checkov` to scan for misconfigurations like open NSG rules, missing encryption, or public storage access. I use `infracost` to show the cost impact of the change as a PR comment so reviewers know if the change adds significant Azure spend. In enterprise environments I have also used HashiCorp Sentinel for policy-as-code enforcement. In AzureShop, all of these ran automatically in the Azure DevOps pipeline on every PR — the plan output and all scan results were posted as PR comments, so the reviewer had the full picture before approving."*

---

<!-- Add more questions as Q16, Q17... -->

---

### Scenario Based Questions

> These are real scenario questions asked by the interviewer to test how you think and handle real-world problems. The interviewer is not just looking for a correct answer — they want to see your thought process, your steps, and your decision making.

---

#### Scenario 1. Someone manually changed an EC2 instance type in the AWS Console. Terraform detects drift. What will you do?

**Answer:**

This is a very practical scenario that happens in real teams. Let me walk through exactly what I would do — step by step — and explain my thinking at each stage.

---

**Step 1 — Stay calm and do NOT immediately run `terraform apply`**

The first and most important thing is — do not panic and do not blindly apply. Running `terraform apply` immediately would revert the instance type back to what Terraform expects — but I do not yet know if that change was intentional or unauthorized. I need to investigate first.

---

**Step 2 — Confirm what the drift actually is**

Run `terraform plan` to see the exact drift:

```bash
terraform plan
```

Output will look something like:

```
~ aws_instance.web will be updated in-place
  ~ instance_type = "t3.xlarge" -> "t3.medium"
```

This tells me:
- The actual EC2 in AWS is currently `t3.xlarge`
- Terraform's code says it should be `t3.medium`
- Someone upgraded the instance type from `t3.medium` to `t3.xlarge` manually

Now I know exactly what changed. I do not know yet **why** it was changed.

---

**Step 3 — Investigate who made the change and why**

Before doing anything, I find out the context behind the change.

**Check AWS CloudTrail:**

AWS CloudTrail logs every API action — including who changed the instance type, from where, and when.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-0abc123xyz \
  --start-time 2026-07-17T00:00:00Z \
  --end-time 2026-07-18T23:59:59Z
```

This gives me:
- **Who** changed it — which IAM user or role
- **When** it was changed
- **What** the change was

With this information I can have an informed conversation with the team.

---

**Step 4 — Decision — Was the change intentional or unauthorized?**

This is the key decision point. Two possible outcomes:

---

**If the change was UNAUTHORIZED (someone made a mistake or bypassed the process):**

Revert the drift — run `terraform apply` to bring the instance back to `t3.medium` as defined in the code:

```bash
terraform apply
```

Then have a conversation with the team:
- Enforce the **no manual changes** policy
- Restrict IAM permissions so developers cannot modify instance types directly
- Document the incident

---

**If the change was INTENTIONAL (for example — an on-call engineer scaled up during a production incident at 2 AM):**

Accept the drift and update the Terraform code to match the new instance type. Here is the full step-by-step:

**Step 4a — Update the state file to match reality:**

```bash
terraform apply -refresh-only
```

This updates the state file to reflect the actual current instance type (`t3.xlarge`) without changing anything in AWS.

**Step 4b — Update the Terraform code:**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.xlarge"    # Updated from t3.medium — scaled up during incident on 18-07-2026
  # ... rest of config ...
}
```

**Step 4c — Verify no drift remains:**

```bash
terraform plan
```

Expected output:
```
No changes. Your infrastructure matches the configuration.
```

**Step 4d — Raise a Pull Request:**

Commit and push the code change and get it reviewed:

```bash
git add main.tf
git commit -m "Scale EC2 web instance from t3.medium to t3.xlarge — production incident response"
git push
```

The PR description explains:
- Why the change was made
- Who authorized it
- That it has been validated with `terraform plan`

This ensures the change is tracked in Git, reviewed by the team, and has a full audit trail.

---

**Step 5 — Prevent it from happening again**

Fixing the current drift is only half the job. The more important question is — how do we prevent this from happening again?

**Actions I would take:**

**1. Tighten IAM permissions:**

Restrict who can modify EC2 instance types. Only the Terraform pipeline's IAM role should have `ec2:ModifyInstanceAttribute` permission. Developers should have read-only access.

```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:ModifyInstanceAttribute",
    "ec2:StopInstances",
    "ec2:TerminateInstances"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalArn": "arn:aws:iam::123456789:role/terraform-pipeline-role"
    }
  }
}
```

**2. Set up a scheduled drift detection pipeline:**

A nightly pipeline runs `terraform plan -detailed-exitcode` and alerts the team if any drift is found:

```bash
terraform plan -detailed-exitcode

if [ $? -eq 2 ]; then
  echo "DRIFT DETECTED — infrastructure changed outside Terraform"
  # Send alert to Slack / Teams / Email
  exit 1
fi
```

**3. Enable AWS Config Rules:**

AWS Config can detect configuration changes and trigger alerts automatically — adding another layer of drift visibility on top of Terraform.

---

**Complete decision flow — what I do end to end:**

```
Drift Detected
      |
      ↓
Run terraform plan — confirm exactly what changed
      |
      ↓
Check CloudTrail — find who changed it and when
      |
      ↓
Was the change intentional?
      |
   ___|___
  |       |
 NO       YES
  |       |
Revert   Accept it
apply    → refresh-only
         → update code
         → PR + review
         → git commit
      |
      ↓
Prevent recurrence
→ Tighten IAM
→ Drift detection pipeline
→ AWS Config rules
```

---

**Summary (what to say if time is short):**

*"First I would run `terraform plan` to confirm exactly what drifted — in this case the EC2 instance type changed from `t3.medium` to `t3.xlarge`. Then I would check AWS CloudTrail to find out who made the change and when. If the change was unauthorized, I run `terraform apply` to revert it back to the code-defined state. If it was intentional — like an on-call engineer scaling up during an incident — I accept the drift by running `terraform apply -refresh-only` to sync the state, update the code to match, and raise a PR so it is reviewed and tracked in Git. Either way, I then tighten IAM permissions and set up a nightly drift detection pipeline so this is caught early in the future."*

---

#### Scenario 2. Your Terraform deployment takes around one hour for both plan and apply. After deployment, a critical production issue is found in only one resource. How will you fix only that resource without applying the complete infrastructure?

**Answer:**

This is a very practical and important scenario. Waiting one hour just to fix one broken resource in production is not acceptable. Terraform has a built-in solution for exactly this — the **`-target`** flag.

Let me walk through the complete approach step by step.

---

**The solution — `terraform plan -target` and `terraform apply -target`**

The `-target` flag tells Terraform to focus only on a specific resource — plan or apply changes for that one resource alone, completely ignoring everything else in your infrastructure.

**Syntax:**

```bash
terraform plan  -target=resource_type.resource_name
terraform apply -target=resource_type.resource_name
```

---

**Step by step — how I would handle this in production:**

---

**Step 1 — Identify exactly which resource has the issue**

First, I need to know the exact Terraform resource address of the broken resource. I can find it by running:

```bash
terraform state list
```

This shows all resources Terraform is managing:

```
azurerm_resource_group.main
azurerm_kubernetes_cluster.aks
azurerm_container_registry.acr
azurerm_key_vault.kv
azurerm_virtual_network.main
azurerm_subnet.aks
azurerm_monitor_diagnostic_setting.aks_diagnostics
```

Let's say the issue is with the Key Vault — secrets are not accessible and the application is failing.

The resource address is: `azurerm_key_vault.kv`

---

**Step 2 — Fix the code for that specific resource**

Update only the relevant part of your `.tf` file. For example, if the Key Vault firewall rule is blocking the application:

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "azureshop-kv"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku_name            = "standard"
  tenant_id           = data.azurerm_client_config.current.tenant_id

  network_acls {
    default_action = "Allow"      # Changed from "Deny" to "Allow" — this was the issue
    bypass         = "AzureServices"
  }
}
```

---

**Step 3 — Run `terraform plan -target` to preview only that resource**

```bash
terraform plan -target=azurerm_key_vault.kv
```

Output:

```
~ azurerm_key_vault.kv will be updated in-place
  ~ network_acls[0].default_action = "Deny" -> "Allow"

Plan: 0 to add, 1 to change, 0 to destroy.
```

This runs in **seconds** — not one hour — because Terraform is only evaluating one resource. No other resource is touched or checked.

Verify the plan shows exactly what you expect — nothing more, nothing less.

---

**Step 4 — Run `terraform apply -target` to fix only that resource**

```bash
terraform apply -target=azurerm_key_vault.kv
```

Or without the confirmation prompt in an emergency:

```bash
terraform apply -target=azurerm_key_vault.kv -auto-approve
```

Terraform applies the change to only the Key Vault — all other 50+ resources in your infrastructure are completely untouched.

The fix that would have taken one hour now takes under a minute.

---

**Step 5 — Verify the fix in production**

After apply completes, immediately verify that the production issue is resolved:
- Check application logs
- Test the specific functionality that was broken
- Check Azure Monitor / alerts to confirm the error is gone

---

**Step 6 — Run a full plan after the fix**

Once the incident is resolved, run the full `terraform plan` without any target flag:

```bash
terraform plan
```

This is important — using `-target` can sometimes leave other parts of the state slightly out of sync because Terraform only refreshed one resource. A full plan after the fix confirms the overall state is still clean.

If the full plan shows no unexpected changes, you are good. If it shows something unexpected, investigate before applying.

---

**Targeting multiple resources at once:**

If the issue spans two related resources, you can target multiple at the same time:

```bash
terraform apply \
  -target=azurerm_key_vault.kv \
  -target=azurerm_key_vault_access_policy.aks_identity
```

---

**Targeting a whole module:**

If your infrastructure is organised into modules and the entire module needs to be reapplied:

```bash
terraform apply -target=module.aks
terraform apply -target=module.networking
```

This applies all resources inside that module only, skipping everything else.

---

**Real AzureShop Example:**

In our AzureShop project, we had a full infrastructure with AKS, ACR, Key Vault, VNet, Subnets, monitoring, and diagnostic settings — a full `terraform apply` took about 45 minutes.

One day after a deployment, our application pods were failing to pull images from ACR. The root cause was that the `AcrPull` role assignment on the AKS kubelet identity had an incorrect scope — it was scoped to the wrong ACR resource ID.

Instead of running a full 45-minute apply, I used:

```bash
# Fix the scope in the code first, then:

terraform plan -target=azurerm_role_assignment.aks_acr_pull

# Verified the plan showed only the role assignment being updated

terraform apply -target=azurerm_role_assignment.aks_acr_pull -auto-approve
```

The fix was applied in under 2 minutes. Application pods immediately started pulling images and the incident was resolved.

---

**Important warnings about `-target` — what NOT to do:**

HashiCorp marks `-target` as an **exceptional use** flag. It is not meant for regular day-to-day use. Here is why you need to be careful:

| Risk | Explanation |
|---|---|
| **Dependency skipping** | If the targeted resource depends on another resource that also changed, `-target` will not apply those dependency changes — your fix may be incomplete |
| **State inconsistency** | Running `-target` only refreshes that one resource's state. Other resources may have a stale picture in the state file until the next full apply |
| **Not a substitute for proper planning** | If you find yourself using `-target` regularly, it is a sign that your infrastructure needs to be split into smaller, faster Terraform configurations |
| **Can mask wider problems** | Fixing one resource with `-target` while ignoring the full plan means you might miss related drift or issues in other resources |

**The golden rule:**
> Use `-target` for emergency production fixes only. After the incident is resolved, always follow up with a full `terraform plan` and eventually a full `terraform apply` to ensure everything is properly in sync.

---

**Better long-term solution — split your infrastructure:**

If your Terraform deployment regularly takes one hour, the real fix is to split it into smaller independent configurations:

```
infrastructure/
  ├── networking/      # VNet, Subnets, NSGs — rarely changes
  ├── aks/             # AKS cluster — changes occasionally  
  ├── acr/             # Container Registry — rarely changes
  ├── keyvault/        # Key Vault, secrets — changes occasionally
  └── monitoring/      # Log Analytics, alerts — changes occasionally
```

Each folder has its own `terraform init`, `terraform plan`, and `terraform apply`. A change to Key Vault only runs the `keyvault/` configuration — no need to scan the entire infrastructure. This reduces plan and apply time dramatically and removes the need for `-target` entirely.

---

**Summary (what to say if time is short):**

*"I would use the `-target` flag to fix only that specific resource without touching the rest of the infrastructure. First I identify the exact resource address using `terraform state list`. Then I fix the code for that resource, run `terraform plan -target=resource_type.resource_name` to preview the change in seconds, and then `terraform apply -target=resource_type.resource_name` to apply only that resource. This takes under a minute instead of one hour. After the incident is resolved, I always run a full `terraform plan` without any target to confirm the overall state is clean. Long term, I would also look at splitting the infrastructure into smaller independent Terraform configurations so we do not rely on `-target` for speed."*

---

#### Scenario 3. If you use `terraform apply -target`, will the Terraform state file also be updated?

**Answer:**

**Yes — absolutely. The state file IS updated when you run `terraform apply -target`.**

But here is the important part that most people miss — it is updated **only for the targeted resource**. The rest of the state file remains exactly as it was. This is what makes `-target` both powerful and risky at the same time.

Let me explain this in detail.

---

**How `terraform apply -target` updates the state file:**

Every time `terraform apply` runs — with or without `-target` — Terraform always does two things for the resources it processes:

1. Makes the actual change in the cloud (creates, updates, or destroys the resource)
2. Updates the state file to reflect the new current state of that resource

The `-target` flag does not change this behaviour. It only **limits the scope** of which resources Terraform processes. For the targeted resource, both steps happen fully and correctly.

---

**Simple example to understand this:**

Imagine you have 5 resources in your infrastructure:

```
azurerm_resource_group.main         ← State: up to date
azurerm_virtual_network.main        ← State: up to date
azurerm_kubernetes_cluster.aks      ← State: up to date
azurerm_key_vault.kv                ← State: OUTDATED (issue found here)
azurerm_container_registry.acr      ← State: up to date
```

You run:

```bash
terraform apply -target=azurerm_key_vault.kv
```

After this runs:

```
azurerm_resource_group.main         ← State: up to date      ✅ (not touched)
azurerm_virtual_network.main        ← State: up to date      ✅ (not touched)
azurerm_kubernetes_cluster.aks      ← State: up to date      ✅ (not touched)
azurerm_key_vault.kv                ← State: UPDATED         ✅ (fixed and state updated)
azurerm_container_registry.acr      ← State: up to date      ✅ (not touched)
```

Only the Key Vault entry in the state file is updated. Everything else is completely untouched.

---

**What exactly gets written to the state file after `-target`:**

Terraform writes the full, fresh details of the targeted resource into the state file — including:

- The resource ID
- All current attribute values (the new ones after the fix)
- Any computed values that Azure returned after the operation
- Timestamps and metadata

So the state file for that one resource is **completely accurate and up to date** after `-target`.

---

**The risk — partial state refresh for OTHER resources:**

Here is where it gets important. When you run a normal `terraform plan` or `terraform apply`, Terraform refreshes the state of ALL resources by calling the cloud API for each one. This keeps the full state file in sync with reality.

When you run `-target`, Terraform only calls the cloud API for the targeted resource. All other resources are **not refreshed**. Their state entries remain exactly as they were from the last full apply.

This means:

- If another resource drifted between the last full apply and your `-target` run — Terraform does not know about it yet
- The state file for those other resources may be slightly stale
- The next full `terraform plan` will catch and show any drift in those other resources

**This is why I always run a full `terraform plan` after using `-target`** — to make sure nothing else has drifted that Terraform did not check during the targeted apply.

---

**Proof — what happens inside the state file:**

The state file is a JSON file. When you run `terraform apply -target=azurerm_key_vault.kv`, Terraform updates the `instances` block for that specific resource:

**Before `-target`:**
```json
{
  "resource": "azurerm_key_vault.kv",
  "instances": [
    {
      "attributes": {
        "network_acls": [
          {
            "default_action": "Deny"
          }
        ]
      }
    }
  ]
}
```

**After `terraform apply -target=azurerm_key_vault.kv`:**
```json
{
  "resource": "azurerm_key_vault.kv",
  "instances": [
    {
      "attributes": {
        "network_acls": [
          {
            "default_action": "Allow"
          }
        ]
      }
    }
  ]
}
```

The state is now accurate for that resource. All other resource blocks in the JSON file are unchanged.

---

**What if the `-target` apply FAILS halfway?**

If the apply fails after the resource is partially updated, Terraform handles it like any failed apply:

- The state file reflects whatever was successfully completed before the failure
- Resources that were successfully updated are recorded in state
- Resources that failed are left in their previous state or marked as tainted if they were in the middle of creation

This is the same behaviour as a normal failed `terraform apply` — `-target` does not change the failure handling.

---

**State locking still applies with `-target`:**

Even when using `-target`, Terraform still acquires a state lock before making any changes. This means:

- No other team member can run `terraform plan` or `terraform apply` at the same time
- The state file is protected from concurrent modifications
- The lock is released as soon as the targeted apply completes

This is important to know because in a production incident, team members may be watching and trying to run Terraform at the same time. The lock ensures only one operation runs at a time.

---

**Summary table — what `-target` does and does not do to the state file:**

| | Does `-target` do this? |
|---|---|
| Updates state for the targeted resource | ✅ Yes — fully and correctly |
| Updates state for all other resources | ❌ No — other resources are not refreshed |
| Acquires state lock during the operation | ✅ Yes — same as a normal apply |
| Detects drift in non-targeted resources | ❌ No — only the targeted resource is checked against the cloud |
| Leaves the state file in a valid format | ✅ Yes — state file is never corrupted by `-target` |

---

**AzureShop Example:**

In our AzureShop incident where I used `terraform apply -target=azurerm_role_assignment.aks_acr_pull` to fix the ACR pull permission:

- The state file was immediately updated with the correct role assignment details — the new scope, the correct principal ID, and the new assignment ID returned by Azure
- All other resources — AKS, ACR, Key Vault, VNet — remained exactly as they were in the state file, untouched
- After the incident was resolved, I ran a full `terraform plan` and it showed no changes — confirming the entire state was clean and accurate

---

**Summary (what to say if time is short):**

*"Yes, the state file is updated when you use `terraform apply -target`. Terraform always updates the state file for every resource it processes — the `-target` flag just limits which resources are processed. So for the targeted resource, the actual infrastructure is changed AND the state file is updated to reflect the new state. For all other resources, the state file remains exactly as it was — they are not refreshed. This is why after using `-target`, I always follow up with a full `terraform plan` to make sure no drift has occurred in the untouched resources. State locking also still applies during a targeted apply, so concurrent runs are still protected."*

---

#### Scenario 4. Have you used `terraform -target` in your production project?

**Answer:**

**Yes, I have used `terraform apply -target` in production — but only in specific situations where it was genuinely needed. I have never used it as a regular practice.**

Let me share two real incidents from the AzureShop project where I used it, what the situation was, what I did, and what I learned from it.

---

**Incident 1 — AcrPull Role Assignment failure after deployment**

**What happened:**

We had just completed a full Terraform deployment of the AzureShop infrastructure. The full apply took around 40–45 minutes. After deployment, our application pods in AKS started failing with the error:

```
Failed to pull image "azureshopacr.azurecr.io/user-service:latest":
rpc error: code = Unknown desc = failed to pull and unpack image:
unauthorized: authentication required
```

The pods could not pull Docker images from our Azure Container Registry. The entire application was down in the dev environment and we needed to fix it fast — the team was waiting.

**Root cause:**

I checked the Terraform code and the state file and found that the `AcrPull` role assignment for the AKS kubelet identity had been applied with an incorrect scope. Instead of scoping it to the specific ACR resource, it was accidentally scoped to the Resource Group — which Azure rejected.

```hcl
# What was wrong — scoped to resource group
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_resource_group.main.id    # WRONG
}
```

```hcl
# What it should be — scoped to ACR
resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.acr.id    # CORRECT
}
```

**What I did:**

Waiting 45 minutes for a full apply was not an option — the environment was down and the team was blocked. I used `-target` to fix only the role assignment:

```bash
# Step 1 — Verify the fix with plan
terraform plan -target=azurerm_role_assignment.aks_acr_pull

# Output showed:
# -/+ azurerm_role_assignment.aks_acr_pull will be replaced
#   ~ scope = "/subscriptions/.../resourceGroups/azureshop-rg"
#          -> "/subscriptions/.../resourceGroups/azureshop-rg/providers/Microsoft.ContainerRegistry/registries/azureshopacr"

# Step 2 — Apply only that resource
terraform apply -target=azurerm_role_assignment.aks_acr_pull -auto-approve
```

The fix was done in under 2 minutes. Pods restarted, images pulled successfully, application came back up.

**After the incident:**

Once the environment was stable, I ran a full `terraform plan` to confirm everything else was clean — no other drift, no unexpected changes. The plan showed no changes, which meant the overall state was healthy.

I also raised a PR with the corrected scope in the code so it was properly reviewed and merged. The `-target` fix was a tactical response — the PR was the proper permanent fix.

---

**Incident 2 — Key Vault network rule blocking application secrets**

**What happened:**

After a security hardening sprint, a team member updated the Key Vault network ACL to `default_action = "Deny"` and added specific subnet rules. However, they missed adding the AKS subnet to the allowed list. The application pods could no longer read secrets from Key Vault and started crashing.

**What I did:**

Again, a full apply would take too long and was risky — it would touch AKS, networking, and other resources unnecessarily. I isolated the fix:

```bash
# Fix the Key Vault network_acls in code first — added the missing AKS subnet rule
# Then:

terraform plan -target=azurerm_key_vault.kv

# Confirmed only the network_acls block would change — nothing else

terraform apply -target=azurerm_key_vault.kv -auto-approve
```

Application secrets were accessible within 3 minutes. Full `terraform plan` ran clean after the fix.

---

**When I use `-target` and when I do not — my personal rule:**

Over time I developed a clear personal guideline for when `-target` is appropriate:

| Situation | Use `-target`? |
|---|---|
| Production incident — one resource broken, full apply takes too long | ✅ Yes — this is the right tool |
| Need to test a specific new resource in isolation during development | ✅ Yes — acceptable in dev/staging |
| Regular day-to-day deployments | ❌ No — always run full apply |
| Trying to avoid applying changes to a resource because I am not sure about it | ❌ No — fix the uncertainty, do not hide it with `-target` |
| Bootstrapping — creating foundational resources (state backend, networking) before the rest | ✅ Yes — common and acceptable |

---

**What I always do after using `-target` in production:**

I follow a strict three-step rule after every `-target` usage:

**1. Verify the fix immediately:**
Confirm the actual issue is resolved in the application or infrastructure.

**2. Run a full `terraform plan`:**
```bash
terraform plan
```
This confirms no other resources drifted, and the overall state is clean.

**3. Raise a PR with the code fix:**
The `-target` apply is a hotfix. The code fix must follow through a proper Pull Request with peer review so the change is permanently recorded in Git with full context.

---

**What I learned from using `-target` in production:**

1. **It is a scalpel, not a hammer** — extremely precise and powerful when used correctly, but dangerous if used carelessly
2. **Always plan before apply** — even in an emergency, the 10 seconds it takes to run `terraform plan -target` is always worth it. Never skip it.
3. **The real fix is in the code, not in the targeted apply** — `-target` resolves the immediate pain. The PR with the corrected code is the actual fix.
4. **Document it** — whenever I use `-target` in production, I add a comment in the incident channel explaining what I did, what resource was targeted, and why. This gives full visibility to the team.
5. **Prevention is better** — both incidents above happened because of a code review gap. After these incidents, we added a stricter Terraform code review checklist that specifically checked scope of role assignments and network ACL rules.

---

**Summary (what to say if time is short):**

*"Yes, I have used `terraform apply -target` in production on the AzureShop project — twice. The first time was when an AcrPull role assignment had an incorrect scope after a full deployment and pods could not pull images from ACR. The second time was when a Key Vault network ACL change missed the AKS subnet and blocked secret access. In both cases, the full apply would have taken 40-45 minutes and touched resources that did not need changing. Using `-target` fixed the specific broken resource in under 3 minutes each time. After each fix, I ran a full `terraform plan` to confirm the overall state was clean and raised a PR with the corrected code for proper review. My rule is — use `-target` only for genuine production incidents or isolated testing in dev, never for regular deployments."*

---

#### Scenario 5. You have an existing Terraform-managed Azure environment. A customer wants Read-Only access to an existing Resource Group. How would you implement this?

**Answer:**

This is a very real-world scenario. Customers often need visibility into their Azure environment — to check resource status, monitor costs, or audit what is deployed — without being able to make any changes.

The correct way to implement this in Azure is using **Azure RBAC (Role Based Access Control)** — specifically the built-in **`Reader`** role — and I would do the entire implementation through Terraform so it is tracked in code, reviewable, and repeatable.

Let me walk through the complete implementation step by step.

---

**Understanding the solution — Azure RBAC:**

Azure RBAC controls who can do what on which resources. Every access grant has three parts:

- **Who** — the user, group, or service principal getting access (called a **Security Principal**)
- **What** — the role being assigned (e.g., `Reader`, `Contributor`, `Owner`)
- **Where** — the scope at which the role applies (subscription, resource group, or individual resource)

For this scenario:
- **Who** — the customer's Azure AD account or guest user
- **What** — `Reader` role (read-only, no changes allowed)
- **Where** — the specific Resource Group they need to see

---

**Step 1 — Find out what type of account the customer has**

Before writing any Terraform, I ask one question: **Does the customer have an Azure AD account in our tenant, or are they external?**

**Case A — Customer is an internal user (already in our Azure AD):**
I just need their Object ID from Azure AD.

**Case B — Customer is external (different company, different tenant):**
I need to invite them as a **Guest User** (Azure B2B) first, which creates them in our Azure AD as a guest. Then I assign the role to that guest account.

In most real cases with customers, it is Case B — they are external. So let me cover that fully.

---

**Step 2 — Invite the customer as a Guest User using Terraform**

For an external customer, we use the **AzureAD provider** (now called `azuread`) to send a B2B guest invitation.

First, add the `azuread` provider:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.0"
    }
  }
}

provider "azuread" {}
provider "azurerm" {
  features {}
}
```

Now invite the customer as a guest:

```hcl
resource "azuread_invitation" "customer_guest" {
  user_email_address = "customer@externalcompany.com"
  redirect_url       = "https://portal.azure.com"

  message {
    additional_recipients = []
    body                  = "You have been granted read-only access to the AzureShop Resource Group. Please accept this invitation to proceed."
  }
}
```

This sends an invitation email to the customer. Once they accept, their guest account is created in our Azure AD.

---

**Step 3 — Get a reference to the existing Resource Group**

Since the Resource Group already exists and is managed by Terraform, we use a `data` source to reference it — we do not recreate it:

```hcl
data "azurerm_resource_group" "existing" {
  name = "azureshop-rg"
}
```

This reads the existing Resource Group without touching or modifying it.

---

**Step 4 — Assign the Reader role to the customer**

Now assign the built-in `Reader` role to the customer's guest account, scoped to only that Resource Group:

```hcl
resource "azurerm_role_assignment" "customer_reader" {
  principal_id         = azuread_invitation.customer_guest.user_id
  role_definition_name = "Reader"
  scope                = data.azurerm_resource_group.existing.id

  depends_on = [azuread_invitation.customer_guest]
}
```

Breaking this down:
- `principal_id` — the Object ID of the customer's guest account
- `role_definition_name = "Reader"` — the built-in Azure read-only role
- `scope` — the Resource Group ID — access is limited to only this RG, nothing else in the subscription
- `depends_on` — ensures the guest invitation is fully processed before the role assignment is attempted

---

**What the `Reader` role allows and does NOT allow:**

| Action | Reader Role |
|---|---|
| View all resources in the Resource Group | ✅ Yes |
| View resource configurations and properties | ✅ Yes |
| View resource metrics and logs (if permitted) | ✅ Yes |
| Create any resource | ❌ No |
| Modify any resource | ❌ No |
| Delete any resource | ❌ No |
| View secrets in Key Vault | ❌ No — Reader does not include secret access |
| View connection strings or passwords | ❌ No |

This is exactly what a customer needs — full visibility, zero ability to make changes.

---

**Step 5 — Scope carefully — do not give more access than needed**

The scope of the role assignment is critical. I always scope it to the **smallest possible level** — the principle of least privilege.

**Option A — Scope to the entire Resource Group (most common for this scenario):**
```hcl
scope = data.azurerm_resource_group.existing.id
# Customer can see ALL resources inside this RG
```

**Option B — Scope to a single resource only (if customer needs even less access):**
```hcl
scope = data.azurerm_kubernetes_cluster.aks.id
# Customer can see ONLY the AKS cluster, nothing else in the RG
```

**Option C — Scope to the entire Subscription (never do this for a customer):**
```hcl
scope = "/subscriptions/${var.subscription_id}"
# Customer can see EVERYTHING — too broad, never for external customers
```

Always choose the narrowest scope that meets the customer's need.

---

**Complete Terraform code — all together:**

```hcl
# 1. Reference the existing Resource Group
data "azurerm_resource_group" "existing" {
  name = "azureshop-rg"
}

# 2. Invite the customer as a guest user
resource "azuread_invitation" "customer_guest" {
  user_email_address = var.customer_email
  redirect_url       = "https://portal.azure.com"

  message {
    body = "You have been granted read-only access to the AzureShop Resource Group."
  }
}

# 3. Assign Reader role scoped to the Resource Group
resource "azurerm_role_assignment" "customer_reader" {
  principal_id         = azuread_invitation.customer_guest.user_id
  role_definition_name = "Reader"
  scope                = data.azurerm_resource_group.existing.id

  depends_on = [azuread_invitation.customer_guest]
}

# 4. Output the details for confirmation
output "customer_access_summary" {
  value = {
    customer_email  = var.customer_email
    role_assigned   = "Reader"
    scope           = data.azurerm_resource_group.existing.name
    assignment_id   = azurerm_role_assignment.customer_reader.id
  }
}
```

Using a variable for the email makes this reusable — next time a different customer needs access, just pass a different email:

```hcl
variable "customer_email" {
  description = "Email address of the customer to grant read-only access"
  type        = string
}
```

---

**Step 6 — Apply using `-target` since the environment already exists**

Since this is an existing environment and we only want to add the guest invitation and role assignment without touching anything else, I use `-target`:

```bash
terraform plan \
  -target=azuread_invitation.customer_guest \
  -target=azurerm_role_assignment.customer_reader

terraform apply \
  -target=azuread_invitation.customer_guest \
  -target=azurerm_role_assignment.customer_reader
```

This adds only the new access — the existing infrastructure is completely untouched.

---

**If the customer already has an Azure AD account in our tenant:**

If the customer is an internal user (same Azure AD tenant), skip the invitation step entirely and use a `data` source to look up their existing account:

```hcl
data "azuread_user" "customer" {
  user_principal_name = "customer@ourcompany.com"
}

resource "azurerm_role_assignment" "customer_reader" {
  principal_id         = data.azuread_user.customer.object_id
  role_definition_name = "Reader"
  scope                = data.azurerm_resource_group.existing.id
}
```

Clean and simple — no invitation needed.

---

**If multiple customers need the same access — use an Azure AD Group:**

If 5 customers all need Reader access, do not create 5 separate role assignments. Create an Azure AD Group, assign the role to the group, and add the customers to the group:

```hcl
# Create a group for all customer readers
resource "azuread_group" "customer_readers" {
  display_name     = "azureshop-customer-readers"
  security_enabled = true
}

# Add customers to the group
resource "azuread_group_member" "customer1" {
  group_object_id  = azuread_group.customer_readers.object_id
  member_object_id = azuread_invitation.customer1_guest.user_id
}

# Assign Reader role to the GROUP — one assignment covers all customers
resource "azurerm_role_assignment" "customer_readers_rg" {
  principal_id         = azuread_group.customer_readers.object_id
  role_definition_name = "Reader"
  scope                = data.azurerm_resource_group.existing.id
}
```

Now adding or removing customers is just adding or removing group members — the role assignment itself never changes. This is cleaner and scales well.

---

**AzureShop Real Example:**

In AzureShop, one of our clients wanted to audit the infrastructure we had deployed for them — they wanted to see their AKS cluster, ACR, and network resources without being able to change anything.

I implemented this exactly as above:
1. Created a `azuread_group` called `azureshop-client-readers`
2. Invited the 3 client contacts as guest users using `azuread_invitation`
3. Added them to the group using `azuread_group_member`
4. Assigned the `Reader` role to the group scoped to `azureshop-rg`
5. Used `-target` to apply only these new resources without touching the existing environment

The clients could log into portal.azure.com and browse all resources in the Resource Group — but every button to create, modify, or delete was greyed out.

---

**Summary (what to say if time is short):**

*"I would implement this using Azure RBAC through Terraform. First I check whether the customer is an external user — if so, I use the `azuread_invitation` resource to invite them as a B2B guest user in our Azure AD. Then I use a `data` source to reference the existing Resource Group without modifying it. Finally, I create an `azurerm_role_assignment` assigning the built-in `Reader` role to the customer's account, scoped specifically to that Resource Group. Reader role gives full visibility — they can see all resources and properties — but zero ability to create, modify, or delete anything. If multiple customers need the same access, I assign the role to an Azure AD group instead of individual users, so managing access is just adding or removing group members. I apply these new resources using `-target` so the existing infrastructure is not touched."*

---

#### Scenario 6. Suppose Terraform Plan detects Drift. How would you proceed? And customer confirms the Drift is valid? How would you proceed? What if the customer says the Drift is NOT valid? Explain both scenarios.

**Answer:**

This is an excellent real-world scenario that tests not just your Terraform knowledge but also your professional judgement and communication skills. Let me walk through both situations completely.

---

**First — What happens when `terraform plan` detects drift:**

You run the nightly drift detection pipeline or a routine `terraform plan` and the output is not clean:

```bash
terraform plan
```

Output:
```
~ azurerm_linux_virtual_machine.app will be updated in-place
  ~ size = "Standard_D2s_v3" -> "Standard_D4s_v3"

~ azurerm_key_vault.kv will be updated in-place
  ~ network_acls[0].default_action = "Allow" -> "Deny"

Plan: 0 to add, 2 to change, 0 to destroy.
```

Terraform has found drift on two resources:
1. The VM size was changed from `Standard_D2s_v3` to `Standard_D4s_v3`
2. The Key Vault network rule was changed from `Allow` to `Deny`

**My immediate reaction — do NOT run `terraform apply` yet.**

Before doing anything to the infrastructure, I need to understand what happened. I follow these steps first.

---

**Step 1 — Investigate who made the change and when**

I check the Azure Activity Log or AWS CloudTrail to find out who made the manual change:

```bash
# Azure — check activity log for the resource group
az monitor activity-log list \
  --resource-group azureshop-rg \
  --start-time 2026-07-17T00:00:00Z \
  --output table
```

This tells me:
- **Who** changed it
- **When** it was changed
- **What exactly** was changed

---

**Step 2 — Communicate with the customer or relevant stakeholder**

Once I have the details, I do not act unilaterally. I raise the drift report to the customer or the team lead who owns the environment and clearly explain:

**What I tell the customer:**

> "Our drift detection pipeline has detected that two resources in your environment were modified outside of Terraform between yesterday and today.
>
> 1. The VM `azureshop-app-vm` size was changed from `Standard_D2s_v3` to `Standard_D4s_v3`
> 2. The Key Vault `azureshop-kv` network access rule was changed from `Allow` to `Deny`
>
> These changes were not made through Terraform and are not reflected in the codebase. Could you confirm — were these changes intentional or unauthorized?"

Now the customer responds. Two scenarios follow.

---

---

## Scenario A — Customer confirms the Drift IS Valid (Intentional Change)

The customer replies:

> "Yes, both changes are valid. The VM was scaled up during a performance issue last night by our on-call engineer. The Key Vault network rule was tightened by our security team for compliance reasons. Please keep both changes."

**This means:** The manual changes are correct and should be kept. I need to accept the drift — update the Terraform state and code to match reality.

---

**Step 1 — Run `terraform apply -refresh-only` to sync the state file**

This updates the Terraform state file to reflect the current actual state of the infrastructure — without making any changes in Azure.

```bash
terraform apply -refresh-only
```

Output:
```
~ azurerm_linux_virtual_machine.app
  ~ size = "Standard_D2s_v3" -> "Standard_D4s_v3"

~ azurerm_key_vault.kv
  ~ network_acls[0].default_action = "Allow" -> "Deny"

This plan will update the Terraform state to match the real infrastructure.
No changes to actual cloud resources will be made.

Do you want to update the Terraform state? yes
```

After this — the state file is now in sync with the real infrastructure.

---

**Step 2 — Update the Terraform code to match the changes**

```hcl
# Update VM size
resource "azurerm_linux_virtual_machine" "app" {
  name = "azureshop-app-vm"
  size = "Standard_D4s_v3"    # Updated — scaled up during performance incident on 17-07-2026
  # ... rest of config unchanged ...
}
```

```hcl
# Update Key Vault network rule
resource "azurerm_key_vault" "kv" {
  name = "azureshop-kv"
  # ... other config ...

  network_acls {
    default_action = "Deny"    # Updated — tightened for compliance on 17-07-2026
    bypass         = "AzureServices"
    ip_rules       = var.allowed_ip_ranges
  }
}
```

---

**Step 3 — Run `terraform plan` to confirm zero drift**

```bash
terraform plan
```

Expected output:
```
No changes. Your infrastructure matches the configuration.
```

This confirms the code, state file, and actual Azure infrastructure are all perfectly in sync now.

---

**Step 4 — Raise a Pull Request**

```bash
git add main.tf
git commit -m "Accept valid drift — VM scaled to D4s_v3, KV network ACL set to Deny (compliance)"
git push
```

The PR description includes:
- What changed
- Who authorized it
- Why it was changed
- That `terraform plan` shows no drift after the update

This gives the full change a proper audit trail in Git — so anyone looking at the history months later can see exactly what happened and why.

---

**Step 5 — Update the drift detection pipeline notification**

Reply to the drift alert confirming it has been resolved and the code has been updated. Close the incident ticket.

---

**Full flow for Scenario A:**

```
Drift detected → Investigate → Communicate to customer
      ↓
Customer says: "Valid change, keep it"
      ↓
terraform apply -refresh-only   (sync state to reality)
      ↓
Update .tf code to match
      ↓
terraform plan → No changes ✅
      ↓
Raise PR → Review → Merge
      ↓
Drift resolved — code and infra in sync
```

---

---

## Scenario B — Customer confirms the Drift is NOT Valid (Unauthorized Change)

The customer replies:

> "No, these changes were not authorized. Nobody from our team made those changes. This is a security concern — the VM was not supposed to be resized and the Key Vault network rule should never have been changed to Deny. Please revert both immediately."

**This means:** The changes were unauthorized. This is now both a **technical issue** and a **security incident**. I need to revert the drift and investigate how it happened.

---

**Step 1 — Immediately revert the drift using `terraform apply`**

Since the changes are unauthorized, I revert both resources back to the Terraform-defined state:

```bash
terraform apply
```

Output:
```
~ azurerm_linux_virtual_machine.app will be updated in-place
  ~ size = "Standard_D4s_v3" -> "Standard_D2s_v3"

~ azurerm_key_vault.kv will be updated in-place
  ~ network_acls[0].default_action = "Deny" -> "Allow"

Plan: 0 to add, 2 to change, 0 to destroy.

Do you want to perform these actions? yes
```

Terraform reverts both resources back to exactly what the code says. The infrastructure is now back in its authorized state.

---

**Step 2 — Verify the revert was successful**

```bash
terraform plan
```

Expected output:
```
No changes. Your infrastructure matches the configuration.
```

Drift is gone. Infrastructure matches the code again.

---

**Step 3 — Investigate the security incident**

Since the changes were unauthorized, this is now a security investigation. I check:

**Who had access to make these changes?**

```bash
# Check Azure Activity Log — who made the specific changes
az monitor activity-log list \
  --resource-group azureshop-rg \
  --start-time 2026-07-17T00:00:00Z \
  --query "[?operationName.value=='Microsoft.Compute/virtualMachines/write']" \
  --output table
```

This shows:
- The IAM identity (user/service principal) that made the change
- The IP address the change came from
- The exact time

**Key questions to answer:**
- Was it a compromised account?
- Was it a misconfigured service principal with too-broad permissions?
- Was it a contractor or ex-employee whose access was not revoked?
- Was it an automated tool (like a cost optimization script) that ran without authorization?

---

**Step 4 — Immediately lock down access**

While investigation is ongoing, I take protective action:

```bash
# Revoke the IAM role that made the unauthorized change
az role assignment delete \
  --assignee <object-id-of-offending-identity> \
  --role "Contributor" \
  --resource-group azureshop-rg
```

If the Terraform service principal itself was compromised:
- Rotate the service principal credentials immediately
- Review all recent pipeline runs for suspicious activity

---

**Step 5 — Tighten permissions to prevent recurrence**

Through Terraform, restrict who can modify infrastructure:

```hcl
# Deny any modification to the resource group except from the Terraform pipeline identity
resource "azurerm_role_assignment" "deny_manual_changes" {
  principal_id         = var.developer_group_object_id
  role_definition_name = "Reader"    # Downgrade developers from Contributor to Reader
  scope                = data.azurerm_resource_group.existing.id
}
```

Only the Terraform pipeline's service principal should have `Contributor` or higher. All human access should be `Reader` only.

---

**Step 6 — Raise a Security Incident Report**

Document the full incident:
- What was changed
- When it was changed
- Who made the change (from Activity Log)
- How it was detected (drift detection pipeline)
- How it was resolved (terraform apply revert)
- What access was revoked
- What controls were added to prevent recurrence

Share this report with the customer and keep it in your incident management system.

---

**Full flow for Scenario B:**

```
Drift detected → Investigate → Communicate to customer
      ↓
Customer says: "NOT valid — revert immediately"
      ↓
terraform apply   (revert drift — restore authorized state)
      ↓
terraform plan → No changes ✅
      ↓
Investigate: who made the change? (Activity Log)
      ↓
Revoke unauthorized access immediately
      ↓
Tighten IAM permissions in Terraform
      ↓
Raise Security Incident Report
      ↓
Share report with customer
```

---

---

**Side by side comparison — Scenario A vs Scenario B:**

| | Scenario A — Drift is Valid | Scenario B — Drift is NOT Valid |
|---|---|---|
| **Customer response** | "Yes, we made this change intentionally" | "No, this was unauthorized" |
| **Action on infrastructure** | Keep the change | Revert the change |
| **Terraform command** | `terraform apply -refresh-only` | `terraform apply` |
| **Code update** | Yes — update `.tf` to match reality | No — code stays as-is (already correct) |
| **Pull Request** | Yes — to commit the accepted change | No — code was never wrong |
| **Security investigation** | No — intentional change | Yes — must find who made it |
| **Access review** | No immediate action | Yes — revoke unauthorized access immediately |
| **Incident report** | Optional — document in PR | Mandatory — formal security incident report |

---

**AzureShop Example:**

In AzureShop, our nightly drift detection pipeline flagged that the AKS node count had changed from 3 to 6.

I raised it to the customer. They said:

> "Yes, our on-call engineer scaled it up during peak load last night — it was intentional. But we should have gone through the proper process."

This was **Scenario A** — valid drift. We ran `refresh-only`, updated the code to `node_count = 6`, raised a PR, and merged it. We also added a reminder to the on-call runbook: always raise a PR within 24 hours of any manual emergency change.

A month later, the pipeline flagged that the Key Vault firewall rules had been completely removed. The customer immediately said this was NOT authorized — nobody from their team did it.

This was **Scenario B** — we immediately ran `terraform apply` to restore the firewall rules, investigated the Activity Log, found it was a misconfigured automation script from a third-party monitoring tool, revoked the script's access, and filed a full security incident report.

---

**Summary (what to say if time is short):**

*"When `terraform plan` detects drift, my first step is always to investigate — check the Activity Log to find who made the change and when — and then communicate clearly to the customer with the exact details of what drifted. If the customer confirms the drift is valid, I accept it by running `terraform apply -refresh-only` to sync the state, update the `.tf` code to match, verify with `terraform plan` that there is no drift remaining, and raise a PR to commit the accepted change with full context. If the customer says the drift is not valid, I immediately run `terraform apply` to revert the infrastructure back to the code-defined state, then investigate who made the unauthorized change using the Activity Log, revoke that access, tighten IAM permissions through Terraform, and raise a formal security incident report. The key difference — valid drift means accept and codify, invalid drift means revert and investigate."*

---

<!-- Add more scenario questions as Scenario 7, Scenario 8... -->

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
