This guide provides a comprehensive, production-ready manual for integrating **Apache NiFi 2.8.0** with **Keycloak 26.5.6** using OpenID Connect (OIDC). This setup moves away from legacy XML-based providers to the modern native OIDC framework introduced in NiFi 2.x.

---

# 🛡️ Apache NiFi 2.8.0 & Keycloak 26.5.6 OIDC Integration Guide


### Prerequisites
- Keycloak 26.5.6
- Apache NiFi 2.8.0
- Java 11 or later
- Ports 8080 (Keycloak) and 8443 (NiFi) available

## 📌 Architecture Overview
In this architecture, Keycloak acts as the **Identity Provider (IdP)**, managing authentication and user attributes. Apache NiFi acts as the **Service Provider (SP)**, enforcing authorization based on the claims (identity and groups) received from Keycloak.



---

## 🔑 Phase 1: Keycloak Configuration

1. Install and Start Keycloak

#### Download Keycloak
```bash
wget https://github.com/keycloak/keycloak/releases/download/26.5.6/keycloak-26.5.6.tar.gz
```
```bash
tar -xzf keycloak-26.5.6.tar.gz
```
```bash
cd keycloak-26.5.6/bin/
```
#### Start Keycloak in development mode

```bash
./kc.sh start-dev --http-port=8080
```
Access Keycloak Admin Console: http://localhost:8080

Create initial admin user by providing user details

