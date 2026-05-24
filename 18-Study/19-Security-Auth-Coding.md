# 19 — Security & Auth — Coding Questions

> Find the vulnerability / predict. (Concepts: [19-Security-Auth.md](19-Security-Auth.md))

---

### Q1 — Password storage
```csharp
user.Password = Encrypt(plainPassword, key);   // store reversibly
```
<details><summary>Answer</summary>

**Wrong** — passwords must be **hashed** (one-way, salted, slow KDF: PBKDF2/bcrypt/Argon2), not **encrypted** (reversible → if the key leaks, all passwords leak). Use ASP.NET Identity's hasher. Never store recoverable passwords.
</details>

---

### Q2 — JWT validation gaps
```csharp
var handler = new JwtSecurityTokenHandler();
var token = handler.ReadJwtToken(jwt);
var userId = token.Claims.First(c => c.Type == "sub").Value;   // trust it?
```
<details><summary>Answer</summary>

**Vulnerable** — `ReadJwtToken` only **parses**; it does **not validate** the signature/issuer/audience/expiry. An attacker can forge any claims. **Fix:** validate via `TokenValidationParameters` (signature with the issuer key, `iss`, `aud`, `exp`) — only trust claims from a *validated* token.
</details>

---

### Q3 — Roll-your-own crypto
```csharp
string Hash(string s) {
    int h = 0; foreach (var c in s) h = h * 31 + c; return h.ToString();
}
```
<details><summary>Answer</summary>

**Never roll your own crypto.** This is a trivial, reversible, collision-prone hash — useless for security. Use BCL primitives (`SHA256`, `HMACSHA256`) for hashing/integrity and a proper KDF for passwords. Custom crypto is almost always broken.
</details>

---

### Q4 — Authorization in the UI only
```razor
@if (user.IsAdmin) { <button @onclick="DeleteAll">Delete All</button> }
@* server endpoint: *@
app.MapPost("/admin/delete-all", () => DeleteEverything());   // no [Authorize]
```
<details><summary>Answer</summary>

**Hiding the button isn't security** — anyone can call `POST /admin/delete-all` directly. **Fix:** enforce authorization on the **server**: `[Authorize(Policy="Admin")]` (or `.RequireAuthorization(...)`). UI checks are UX, not a boundary.
</details>

---

### Q5 — Secret in config
```csharp
// appsettings.json:  { "ConnectionStrings": { "Db": "Server=...;Password=P@ss123;" } }
```
<details><summary>Answer</summary>

**Secret leak** — committed to source control. **Fix:** user-secrets in dev; **Key Vault + managed identity** in prod (so there's no secret stored in the app at all). Reference it via configuration; never commit credentials.
</details>

---

### Q6 — CSRF exposure
```csharp
// Cookie-authenticated MVC app, state-changing POST with no anti-forgery token.
[HttpPost] public IActionResult Transfer(decimal amount) { ... }
```
<details><summary>Answer</summary>

**CSRF-vulnerable** — a malicious site can auto-submit a form using the victim's auto-sent auth cookie. **Fix:** anti-forgery tokens (`[ValidateAntiForgeryToken]` / `@Html.AntiForgeryToken()`) and `SameSite` cookies. (Bearer-token APIs are largely immune since the token isn't auto-sent.)
</details>

---

### Q7 — Tokens in localStorage (senior)
```javascript
localStorage.setItem("access_token", token);   // SPA
```
<details><summary>Answer</summary>

**XSS-exposed** — any injected script can read `localStorage`. Prefer **secure, httpOnly cookies** (not script-readable) or a **BFF (Backend-for-Frontend)** pattern that keeps tokens server-side. Pair with short token lifetimes + refresh-token rotation.
</details>

---

### Q8 — OAuth2 vs OIDC choice (senior)
```csharp
// You need to know WHO the user is (their identity) for your app's login. Which protocol?
```
<details><summary>Answer</summary>

**OpenID Connect (OIDC)** — it adds an **`id_token`** (authentication: who the user is) on top of OAuth2. Plain **OAuth2** is for **authorization** (an access token to call an API), not for proving identity. Use Authorization Code flow **+ PKCE** for web/SPA/mobile.
</details>
