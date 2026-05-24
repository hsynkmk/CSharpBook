# 19 — Security & Auth

## ⚡ 30-second answer

**Authentication (authn)** = *who you are*; **authorization (authz)** = *what you're allowed to do* — always in that order. Modern APIs authenticate with **JWT** bearer tokens (stateless, signed claims) issued via **OAuth 2.0 / OpenID Connect** by an identity provider; browser apps often use **cookies**. The server validates the token's **signature, issuer, audience, expiry**, builds a **`ClaimsPrincipal`**, and authorizes via **policies/roles/claims**. Key rules: **never roll your own crypto** (use the BCL), **never store secrets in code**, and protect against **CSRF** (cookies) and token leakage.

---

## Core mechanics

**Authn vs Authz** — two halves:
```csharp
app.UseAuthentication();   // validate credentials/token → build ClaimsPrincipal (who)
app.UseAuthorization();    // check policies/roles/claims (what) — must come AFTER authn ([16])
```

**JWT** (JSON Web Token) — three base64 parts `header.payload.signature`:
- **Stateless**: the server validates the **signature** (with the issuer's key) and the claims — no server-side session lookup.
- Validate: **signature**, **issuer** (`iss`), **audience** (`aud`), **expiry** (`exp`), not-before. Short-lived access token + **refresh token** for renewal.

**OAuth 2.0 / OIDC**:
- **OAuth2** = delegated **authorization** (get a token to access an API). **OIDC** = layer on top for **authentication** (an `id_token` proving who the user is).
- **Authorization Code flow + PKCE** is standard for web/mobile/SPA (PKCE prevents code interception).

**Claims & policies**:
```csharp
builder.Services.AddAuthorization(o => o.AddPolicy("CanEdit",
    p => p.RequireClaim("permission", "orders:edit")));
[Authorize(Policy = "CanEdit")] public IActionResult Edit() => …;
```

**Cookies**: server sets an encrypted auth cookie; the browser sends it automatically → vulnerable to **CSRF** (use anti-forgery tokens).

**Secrets & crypto**: passwords are **hashed** (PBKDF2/bcrypt/Argon2 via ASP.NET Identity), never encrypted/plaintext. Use **`IDataProtector`** for tokens/cookies, BCL crypto for HMAC/encryption — **don't invent algorithms**. Secrets live in user-secrets (dev) / **Key Vault** + managed identity (prod).

---

## Comparison tables

| | JWT (bearer) | Cookie |
|---|---|---|
| State | **stateless** (self-contained) | server/encrypted ticket |
| Sent | `Authorization: Bearer` header | automatic by browser |
| Best for | APIs, SPAs, mobile, service-to-service | server-rendered web apps |
| CSRF risk | low (header, not auto-sent) | **yes** (need anti-forgery) |
| Revocation | hard (valid until expiry) | easy (server-side) |

| | OAuth 2.0 | OpenID Connect |
|---|---|---|
| Purpose | **authorization** (access token) | **authentication** (id token) |
| Answers | "can this app access X?" | "who is this user?" |

| Authorization style | Use |
|---|---|
| **Role-based** (`[Authorize(Roles=…)]`) | coarse buckets (Admin/User) |
| **Policy-based** (claims/requirements) | fine-grained, testable, recommended |

---

## 🪤 Traps & gotchas

- **Confusing authn and authz** — and ordering them wrong (`UseAuthorization` before `UseAuthentication` → no identity to check) ([16](16-AspNetCore.md)).
- **Rolling your own crypto / hashing** — almost always insecure. Use ASP.NET Identity hashing, `IDataProtector`, and BCL primitives.
- **Storing passwords encrypted or plaintext** — hash them (one-way, salted, slow KDF). Encryption is reversible → wrong for passwords.
- **Secrets in source/appsettings** — leak via git. Use user-secrets/Key Vault + **managed identity** (no secret to store).
- **Not validating all JWT claims** — skipping `aud`/`iss`/`exp` validation lets tokens from other issuers/apps in. Validate signature **and** standard claims.
- **Long-lived access tokens** — hard to revoke if leaked. Keep them short; use refresh tokens.
- **CSRF on cookie auth** — a malicious site can trigger authenticated requests. Use anti-forgery tokens (and `SameSite` cookies).
- **Tokens in `localStorage`** (SPA) — exposed to XSS. Prefer secure, httpOnly cookies / BFF pattern.
- **Authorization only in the UI** — hiding a button isn't security; enforce on the server/API.

---

## ❓ Likely questions

**Q: Authentication vs authorization?**
A: Authentication establishes identity (who you are); authorization decides permissions (what you can do). Authn first, then authz.

**Q: What is a JWT and how is it validated?**
A: A signed, base64 token (header.payload.signature) carrying claims. The server validates the signature (issuer's key) plus `iss`, `aud`, `exp`. Stateless — no session lookup.

**Q: OAuth 2.0 vs OpenID Connect?**
A: OAuth2 is for authorization (issuing access tokens to call APIs). OIDC adds an identity layer (id_token) for authentication. OIDC = OAuth2 + "who is the user".

**Q: JWT vs cookie auth — when each?**
A: JWT for APIs/SPAs/mobile/service-to-service (stateless, header-based, CSRF-resistant). Cookies for server-rendered web apps (auto-sent, easy revocation, but need CSRF protection).

**Q: How should passwords be stored?**
A: Hashed with a slow, salted KDF (PBKDF2/bcrypt/Argon2 — ASP.NET Identity does this). Never encrypted or plaintext; hashing is one-way.

**Q: Role-based vs policy-based authorization?**
A: Roles are coarse buckets; policies are fine-grained, claim/requirement-based, composable, and testable — recommended for non-trivial rules.

**Q: What's CSRF and who's affected?**
A: Cross-site request forgery — a malicious site triggers authenticated requests using the victim's auto-sent cookie. Cookie auth is vulnerable; mitigate with anti-forgery tokens and `SameSite`. Bearer-token APIs are largely immune.

**Q: Where do you store secrets?**
A: Never in code/appsettings. User-secrets in dev; a secret manager (Key Vault) + managed identity in prod, so there's no secret to leak.

---

## 🎓 Senior Extra

- **PKCE**: the auth-code flow for public clients (SPA/mobile) sends a hashed `code_challenge`, so an intercepted authorization code is useless without the verifier — now standard even for confidential clients.
- **Token validation internals**: signature via the issuer's published **JWKS** (key rotation via key refresh); validate `aud`/`iss`/`exp`/`nbf`; allow clock skew. Symmetric (HMAC) vs asymmetric (RSA/ECDSA, public-key) signing.
- **Claims transformation** (`IClaimsTransformation`) enriches the principal post-authentication (map external claims to internal roles/permissions).
- **`IDataProtection`**: encrypts cookies/anti-forgery/tokens; in a scaled-out app the **key ring must be shared** (Redis/Key Vault/file share) or instances can't read each other's cookies.
- **Authorization handlers/requirements**: implement `AuthorizationHandler<TRequirement>` for **resource-based** authz (e.g., "can edit *this* order" — owner check), beyond static policies.
- **mTLS / API keys / managed identity** for service-to-service; OAuth client-credentials flow for machine clients.
- **Defense in depth**: validate/encode inputs (XSS), parameterize queries (EF Core does this), security headers (HSTS, CSP), rate-limit auth endpoints ([16](16-AspNetCore.md)), log auth events ([20](20-Observability-Messaging-Background.md)).
- **Refresh-token rotation + revocation lists** handle the JWT revocation gap; short access tokens + server-side refresh state.

→ Deeper: [`../DotNetBook/10-Identity/`](../DotNetBook/10-Identity/README.md)
