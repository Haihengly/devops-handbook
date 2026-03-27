# 🔐 Jenkins RBAC Setup Guide

## 📌 What is RBAC
Role-Based Access Control (RBAC) lets you control what users can do in Jenkins.

---

## ⚙️ Requirements
- Jenkins installed
- Admin access

---

## 🔌 Install Role-Based Plugin
1. Go to **Manage Jenkins**
2. Click **Plugins**
3. Search: `Role-Based Strategy`
4. Install it

---

## 🔒 Enable RBAC
1. Go to **Manage Jenkins → Configure Global Security**
2. Under Authorization → select:
   **Role-Based Strategy**

---

## 👤 Create Roles

### Global Roles
- **Admin** → Full access
- **Dev** → Read + Build
- **Viewer** → Read only

---

## 🧑‍💻 Assign Roles
1. Go to **Manage Jenkins**
2. Click **Manage and Assign Roles**
3. Assign roles to users

---

## 📊 Example Permissions

| Role   | Read | Build | Configure | Admin |
|--------|------|-------|----------|-------|
| Admin  | ✅   | ✅    | ✅       | ✅    |
| Dev    | ✅   | ✅    | ❌       | ❌    |
| Viewer | ✅   | ❌    | ❌       | ❌    |

---

## ⚠️ Notes
- Always keep at least one admin account
- If no role is assigned → user has no access
- Be careful not to lock yourself out