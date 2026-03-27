# 🔐 Jenkins RBAC (Role-Based Access Control) Guide

> Step-by-step guide to set up Role-Based Access Control in Jenkins.

---

## What is RBAC?

RBAC allows you to control **who can do what** in Jenkins.  
It improves security by restricting access based on **roles**.

- Admin : Full access
- Manager : Trigger build, View builds, view reports
- Developer : Create, edit, build job
- QA : Trigger builds, view job status, access logs
- Viewer : Read-only

---

## ⚙️ Requirements

- Jenkins installed
- Admin access to Jenkins
- Role-Based Strategy Plugin

---

## Step 1️⃣ : Install Role-Based Strategy Plugin

1. Go to **Manage Jenkins → Plugins**  
2. Search for **Role-Based Strategy**  
3. Click **Install** (restart if needed)  

---

## Step 2️⃣ : Enable RBAC

1. Go to **Manage Jenkins → Security**  
2. Under **Authorization**, select **Role-Based Strategy**
3. Click **Save**  

> ✅ Important: Keep at least one admin user to avoid locking yourself out

---

## Step 3️⃣: Create Roles

> Go to **Manage Jenkins → Manage and Assign Roles**

### 3.1. Permission Templates

> Templates help standardize permissions across roles

**Permission Templates Example**

| Name                | Read | Build | Configure |
|---------------------|------|--------|----------|
| Developer Template  | ✅   | ✅    | ✅       | 
| Auditor Template    | ✅   | ❌    | ❌       | 
| QA Template         | ✅   | ❌    | ❌       | 
| Manager Template    | ✅   | ✅    | ❌       | 

- [x] **Click Save**  

### 3.2. Manage Roles

> Under Role to add **Enter Roles -> Click Add**

**Global Roles Example:**

| Role     | Read | Administer | 
|----------|------|------------|
| admin    | ❌   | ✅        | 
| Developer | ✅   | ❌        |
| Manager  | ✅   | ❌        | 
| QA       | ✅   | ❌        | 
| Auditor  | ✅   | ❌        | 

- [x] **Click Save** 

> **Item Role**
❔ What is an Item Role?
- In Jenkins, an Item Role is a role that applies to specific jobs, folders, or pipelines.
- Different from Global Roles (which apply across Jenkins).
- Allows you to limit access to only certain items without giving full Jenkins access.

**Step 1 :**Add role name, e.g., Developer
**Step 2 :**Set a pattern to match jobs/folders (REGEX)
- Example: `Project-A*`
**Step 3 :**Choose Permission Templates
**Step 4:** Click **Add** and **Save** 

> 💡 Item Roles + Global Roles can be combined for fine-grained access control

### 3.3. Assign Roles

> Add User ID and Select the role you want to assign
> ✅ Check with a test user to confirm permissions

---

## ⚠️ Notes 

- If a user has **no role**, they have **no access**  
- Always keep at least **one admin account**  
- Double-check roles after creating pipelines or plugins  

---

## 🔗 References

- [Role-Based Strategy Plugin](https://plugins.jenkins.io/role-strategy/)  