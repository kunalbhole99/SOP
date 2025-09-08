✅ Step 1: Edit ArgoCD RBAC Config

Open RBAC config in your namespace (cd):

Bash 
- kubectl edit configmap argocd-rbac-cm -n cd

If it’s empty, add something like this:

Yaml:

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

✅ Step 2: Restart ArgoCD API Server

Changes take effect after restart:

Bash
- kubectl rollout restart deployment argocd-server -n cd

✅ Step 3: Login as Different Users

If you’re using ArgoCD’s local accounts (not Keycloak yet):

1. Create users in argocd-cm:

Bash
- kubectl edit configmap argocd-cm -n cd

Add under data::

Yaml:

data:
   accounts.adminuser: apiKey, login
   accounts.readonlyuser: apiKey, login

2. Restart API server again:

Bash
- kubectl rollout restart deployment argocd-server -n cd

3. Now set passwords:
Bash:
- argocd account update-password --account adminuser
- argocd account update-password --account readonlyuser


