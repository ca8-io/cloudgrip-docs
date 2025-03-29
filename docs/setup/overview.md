# ☁️ CloudGrip Schema Overview

This documentation provides an overview of the JSON schema files that define the core structure of a CloudGrip-managed Azure environment. These files — `tenant.json`, `managementgroup.json`, and `subscription.json` — serve as declarative blueprints for provisioning and managing resources within Azure using standardized configuration files.

---

## 📁 Schema Files Overview

| File | Purpose | Azure Equivalent |
|------|---------|------------------|
| `tenant.json` | Defines the entire tenant-level structure including root management group and global policies. | Azure AD Tenant and Root Management Group |
| `managementgroup.json` | Defines child management groups, their policies, RBAC, and subscriptions. | Azure Management Group and related structure |
| `subscription.json` | Defines individual Azure subscriptions, including deployments, tags, RBAC, and marketplace features. | Azure Subscription |

---

## 🧱 How It Works

These schemas represent the **CloudGrip Configuration-as-Code model** for enterprise-scale Azure architecture. Each layer builds upon the previous, allowing for a modular, auditable, and repeatable deployment pattern.

### 🏢 1. `tenant.json`: Tenant-Level Blueprint

The `tenant.json` file represents the **entry point** into CloudGrip's configuration model. It outlines:

- **Tenant Metadata**: Name, ID, and primary domain.
- **Root Management Group**: The top-level group in the hierarchy.
- **Global Policies and Roles**: Security baselines, RBAC assignments.
- **Centralized Marketplace Features**: SaaS integrations or partner services.
- **Initial Subscriptions**: Bootstrapped under the root group.

> 🔗 *This correlates with your EntraId tenant and the top-level Root Management Group (`/providers/Microsoft.Management/managementGroups/{root}`)*

---

### 🧩 2. `managementgroup.json`: Scalable Hierarchy

The `managementgroup.json` file defines a **single management group** and its contents:

- **Child Management Groups**: For structure like `Identity`, `Connectivity`, `Platform`, etc.
- **Policy Assignments**: Compliant environments per group.
- **RBAC Configuration**: Access control at the group level.
- **Subscriptions**: Subscriptions logically grouped here.
- **Optional PIM Assignments**: Privileged access control.

> 🔗 *This schema maps directly to Azure Management Groups under your tenant, allowing fine-grained access, tagging, and policy enforcement per domain (e.g., business unit, environment).*

---

### 📦 3. `subscription.json`: The Deployable Unit

Each `subscription.json` file defines an Azure **subscription** in detail:

- **Metadata**: Subscription ID, name, tenant linkage.
- **RBAC**: Who gets access and what role.
- **Tags**: Cost center, environment labels, etc.
- **Deployments**: Infrastructure-as-code components (Bicep, ARM, Terraform).
- **Central Config References**: Shared VNet, vHub, LAW workspace.
- **Marketplace Features**: Enablement of CloudGrip products.

> 🔗 *This file corresponds to an actual Azure subscription, and provides everything needed to deploy infrastructure inside it.*

---

## 🔄 Schema Relationships
