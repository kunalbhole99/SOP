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

## Restart ArgoCD API Server

Changes take effect after restart:

```bash
kubectl rollout restart deployment argocd-server -n cd
```

## ✅ Step 2: add keycloak OIDC config data in argocd-cm configmap


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


## ✅ Step 2: Create secret for keycloak.crt

1. Go to the secret path and run command.

```bash
kubectl -n argocd create secret generic keycloak-ca-secret-for-argocd --from-file=keycloak.crt
```

2. Mount secret in argocd-server deployment.

under volumes:
```bash
- name: keycloak-ca
        secret:
          defaultMode: 420
          secretName: keycloak-ca-secret-for-argocd
```

under volumeMounts:
```bash
- mountPath: /app/config/tls/172.31.52.46.crt
          name: keycloak-ca
          subPath: 172.31.52.46.crt
```

test Data:

```bash
  accounts.admin: apiKey,login
  accounts.readonly: apiKey,login
  oidc.config: |
    name: Keycloak
    issuer: https://172.31.52.46/realms/argocd
    clientID: argocd
    clientSecret: lTPnIdrFC9Zw0VZQ29qB8gbIY7BTWHxg
    requestedScopes: ["openid", "profile", "email", "groups"]
    rootCA: |
      -----BEGIN CERTIFICATE-----
      MIIDmzCCAoOgAwIBAgIUL8UjfO71mew4sf9mY2jj06bpx7UwDQYJKoZIhvcNAQEL
      BQAwWzELMAkGA1UEBhMCVVMxDjAMBgNVBAgMBVN0YXRlMQ0wCwYDVQQHDARDaXR5
      MRUwEwYDVQQKDAxPcmdhbml6YXRpb24xFjAUBgNVBAMMDTE3Mi4zMS41Mi4xNDYw
      HhcNMjUwOTA4MDk1NjEyWhcNMjYwOTA4MDk1NjEyWjBbMQswCQYDVQQGEwJVUzEO
      MAwGA1UECAwFU3RhdGUxDTALBgNVBAcMBENpdHkxFTATBgNVBAoMDE9yZ2FuaXph
      dGlvbjEWMBQGA1UEAwwNMTcyLjMxLjUyLjE0NjCCASIwDQYJKoZIhvcNAQEBBQAD
      ggEPADCCAQoCggEBALp+pJv92h2up2cGKiPRx0KxpZ/Uo+2VHJyShzqOHV1i9Zb0
      2HDqKkkNVC8v4F2/pXfaqSTlST4ThB4LOD3LO/Z/vMzrOuNSu+1c6BGwEosoVyeQ
      he1IV86mv+37oe4O2iwZ98EmNaHrBxw5eXm1YIAcLLtD+mKibnp5bL078q0DPHe+
      JXFNjR5i2HO/1dpICSstFIsdEoaCYYodsfOeNp5TBdMJRy8L4uZaXjRjtehmT3VQ
      k3sat1LG+9vnaCW00FqyXUVYqaXKsMHMdYbQKYQKBZelwiODStZldvc4P3BO5VRp
      DRlgb+w3rPzXT4/H2ESTUEzSVO+UaJ+SwWYqtU0CAwEAAaNXMFUwDgYDVR0PAQH/
      BAQDAgSwMBMGA1UdJQQMMAoGCCsGAQUFBwMBMA8GA1UdEQQIMAaHBKwfNC4wHQYD
      VR0OBBYEFNQ7zRFsYrk/wZoh/U7LmQ00vB0pMA0GCSqGSIb3DQEBCwUAA4IBAQA6
      kG+Dg0uphpOInebGY7zHrcDqvh5g4twln9KHSJciOJQNa0HtDjcJbM3ra9TmvmUh
      PJKKMV07QuipPtBp5X+NCqgUsY72XgZvc0QiMUkNvDBaNJEgC0Qo43+EKIh9DUzx
      X//vjTRh2pjVQokpsMKexby+h9O8tf4iLEExcxXqpPGSmQEYj0lLbSA+esp9Kx8P
      x2lMYLEDJhCNADJMOC7NonIYUMzZ1izlF3wus994S569SDXjk2+pJGkg7hQVP7If
      LhRB3XE55NdZA5WfqQ7HzmdaIl20vMT5W8EzRvbTmyRkt/4HAJHWf4zioHbiYheM
      ZBtrSHb03RWl3WHkmOK/
      -----END CERTIFICATE-----
    logoutURL: https://172.31.52.46/realms/argocd/protocol/openid-connect/logout

```

