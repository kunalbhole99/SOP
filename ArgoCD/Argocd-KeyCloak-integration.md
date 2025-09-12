# ArgoCD & KeyCloak Integration

---
# Keycloak Configuration for ArgoCD

## ✅ Step 1: Create a Client in Realm
1. Select **Client Type** as **OpenID Connect**  
2. Provide **Client ID** (example: `argocd`)  
3. Provide **Name** and **Description**  
4. Set **Always Display in UI** → **ON**  
5. Enable **Client Authentication**  
6. In **Authentication Flow**, choose:  
   - Standard Flow  
   - Direct Access Grants  
7. Configure Redirects:  
   - **Valid Redirect URIs** → `https://172.31.7.140:30015/auth/callback`  
   - **Valid Post Logout Redirect URIs** → `https://172.31.7.140:30015`  
   - **Web Origins** → `*`  

---

## ✅ Step 2: Create a Client Scope
- **Name** → `groups`  
- **Description** → `groups`  
- **Select Type** → `Default`  

---

## ✅ Step 3: Create a Mapper in that Client Scope
1. Add a new **Mapper** by configuration  
2. Choose **Group Membership**  
3. Provide:  
   - **Name** → `groups`  
   - **Token Claim Name** → `groups`  
4. Deselect **Full Group Path**  

---

## ✅ Step 4: Create Groups
- Create groups as per your requirements.  

---

## ✅ Step 5: Create Users and Assign to Groups
- Create users and assign them to the created groups.  



# ArgoCD Configuration for keycloak

## ✅ Step 1: Edit ArgoCD RBAC Config

Open RBAC config in your namespace (argocd):

```bash 
kubectl edit configmap argocd-rbac-cm -n argocd
```
If it’s empty, add something like this:

```Yaml:
data:
  policy.csv: |
    # Admin role → Full access to everything in your custom project
    p, role:project-admin, applications, *, */*, allow
    p, role:project-admin, projects, *, cloudprotect, allow

    # Readonly role → Can only view apps/projects
    p, role:project-readonly, applications, get, cloudprotect/*, allow
    p, role:project-readonly, projects, get, cloudprotect, allow

    # ArgoCD RBAC Users (Keycloak group → role:admin)
    g, admin, role:project-admin
    g, readonly, role:project-readonly

  policy.default: 
```
> [!NOTE]
> Due to **policy.default: role:readonly** line in above yaml.
> Any user without an explicit deny automatically gets ArgoCD’s built-in readonly role, which has global read access to everything (all apps/projects/repos). 
>
>  So even though you wrote p, role:project-readonly, ..., your readuser is still inheriting the global role:readonly.
>
> Setting policy.default: "" → denies access by default unless explicitly granted.


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

## ✅ Step 2: Restart ArgoCD API Server

Changes take effect after restart:

```bash
kubectl rollout restart deployment argocd-server -n cd
```

## ✅ Step 3: Login as Different Users

If you’re using ArgoCD’s local accounts (not Keycloak yet):

1. Create users in argocd-cm:

```bash
kubectl edit configmap argocd-cm -n cd
```

* Add under data::

```Yaml:
  accounts.your-group-name: apiKey,login
  accounts.your-group-name: apiKey,login
  oidc.config: |
    name: Keycloak
    issuer: https://keycloak.example.com/realms/your-realm-name
    clientID: argocd
    clientSecret: your-client-secret-value
    requestedScopes: ["openid", "profile", "email", "groups"]
    rootCA: |
      -----BEGIN CERTIFICATE-----
      ---Your CA certificate---
      -----END CERTIFICATE-----
    logoutURL: https://keycloak.example.com/realms/your-realm-name/protocol/openid-connect/logout
```

> [!NOTE]
> your-group-name - Group created in keycloak.
> 
> keycloak.example.com - Your keyCloak ip or Domain name with port
> 
> your-realm-name - realm name which you used to create client in your keycloak.
> 
> your-client-secret-value - your client secret value

2. Restart API server again:

```bash
kubectl rollout restart deployment argocd-server -n cd
```


