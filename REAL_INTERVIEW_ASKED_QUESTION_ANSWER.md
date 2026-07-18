# Real Interview Asked Questions & Answers

> This file is a personal log of actual questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.

---

## Table of Contents

- [Interview #1](#interview-1)

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

<!-- Add more questions as Q2, Q3... -->

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
