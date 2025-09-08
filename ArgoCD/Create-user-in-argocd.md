---
✅ Step 1: Edit ArgoCD RBAC Config

Open RBAC config in your namespace (argocd):

```bash 
kubectl edit configmap argocd-rbac-cm -n cd
```
If it’s empty, add something like this:

```Yaml:
data:
  policy.csv: |
  # Admin role → Full access to everything in your custom project
    p, role:project-admin, applications, *, cd/*, allow
    p, role:project-admin, projects, *, cd/*, allow

  # Readonly role → Can only view apps/projects
    p, role:project-readonly, applications, get, cd/*, allow
    p, role:project-readonly, projects, get, cd/*, allow

  # Map users to roles
    g, adminuser, role:project-admin
    g, readonlyuser, role:project-readonly

  policy.default: role:readonly
```

🔎 Explanation of Each Section:

1. Policy Definition (p, …)

p = permission rule

Format is:
```
p, <subject>, <resource>, <action>, <object>, <effect>
```
* subject → The role or user (e.g., role:project-admin)

* resource → Type of ArgoCD resource (applications, projects, clusters, repositories, etc.)

* action → What operation is allowed (get, create, update, delete, sync, override, action/*, *)

* object → Which resource instance(s) (cd/* → all in namespace cd)

* effect → allow or deny

✅ Step 2: Restart ArgoCD API Server

Changes take effect after restart:

```bash
kubectl rollout restart deployment argocd-server -n cd
```

✅ Step 3: Login as Different Users

If you’re using ArgoCD’s local accounts (not Keycloak yet):

1. Create users in argocd-cm:

```bash
kubectl edit configmap argocd-cm -n cd
```

* Add under data::

```Yaml:
data:
  accounts.adminuser: apiKey, login
  accounts.readonlyuser: apiKey, login
```

2. Restart API server again:

```bash
kubectl rollout restart deployment argocd-server -n cd
```
3. To set password of newly created users:
* Login to ArgoCD:
  
```bash
argocd login <ARGOCD_SERVER> --username admin --password <INITIAL_ADMIN_PASSWORD> --insecure
```

* Replace <ARGOCD_SERVER> with the URL from step 1 (example: localhost:8080 or argocd.mycompany.com).

* --insecure is needed if you haven’t set up TLS properly yet.

👉 The initial admin password is stored in Kubernetes secret:

```
kubectl -n cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

3. Now Set Password for Your User:
```bash:
argocd account update-password --account adminuser
```

```bash:
argocd account update-password --account readonlyuser
```

* You will prompt to set new password to given user.

  