### Phase 1 quick links
1. [Create Realm](#1-create-realm-order-1-realm)
2. [Create Client](#2-create-client-order-2-client)
3. [Create Roles](#3-create-roles-order-3-roles)
4. [Create Groups](#4-create-groups-order-4-groups)
5. [Create Users](#5-create-users-order-5-users)
6. [Configure Group Membership Mapper](#6-configure-group-membership-mapper)

![Keycloak first-time setup](images/keycloak-firsttime-page.png)

### 1. Create Realm
* CLick on **Manage Realms** > **Create Realm**.
* Realm name: `NiFi`
* Enabled: `ON`

![Create realm](images/create-realm-page.png)  

![Create realm step 2](images/create-realm-page-2.png)

### 2. Create Client
* Under the NiFi realm, create a new client.
* Click on **Clients** > **Create client**.



* Access Type: `confidential`
* Valid Redirect URIs: `https://localhost:8443/nifi-api/access/oidc/callback`
* Standard Flow: `ON`
* Direct Access Grants: `ON`

Screenshots:
- ![Create client 1](images/create-client-page-1.png)

In General Settings, set the following:
* Client Type: `OpenID Connect`
* Client ID: `nifi`

- ![Create client 2](images/create-client-page-2.png)

In Capability config, set the following:
* Client authentication : `ON`
* Authorization : `OFF`
* Authentication flow :
    - Standard flow : `Check`
    - Direct access grants : `Check`
    - Implicit flow : `Un-Check`
    - Service account roles : `Un-Check`
    - Standard Token Exchange : `Un-Check`
    - OAuth 2.0 Device Authorization Grant : `Un-Check`
    - OIDC CIBA Grant : `Un-Check`

- ![Create client 3](images/create-client-page-3.png)

Login Settings (Localhost)
 - Root URL: https://localhost:8443
 - Home URL: /nifi
 - Valid redirect URIs: https://localhost:8443/nifi-api/access/oidc/callback
 - Valid post logout redirect URIs: https://localhost:8443/nifi/logout-complete
 - Web origins: + (or https://localhost:8443)

- ![Create client 4](images/create-client-page-4.png)

<!--
- ![Create client 5](images/create-client-page-5.png)
- ![Create client 6](images/create-client-page-6.png)
- ![Create client 7](images/create-client-page-7.png)
- ![Create client 8](images/create-client-page-8.png)
- ![Create client 9](images/create-client-page-9.png)
- ![Create client 10](images/create-client-page-10.png)
- ![Create client 11](images/create-client-page-11.png) -->

### 3. Create Roles
* Go to **Roles** > **Add Role**.
* Add roles like `nifi-admin`, `nifi-view-only` or `nifi-user`, etc.

![Create role 1](images/create-role-page-1.png)
![Create role 2](images/create-role-page-2.png)

### 4. Create Groups
* Go to **Groups** > **New**.
* Create groups like `nifi-admins`, `nifi-operators`, `nifi-read-only`.

![Create group 1](images/create-group-page-1.png)
* Provide the Group Names: NiFi Admins, NiFi Operators, NiFi Users/NiFi Read-Only.

![Create group 2](images/create-group-page-2.png)

* Assign roles to groups. For example, assign `nifi-admin` role to `Nifi Admins` group, `nifi-view-only` role to `Nifi Users` group, etc.

![Create group 3](images/create-group-page-3.png)
![Create group 4](images/create-group-page-4.png)
![Create group 5](images/create-group-page-5.png)
<!-- ![Create group 6](images/create-group-page-6.png) -->

### 5. Create Users
* Go to **Users** > **Add user**.
* Set username, email, first name, last name.
* Set initial password and required actions (update password on first login).
* Assign roles and groups.

![Create user 1](images/create-user-page-1.png)
![Create user 2](images/create-user-page-2.png)

### 6. Configure Group Membership Mapper
To allow NiFi to authorize users based on Keycloak groups, map group membership into OIDC token.
* Navigate to **Clients** > **nifi** > **Client Scopes** > `nifi-dedicated`.
* Click **Add Mapper** > **By Configuration** > **Group Membership**.
* **Name**: `groups`.
* **Token Claim Name**: `groups`.
* **Full group path**: `OFF`.
* **Add to ID Token**: `ON`.

---

## ⚙️ Phase 2: Apache NiFi Configuration

### 1. `nifi.properties`
Native OIDC in NiFi 2.x is configured entirely within this file. Update the following properties:

```properties
nifi.security.user.authorizer=managed-authorizer
nifi.security.allow.anonymous.authentication=false
nifi.security.user.login.identity.provider=
nifi.security.user.jws.key.rotation.period=PT1H

# Optional but recommended
nifi.security.user.oidc.scopes=openid,email,profile

# OpenId Connect SSO Properties #
nifi.security.user.oidc.discovery.url=http://localhost:8080/realms/NiFi/.well-known/openid-configuration

nifi.security.user.oidc.connect.timeout=5 secs
nifi.security.user.oidc.read.timeout=5 secs
nifi.security.user.oidc.client.id=nifi
nifi.security.user.oidc.client.secret=CLIENT_SECRET_FROM_KEYCLOAK
nifi.security.user.oidc.preferred.jwsalgorithm=
nifi.security.user.oidc.additional.scopes=openid
nifi.security.user.oidc.claim.identifying.user=preferred_username
nifi.security.user.oidc.fallback.claims.identifying.user=email,sub
```

### 2. `authorizers.xml`
Define the initial administrator and the user group provider.

```xml
<authorizers>
    <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>
		<property name="Initial User Identity 1">nifi-admin</property>
		<property name="Initial Admin Identity">nifi-admin</property>
    </userGroupProvider>
<accessPolicyProvider>
    <identifier>file-access-policy-provider</identifier>
    <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
    <property name="User Group Provider">file-user-group-provider</property>
    <property name="Authorizations File">./conf/authorizations.xml</property>
    <property name="Legacy Authorized Users File"></property>
    <property name="Initial Admin Identity">nifi-admin</property>
</accessPolicyProvider>
    <authorizer>
        <identifier>managed-authorizer</identifier>
        <class>org.apache.nifi.authorization.StandardManagedAuthorizer</class>
        <property name="Access Policy Provider">file-access-policy-provider</property>
    </authorizer>
</authorizers>
```

### 3. `login-identity-providers.xml`
In NiFi 2.x, this file must **not** contain the `OidcIdentityProvider` class.

```xml
<loginIdentityProviders>
    </loginIdentityProviders>
```

---

## 🚀 Phase 3: Deployment
To apply these changes cleanly, follow this sequence:
1.  **Stop NiFi**.
2.  **Delete** `conf/users.xml` and `conf/authorizations.xml` to force NiFi to regenerate them using the new "Initial Identity" settings.
3.  **Start NiFi**.

---

## 🛠️ Troubleshooting Guide

| Issue | Symptom | Root Cause & Resolution |
| :--- | :--- | :--- |
| **Class Not Found Error** | NiFi fails to start with `Login Identity Provider class [org.apache.nifi.oidc.OidcIdentityProvider] not registered` | **Cause**: You still have OIDC defined in `login-identity-providers.xml`. **Fix**: Remove the `<provider>` block from that XML file; it is deprecated in NiFi 2.x. |
| **403 Forbidden (Identity Mismatch)** | User logs in via Keycloak but sees a rotating "Loading" screen or "Access Denied" | **Cause**: The identity string from Keycloak (e.g., `nifi-admin`) does not match the `Initial Admin Identity` in `authorizers.xml`. **Fix**: Check `nifi-user.log` for the exact identity string and update `authorizers.xml`. |
| **403 Forbidden (No Policies)** | User is authenticated but cannot view the flow | **Cause**: The `Initial Admin Identity` was missing from the `accessPolicyProvider` section of `authorizers.xml`. **Fix**: Add that property and delete `users.xml`/`authorizations.xml` before restarting. |
| **Groups Not Appearing** | `nifi-user.log` shows `Groups []` even though the user is in a Keycloak group | **Cause**: The Protocol Mapper in Keycloak is missing or the `Token Claim Name` does not match `nifi.security.user.oidc.claim.groups`. **Fix**: Ensure both are set to `groups`. |
| **Stale Authorization** | Changes to `authorizers.xml` are ignored | **Cause**: NiFi only reads "Initial" settings if `users.xml` and `authorizations.xml` do not exist. **Fix**: Delete those two files and restart. |
