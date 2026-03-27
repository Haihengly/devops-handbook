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

> 💡 Screenshot example: `./screenshots/plugins.png`

---

## Step 2️⃣ : Enable RBAC

1. Go to **Manage Jenkins → Security**  
2. Under **Authorization**, select **Role-Based Strategy**
3. Click **Save**  

> ✅ Important: Keep at least one admin user to avoid locking yourself out

---

## Step 3️⃣: Create Roles

1. ### Permission Templates

> Use template to quickly assign permissions to each role

**Global Roles Example:**

Permission Templates

| Name                | Read | Build | Configure | Admin |
|---------------------|------|--------|----------|--------|
| Developer Template  | ✅   | ✅    | ✅       | ✅    |
| Auditor Template    | ✅   | ✅    | ❌       | ❌    |
| QA Template         | ✅   | ❌    | ❌       | ❌    |
| Manager Template    | ✅   | ❌    | ❌       | ❌    |





**Global Roles Example:**

| Role   | Read | Build | Configure | Admin |
|--------|------|-------|----------|-------|
| Admin  | ✅   | ✅    | ✅       | ✅    |
| Dev    | ✅   | ✅    | ❌       | ❌    |
| Viewer | ✅   | ❌    | ❌       | ❌    |

- [x] Go to **Manage Jenkins → Manage and Assign Roles → Manage Roles**  
- [x] Create roles with the desired permissions  

> 💡 Screenshot example: `./screenshots/rbac-config.png`

---

## 🧑‍💻 Step 4: Assign Roles to Users

1. Go to **Manage Jenkins → Manage and Assign Roles → Assign Roles**  
2. Assign each user to the proper role  
3. Save changes

> ✅ Check with a test user to confirm permissions

---

## ⚠️ Notes / Gotchas

- If a user has **no role**, they have **no access**  
- Always keep at least **one admin account**  
- Double-check roles after creating pipelines or plugins  
- Keep screenshots for future reference  

---

## 📝 Example Checklist for Setup

- [x] Plugin installed  
- [x] RBAC enabled  
- [x] Roles created  
- [ ] Users assigned  

---

## 🔗 References

- [Jenkins RBAC Documentation](https://www.jenkins.io/doc/book/security/authorization/)  
- [Role-Based Strategy Plugin](https://plugins.jenkins.io/role-strategy/)  