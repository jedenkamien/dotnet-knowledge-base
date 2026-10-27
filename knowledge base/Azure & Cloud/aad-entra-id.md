# Azure Active Directory / Microsoft Entra ID

## What It Is
Microsoft's cloud **identity and access management** service. Now rebranded **Microsoft Entra ID** (same product). Stores users, groups, and applications; issues tokens for authentication and authorization.

## Tenants
- A **tenant** is a dedicated AAD instance for an organization.
- Each Azure subscription is associated with exactly one tenant.
- Users and apps live in a tenant; trust between tenants enables B2B / multi-tenant apps.

## Core Concepts

| Term | Meaning |
|------|---------|
| User | Person identity |
| Group | Collection of users (security / Microsoft 365) |
| Application (App Registration) | Definition of an app that uses AAD |
| Service Principal | Instance of an app in a tenant (the "user" the app logs in as) |
| Managed Identity | Auto-managed service principal for Azure resources — no secrets to handle |
| Role | Permission set (built-in or custom) assigned via RBAC |

## Authentication Protocols
- **OAuth 2.0** — authorization framework. Answers: *"what can this app do on your behalf?"* Issues an **access_token** scoped to specific permissions. Does NOT tell you who the user is.
- **OpenID Connect (OIDC)** — identity layer on top of OAuth 2.0. Answers: *"who is this user?"* Adds an **id_token** with user claims. Most modern apps use both together.
- **SAML 2.0** — legacy SSO for enterprise apps
- **WS-Federation** — older Microsoft protocol

### OAuth 2.0 vs OIDC in practice
- Login / "sign in with Google" → OIDC (you need the user's identity)
- Calling an API on the user's behalf → OAuth (you need the access token)

### OIDC is a protocol, not a company
OIDC is an open standard that anyone can implement. The ecosystem:
- **Identity Providers (IdPs)** — implement the protocol: Google, Facebook, Microsoft Entra ID, Okta, Auth0, etc.
- **Relying Parties** — your app/website that trusts and consumes the tokens

When you click "Sign in with Google", the website redirects you to Google's OIDC endpoint. You authenticate there; Google sends back a signed `id_token`. The website never sees your password — it only receives and validates the JWT.

When you log into a third-party site with your **Microsoft account**, the same applies: the site redirects to `login.microsoftonline.com`, Entra ID handles authentication (including MFA, Conditional Access), and returns a signed token.

## Token Types
- **id_token** — proves who the user is (OIDC); contains **claims** about the user
- **access_token** — grants access to an API (OAuth, JWT format)
- **refresh_token** — get new access tokens without re-login

### id_token claims
The `id_token` is a JWT containing user claims, for example:
- `sub` (**subject**) — stable, unique user ID within that IdP (opaque GUID/string, not email). Always present.
- `email`, `email_verified`
- `name`, `given_name`, `family_name`
- `picture` — a **URL** pointing to the profile image (not the image itself — no base64). The app fetches the image separately if needed.

The IdP decides what claims to include. The receiving app can control what's requested via **scopes** (e.g., `openid` = minimal; `profile` + `email` = more).

## Common Flows
- **Authorization Code + PKCE** — SPAs, mobile, web apps
- **Client Credentials** — service-to-service (no user)
- **On-Behalf-Of (OBO)** — API calls another API as the user
- **Device Code** — CLI / devices without browser

## Managed Identities (very useful)
An Azure resource (VM, Function, App Service) can be assigned a **Managed Identity** — you enable it with one toggle in the portal or a config line in IaC. Azure then automatically creates a Service Principal for it in Entra ID.

### How it works under the hood
There are no secrets exposed to you — but a secret *does exist*, managed and rotated entirely by Azure. The flow:
1. Your code calls `new DefaultAzureCredential()` and uses it.
2. The SDK internally calls the **IMDS endpoint** (`http://169.254.169.254/metadata/identity/oauth2/token`).
3. Azure verifies the request came from that specific resource (link-local address — only reachable from inside the Azure host, not from outside).
4. Azure's token service issues a short-lived **JWT access token** signed by Entra ID.
5. Your code uses that JWT to call Azure APIs (Key Vault, Storage, etc.) normally.

### Local development vs Azure
- **In Azure** → SDK hits IMDS, gets a real token via Managed Identity.
- **Locally** → IMDS doesn't exist; `DefaultAzureCredential` falls through to the next option (Visual Studio cached login, Azure CLI, etc.) and gets a token *as you*, the developer. VS stores a token from when you signed in; the SDK uses that (via MSAL) to call Entra ID and get an access token.

Same code, different credential source — that's the elegance of it.

```csharp
var credential = new DefaultAzureCredential();
var client = new SecretClient(new Uri(vaultUrl), credential);
```

`DefaultAzureCredential` tries (in order): env vars → managed identity → Visual Studio → Azure CLI → etc.

## Roles & RBAC
- **Azure RBAC**: controls access to Azure *resources* (subscriptions, resource groups).
- **AAD Roles**: control access to *AAD itself* (User Admin, Global Admin).
- **App Roles / Scopes**: defined per app, embedded in tokens for authorization.

## B2B vs B2C

**B2B (Business-to-Business)**
- Your Entra ID tenant invites **external users** (from other companies) as **guests**.
- The guest authenticates with *their own* organization's IdP — your tenant trusts the result.
- Use case: a partner company's employee needs access to your internal app or SharePoint.

**B2C (Business-to-Consumer)**
- A **completely separate Azure product** (not just a feature of Entra ID).
- Designed for millions of end-customers with social logins (Google, Facebook, Apple) or local email/password accounts.
- Full control over sign-up/sign-in UI flows, password reset, etc.
- Your app acts as an IdP that federates to Google/Facebook underneath.
- Use case: a public-facing e-commerce or mobile app where customers register themselves.

**Key difference:** B2B is for *known business partners* (managed guests); B2C is for *anonymous public users* (self-service registration at scale).

## Conditional Access
Policy engine: "if user from untrusted location, require MFA." Combines identity + signals (device, location, risk) before granting access.
