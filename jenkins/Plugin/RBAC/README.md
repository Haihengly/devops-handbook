# Jenkins RBAC Setup Guide

## 📌 What is RBAC
Role-Based Access Control lets you control what users can do in Jenkins.

## ⚙️ Install Plugin
1. Go to Manage Jenkins
2. Click Plugins
3. Install Role-Based Strategy Plugin

## 👤 Roles

| Role   | Read | Build | Configure |
|--------|------|-------|----------|
| Admin  | ✅   | ✅    | ✅       |
| Dev    | ✅   | ✅    | ❌       |
| Viewer | ✅   | ❌    | ❌       |

## 📝 Notes
- Always keep one admin user
- Be careful not to lock yourself out