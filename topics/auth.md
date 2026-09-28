# Authentication and authorization

> Authentication proves who you are; authorization decides what you're allowed to do once you're in — two different jobs that most login bugs get confused between.

[← Back to Auth & identity](../README.md#auth--identity)

## Why this matters

Walk into a hotel and the first thing that happens at the front desk is an ID check: the clerk looks at your passport, matches it to a booking, and confirms you are who you say you are. That's **authentication**. The clerk doesn't then hand you a master key to every room in the building. Instead you get a key card programmed for exactly one door — your room, maybe the gym, maybe the pool if your package includes it, and nothing else. Deciding which doors that card opens is **authorization**. Two different desks could do this job (checking ID vs programming a key card), and in software they usually are two different systems too.

Keeping them separate matters because a bug in one is a completely different kind of bug than a bug in the other. If a bookshop's login page lets someone in with the wrong password, that's an authentication failure — the ID check itself is broken. If a logged-in customer can view someone else's order history by editing a number in the URL, that's an authorization failure — the ID check worked fine, but nobody checked whether that "room" was actually theirs to enter. Most of this page is organized around that split: first the ways of proving who you are, then how a system remembers that proof, then how it lets other systems rely on that proof, and finally how it decides what the proven identity may actually do.

## The map

```mermaid
flowchart LR
    subgraph Prove["Proving who you are"]
        P1[Passwords]
        P2[MFA]
        P3[Passkeys]
        P4[Magic links]
        P5[Social login]
    end
    subgraph Remember["Remembering you"]
        R1[Sessions]
        R2[Tokens]
    end
    subgraph Delegate["Delegating and federating"]
        D1["OAuth 2.0"]
        D2["OpenID Connect"]
        D3["SAML / SSO"]
    end
    subgraph Decide["Deciding what you may do"]
        A1[RBAC]
        A2[ABAC]
        A3[ReBAC]
        A4[Row-level security]
    end
    Prove --> Remember
    Remember --> Delegate
    Delegate --> Decide
```

Read it left to right: you prove who you are once, the system remembers that for the rest of your visit, sometimes that proof gets handed to other apps or services, and every single request along the way gets checked against a set of rules for what that identity may actually touch.

## Proving who you are

### Passwords

**In one line:** a secret only you (should) know, checked against a scrambled copy the server keeps — never against your typed password matched directly.

**How it works:** think of hashing like a paper shredder. You feed a document (your password) in, and it comes out as confetti (a hash) — there's no way to un-shred the confetti back into the original page. The server never stores your actual password; it stores the confetti. When you log in later, it shreds what you typed with the exact same shredder and checks whether the confetti pattern matches what it kept on file. If an attacker steals the whole database, they get a pile of confetti, not a stack of readable documents.

Two details make this safe in practice. First, a **salt** — a random string mixed in before shredding, unique per user — so two people who both pick "password123" produce completely different confetti; without a salt, an attacker who cracks one hash instantly knows every account sharing that password. Second, the shredder itself has to be deliberately slow: **bcrypt** and **argon2** (OWASP's current recommendation, argon2id specifically) are built to take a noticeable fraction of a second per hash, unlike fast general-purpose hashes like SHA-256 that are built for speed and are a bad fit here — a fast hash lets an attacker try billions of guesses a second against a stolen database; a slow one caps that at a few thousand.

```mermaid
sequenceDiagram
    participant C as Customer
    participant S as Bookshop server
    participant DB as Database

    Note over C,DB: Sign up
    C->>S: Choose password "R3adBooks!"
    S->>S: Generate a random salt, hash(password + salt) with bcrypt
    S->>DB: Store email + hash only, never the plain password

    Note over C,DB: Log in later
    C->>S: Type the password again
    S->>DB: Fetch the stored hash for this email
    S->>S: Hash the typed password the same way, with the stored salt
    S->>S: Compare the new hash to the stored hash
    S-->>C: Match = logged in, no match = "wrong password"
```

**Example:**

```js
// signup
const hash = await bcrypt.hash(plainPassword, 12); // 12 = cost factor
await db.users.insert({ email, passwordHash: hash });

// login
const user = await db.users.findByEmail(email);
const ok = await bcrypt.compare(plainPassword, user.passwordHash);
```

**Good for:**
- Any app where users are willing to remember or store a secret
- Combining with MFA for a second, independent layer

**Watch out for:**
- Never store plain text or reversibly encrypt a password — always a one-way, salted, slow hash
- Never write your own hashing scheme; use a maintained library (bcrypt, argon2, scrypt)

### Multi-factor authentication

**In one line:** a second, different kind of proof required on top of your password, so a leaked password alone isn't enough to get in.

**How it works:** it's the hotel asking for your ID *and* calling the phone number on the booking before handing over a key. The three common "second factors": a **TOTP** (time-based one-time password) app like Google Authenticator, which shares a secret with the server once and then both sides independently compute the same 6-digit code every 30 seconds from that secret plus the current time; **SMS codes**, which are easy to set up but weaker — phone numbers can be hijacked through SIM-swap attacks or intercepted, which is why NIST's guidance (SP 800-63B) labels SMS a "restricted" authenticator rather than a recommended one; and **push notifications**, where you tap "approve" on a phone you're already logged into elsewhere — convenient, but vulnerable to "MFA fatigue" attacks where an attacker spams approval requests hoping you tap "yes" just to make the notifications stop.

```mermaid
sequenceDiagram
    participant C as Customer
    participant S as Bookshop server
    participant A as Authenticator app

    C->>S: Correct password
    S-->>C: Password OK, enter your 6-digit code
    C->>A: Open authenticator app
    A->>A: Generate code from the shared secret + current time
    C->>S: Type the 6-digit code
    S->>S: Generate the same code independently and compare
    S-->>C: Codes match = logged in
```

**Example:** provisioning a TOTP secret into an authenticator app, usually shown as a QR code encoding a URI like:

```
otpauth://totp/Bookshop:alwin@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Bookshop
```

**Good for:**
- Any account worth protecting beyond a password, especially admin and payment-related accounts
- Layering on top of any of the other methods on this page

**Watch out for:**
- SMS-only MFA gives a false sense of security against a determined attacker
- Push-only MFA needs "number matching" (typing a code shown on the login screen into the app) to resist fatigue attacks, not just a bare "approve?" tap

### Passkeys / WebAuthn

**In one line:** passwordless login using public-key cryptography tied to your device, unlocked with your fingerprint, face, or PIN, and resistant to phishing by design.

**How it works:** it's like a lock that only your uniquely shaped key can open, and that key physically never leaves your pocket. When you register, your device creates a matching pair of keys — a private key that stays on the device (protected by your fingerprint or PIN) and a public key that gets sent to the server. Logging in means the server sends a random challenge, your device signs it with the private key after you unlock it biometrically, and the server checks the signature against the public key it stored. The private key is never transmitted, so there's nothing for a phishing site to steal — and critically, the signature is also bound to the real site's domain, so even a pixel-perfect fake login page can't get a valid signature out of your device. **WebAuthn** is the browser standard that makes this possible; a **passkey** is the credential itself, and modern passkeys sync across your devices through iCloud Keychain or Google Password Manager, so losing your phone doesn't lock you out.

```mermaid
sequenceDiagram
    participant C as Customer
    participant B as Browser
    participant S as Bookshop server

    Note over C,S: Registration, once
    C->>S: Sign up with a passkey
    S-->>B: Challenge to sign
    B->>C: Ask for fingerprint or face
    C->>B: Approve
    B->>B: Create a key pair, keep the private key on this device
    B->>S: Send the public key + signed challenge
    S->>S: Store the public key against this account

    Note over C,S: Login, every time after
    S-->>B: New challenge to sign
    B->>C: Ask for fingerprint or face
    C->>B: Approve
    B->>B: Sign the challenge with the private key
    B->>S: Send the signed challenge
    S->>S: Verify it with the stored public key
    S-->>C: Verified = logged in
```

**Example:** the server asks the browser to create a credential with options roughly like:

```json
{
  "rp": { "name": "Bookshop", "id": "bookshop.example.com" },
  "user": { "id": "dXNlcl8xODI=", "name": "alwin@example.com", "displayName": "Alwin" },
  "pubKeyCredParams": [{ "type": "public-key", "alg": -7 }],
  "authenticatorSelection": { "userVerification": "required" },
  "challenge": "Y2hhbGxlbmdlLWJ5dGVzLWZyb20tc2VydmVy"
}
```

**Good for:**
- Consumer apps that want both stronger security and a smoother login than passwords
- Killing phishing as an attack vector for the accounts that support it

**Watch out for:**
- Needs a fallback (another passkey device, or a recovery flow) for lost-device scenarios
- Not every browser/OS/device combination supports it equally well yet, so most apps offer it alongside, not instead of, another method

### Magic links and one-time codes

**In one line:** prove who you are by proving you control your email or phone, instead of remembering a password at all.

**How it works:** it's the hotel calling the number on file and only letting you in if you pick up. You type your email, the server generates a random, single-use token good for a short window (commonly 10–15 minutes), and emails you a link containing it. Clicking the link sends that token back to the server, which checks it's valid, unused, and not expired, then logs you in and immediately marks the token as spent so it can never be reused. A one-time code (a 6-digit number sent by email or SMS that you type in yourself) is the same idea without needing to click a link.

```mermaid
sequenceDiagram
    participant C as Customer
    participant S as Bookshop server
    participant E as Email inbox

    C->>S: Enter email, no password
    S->>S: Create a one-time token, expires in 15 minutes
    S->>E: Send a link containing the token
    C->>E: Open inbox, click the link
    E->>S: GET /auth/magic?token=...
    S->>S: Check the token is valid, unused, not expired
    S->>S: Mark the token as used
    S-->>C: Logged in, this token can never be used again
```

**Example:**

```
https://bookshop.example.com/auth/magic?token=b3ff7a2c-91de-4f6a-9c2e-7a1c9f0d5e33&exp=1719500000
```

**Good for:**
- Low-friction signup, or apps where users would otherwise forget or reuse weak passwords
- Removing the password database as an attack target entirely

**Watch out for:**
- Only as secure as the inbox or phone it's delivered to — a compromised email account is a compromised login
- Links must expire quickly and be single-use, or they become a standing backdoor

## Remembering who you are

### Server sessions and cookies

**In one line:** the server keeps a record that says "this person is logged in" and gives the browser a cookie holding the ID that points at that record.

**How it works:** it's a coat check. You hand over your coat once (your credentials), and get back a numbered ticket (the session ID). Every time you want your coat back, you show the ticket, and the attendant looks it up in the back room. The actual "coat" — who you are, your roles, when the session started — lives entirely on the server, in memory, Redis, or a database table. The browser only ever holds the meaningless ticket number, inside a cookie that gets sent back automatically on every request to that site.

Three cookie flags matter a lot here: **HttpOnly** stops any JavaScript running on the page from reading the cookie at all, which is what keeps a cross-site scripting (XSS) bug from being able to steal it; **Secure** means the cookie is only ever sent over HTTPS, never plain HTTP; and **SameSite** (`Lax` or `Strict`) controls whether the cookie gets sent along with requests that originate from a different site, which is the main defence against CSRF (see the attacks table further down).

```mermaid
sequenceDiagram
    participant C as Customer browser
    participant S as Bookshop server
    participant St as Session store

    C->>S: POST /login with email + password
    S->>S: Check credentials
    S->>St: Create a session record: user_182, expires in 24h
    S-->>C: Set-Cookie with the session ID (HttpOnly, Secure)
    Note over C,S: Every later request
    C->>S: GET /orders, cookie attached automatically
    S->>St: Look up the session ID
    St-->>S: user_182, still valid
    S-->>C: Here are your orders
```

**Example:**

```
Set-Cookie: session_id=8f14e45fceea167a5a36dedd4bea2543; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
```

**Good for:**
- A single server (or servers sharing a session store) rendering pages directly
- Instant revocation — delete the session record and the user is logged out immediately

**Watch out for:**
- Needs a shared session store (like Redis) once you run more than one server
- Rotate the session ID right after login to prevent session fixation

### JWT access tokens

**In one line:** a signed, self-contained token the server can verify with math instead of a database lookup, made of `header.payload.signature`.

**How it works:** it's a concert wristband stamped by the venue itself. Security at any door can check the stamp is genuine without phoning head office, but nobody can peel the wristband off and re-stamp it to claim a better seat. A JWT has three base64-encoded parts joined by dots: a **header** naming the signing algorithm, a **payload** holding claims (who the user is, their roles, when it expires), and a **signature** — the header and payload run through a signing algorithm with a secret key only the server knows, so any tampering with the payload breaks the signature check. Any service that has the verification key can check a JWT is genuine on its own, with no call back to a central database.

That self-contained property is also the catch: a JWT is valid until its `exp` claim says it isn't, and the server that issued it isn't tracking it anywhere. "Logging out" doesn't make a JWT stop working on its own — you either keep the access token's lifetime very short (minutes), or maintain a denylist of revoked tokens, which brings back the database lookup you were trying to avoid in the first place.

```mermaid
sequenceDiagram
    participant C as Customer app
    participant S as Auth server
    participant API as Orders API

    C->>S: Log in with email + password
    S->>S: Build the payload: sub, roles, exp
    S->>S: Sign header + payload with the secret key
    S-->>C: access_token (JWT)
    C->>API: GET /orders, Authorization: Bearer <token>
    API->>API: Verify the signature with the same key, check exp
    API-->>C: Signature valid, not expired = here are the orders
```

**Example:** a JWT looks like three dot-separated chunks —

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzE4MiIsImVtYWlsIjoiYWx3aW5AZXhhbXBsZS5jb20iLCJyb2xlcyI6WyJjdXN0b21lciJdLCJpYXQiOjE3MTk0ODAwMDAsImV4cCI6MTcxOTQ4MzYwMH0.4f8b2c9e1a7d3f0b6c5e8a1d2f4b7c9e
```

which decodes (header and payload only — the signature isn't readable text) to:

```json
{ "alg": "HS256", "typ": "JWT" }
```

```json
{
  "sub": "user_182",
  "email": "alwin@example.com",
  "roles": ["customer"],
  "iat": 1719480000,
  "exp": 1719483600
}
```

Anyone can decode and read a JWT's payload (it's just base64, not encrypted) — only the signature stops someone from *changing* it. Never put secrets in the payload.

**Good for:**
- Multiple independent services verifying identity without a shared database
- Stateless, horizontally scaled APIs

**Watch out for:**
- Hard to revoke individually — keep access tokens short-lived and pair with a refresh token
- Store them in an HttpOnly cookie where possible, not in `localStorage` (see Common mistakes)

### Refresh tokens and rotation

**In one line:** a longer-lived, more carefully guarded token whose only job is fetching a new access token once the old one expires.

**How it works:** the wristband (access token) is only good for an hour, but you also got a claim ticket kept in your hotel room safe (the refresh token) that you can exchange at the desk for a fresh wristband without showing ID all over again. **Rotation** means every time a refresh token is used, the server retires it and issues a brand-new one — so a refresh token is single-use, chained to the next one. This lets the server detect theft: if a retired (already-used) refresh token is ever presented again, that's a signal that two different parties have a copy of it — the legitimate one who used it already, and an attacker who's now trying to catch up — and the server can revoke the entire chain immediately.

```mermaid
sequenceDiagram
    participant C as Customer app
    participant S as Auth server

    C->>S: Log in
    S-->>C: access_token (1 hour), refresh_token (30 days)
    Note over C,S: 1 hour later, access_token has expired
    C->>S: POST /oauth/token, grant_type=refresh_token
    S->>S: Check the refresh_token is valid and unused
    S->>S: Retire the old refresh_token, issue a new one
    S-->>C: New access_token + new refresh_token
    Note over C,S: If the OLD (retired) refresh_token is reused
    C->>S: POST /oauth/token with the retired token
    S->>S: Reuse detected: this token was already retired
    S-->>C: Reject, revoke the entire token family
```

**Example:**

```
POST /oauth/token HTTP/1.1
Host: auth.bookshop.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=rt_9f2a7c1b...&client_id=bookshop_web
```

**Good for:**
- Keeping users logged in for weeks without keeping access tokens dangerously long-lived
- Detecting token theft through reuse detection

**Watch out for:**
- Refresh tokens need to be stored even more carefully than access tokens — they're the longer-lived, higher-value target
- Without rotation, a stolen refresh token is valid for its entire lifetime with no way to notice

### Sessions vs JWT — which one?

| | Server sessions | JWT |
|---|---|---|
| Where the truth lives | On the server (session store) | Inside the token itself |
| Instant revocation | Yes — delete the record | Hard — valid until it expires, unless you keep a denylist |
| Per-request database lookup | Yes | No, just a signature check |
| Works across independent services | Needs a shared session store | Yes, any service with the key can verify it |
| Size on the wire | Small (just an opaque ID) | Larger (the whole payload, base64-encoded) |
| Typical fit | Server-rendered app, one backend | SPA + API, mobile, multiple services |

Neither is strictly "more secure" — they move the same trust to different places. If you can't decide, default to sessions for a single server-rendered app, and JWTs (short-lived, with a refresh token) once more than one independent service needs to verify identity on its own.

## Letting other apps in

### OAuth 2.0

**In one line:** a standard that lets one app get limited access to your data on another service, without that app ever seeing your password there.

**How it works:** it's a valet key for a car — the valet can drive and park it, but the key doesn't open the trunk or the glovebox. OAuth defines four roles: the **resource owner** (you), the **client** (the app requesting access, e.g. a reading-tracker app that wants your Bookshop order history), the **authorization server** (issues tokens, e.g. Bookshop's login system), and the **resource server** (holds the actual data an API call reads, often the same service as the authorization server). The client never sees your password — it only ever receives a token scoped to what it asked for.

There are a few different **grant types** for different situations. **Authorization code + PKCE** is the standard flow today for anything with a user present — web apps, mobile apps, and single-page apps alike — where PKCE (a locally generated secret checked at the end of the flow) stops a stolen authorization code from being redeemed by anyone else. **Client credentials** is for machine-to-machine calls with no user involved at all — one backend service authenticating directly to another. **Device code** is for devices with no good way to type, like a smart TV: you get a short code on the TV screen and enter it on your phone or laptop instead. Two older grant types — **implicit** (tokens returned directly in the URL, no code exchange) and **password/ROPC** (the app collects your password directly and trades it for a token) — are both explicitly deprecated by the current OAuth 2.0 Security Best Current Practice (RFC 9700) and shouldn't be used in new code.

```mermaid
sequenceDiagram
    participant W as Warehouse system (client)
    participant Auth as Bookshop authorization server
    participant API as Bookshop inventory API

    Note over W,API: No human involved — service to service
    W->>Auth: POST /token, client_id + client_secret, grant_type=client_credentials
    Auth->>Auth: Verify the client_id and client_secret
    Auth-->>W: access_token, scope=update_inventory
    W->>API: PATCH /inventory/42, Authorization: Bearer <token>
    API->>API: Check the token, check scope includes update_inventory
    API-->>W: Stock updated
```

**Example:** an authorization code + PKCE redirect, sending the customer to log in and approve access:

```
https://accounts.bookshop-oauth.example.com/authorize?
  response_type=code&
  client_id=bookclub_app&
  redirect_uri=https%3A%2F%2Fbookclub.example.com%2Fcallback&
  scope=read_orders&
  state=xyz123&
  code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
  code_challenge_method=S256
```

**Good for:**
- Letting a third-party app act on your behalf at another service, with a limited, revocable scope
- Machine-to-machine access with client credentials, no human required

**Watch out for:**
- OAuth 2.0 on its own only proves access, not identity — it doesn't tell the client *who* the user is (that's OpenID Connect, next)
- Always use PKCE, even for confidential clients — it's cheap insurance the current BCP recommends broadly

### OpenID Connect

**In one line:** a thin identity layer built on top of OAuth 2.0 that adds "here's who this user actually is."

**How it works:** OAuth 2.0 hands out the valet key; OpenID Connect (OIDC) adds a signed ID badge that says whose car it is. Practically, OIDC adds one new thing to an OAuth flow: an **ID token**, which is itself a JWT containing the user's identity (their subject ID, email, name) and is meant for the client app to read and trust — not to send to an API. The regular **access token** from OAuth still does the job of calling APIs; the ID token's only job is telling the app who just logged in. This is the difference that trips people up: access token = "what I can do," ID token = "who I am."

```mermaid
sequenceDiagram
    participant C as Customer
    participant A as Bookshop app
    participant G as Google (OpenID provider)

    C->>A: Click "Sign in with Google"
    A->>G: Redirect with scope=openid profile email
    C->>G: Log in and approve
    G-->>A: Authorization code
    A->>G: Exchange the code for tokens
    G-->>A: id_token (who you are) + access_token (API access)
    A->>A: Verify the id_token signature, read sub / email
    A-->>C: Logged in as alwin@example.com
```

**Example:** a decoded ID token payload —

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484974386270000",
  "email": "alwin@example.com",
  "email_verified": true,
  "aud": "bookshop-client-id",
  "exp": 1719483600
}
```

**Good for:**
- "Sign in with Google / Microsoft / Apple" style buttons
- Any case where the client app itself needs to know who's logged in, not just what it can access

**Watch out for:**
- Never treat a plain OAuth access token as proof of identity — without OIDC, the token proves access, not who requested it
- Always verify the ID token's signature and `aud` claim; don't just trust whatever the browser hands back

### SAML and enterprise SSO

**In one line:** an older, XML-based standard companies use to let staff log into many internal apps with one identity-provider login.

**How it works:** it's a company badge that many office doors trust because they all trust the same security desk. The **identity provider (IdP)** — often Okta, Entra ID (Azure AD), or Google Workspace in a company setting — is that security desk; each internal tool is a **service provider (SP)** that trusts assertions signed by the IdP. An employee opens a tool, gets redirected to the IdP, logs in (often with MFA), and the IdP hands back a signed XML document called an **assertion** saying "this is definitely this person, and here are their groups." The service provider checks the signature against the IdP's public certificate and logs the employee in — no password is ever typed into the service provider itself. This is the older, heavier cousin of OpenID Connect: same basic idea (one login, many apps trust the answer), but XML-based, older, and still dominant in large enterprise software because so much of it was built before OIDC existed.

```mermaid
sequenceDiagram
    participant E as Employee
    participant SP as Bookshop admin tool (Service Provider)
    participant IdP as Company identity provider

    E->>SP: Open the admin tool
    SP-->>E: Not logged in — redirect to the IdP
    E->>IdP: Log in with company password + MFA
    IdP->>IdP: Build a signed XML assertion: this is E, these are their groups
    IdP-->>E: POST the assertion back to the browser
    E->>SP: Forward the signed assertion
    SP->>SP: Verify the XML signature against the IdP's certificate
    SP-->>E: Logged in, session started
```

**Example:** a trimmed SAML assertion (a real one is longer and includes the digital signature block):

```xml
<saml:Assertion>
  <saml:Subject>
    <saml:NameID>priya@bookshop.example.com</saml:NameID>
  </saml:Subject>
  <saml:AttributeStatement>
    <saml:Attribute Name="groups">
      <saml:AttributeValue>store-managers</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

**Good for:**
- Enterprise environments where the company already runs an IdP for every employee
- Centralizing offboarding — disable one account at the IdP and every connected app loses access at once

**Watch out for:**
- Heavier to implement than OIDC; most new consumer-facing apps should reach for OIDC instead
- The security of every connected app now depends entirely on the IdP's own security

### API keys (and why they're not user login)

**In one line:** a long secret string that identifies which application or service is calling an API — not which human is using it.

**How it works:** it's a supplier's loading-dock keycard. It proves "this is the bookshop's approved supplier system," not which specific staff member is walking through the door with it. An API key is typically a single long random string sent with every request, usually in an `Authorization` header. Unlike a session or a JWT, it generally isn't tied to one human, often doesn't expire on its own, and carries no proof of *who* is currently using it — only *that* whoever holds the string is treated as the app it was issued to. That makes it a poor substitute for a real login flow whenever a human user's identity actually matters.

```mermaid
sequenceDiagram
    participant Sup as Supplier system
    participant API as Bookshop inventory API

    Sup->>API: GET /v1/inventory, Authorization: Bearer bk_live_51H...
    API->>API: Look up which application this key belongs to
    API->>API: Check the key is active and not revoked
    API-->>Sup: Inventory data — no human identity involved
```

**Example:**

```bash
curl -H "Authorization: Bearer bk_live_51Hxyz9f2a7c1b" \
  https://api.bookshop.example.com/v1/inventory
```

**Good for:**
- Server-to-server integrations where no individual user is involved
- Simple, low-friction access to a public or partner API

**Watch out for:**
- Never use an API key as a stand-in for user login — it can't tell two employees of the same company apart
- Keys leak easily into logs, client-side code, and git history; scope them narrowly and rotate them

## Deciding what you're allowed to do

### RBAC

**In one line:** permissions are grouped into named roles (admin, editor, viewer), and access checks ask "does this role allow this action."

**How it works:** it mirrors job titles at the bookshop — a cashier role can process refunds up to a set limit, a store manager role can void any transaction, a warehouse role can update stock but not touch customer data. You assign users to roles, and roles to permissions, rather than handing permissions to individual people one by one. This is by far the most common access model because it's easy to reason about and easy to audit: "who can do X" is just "who has role Y."

```mermaid
flowchart LR
    U["User: Priya"] -->|assigned| Role["Role: Store manager"]
    Role -->|grants| P1[refund_any_order]
    Role -->|grants| P2[edit_book_listing]
    Req["Request: refund order #881"] --> Check{"Does Priya's role<br/>include this permission?"}
    Check -->|yes| Allow[Allowed]
    Check -->|no| Deny[Denied]
```

**Example:**

```js
if (user.roles.includes("store_manager")) {
  allow("refund_any_order");
}
```

**Good for:**
- Most business apps with a fairly stable, small set of job functions
- Easy auditing: list the roles, and you've listed the access

**Watch out for:**
- "Role explosion" when real access needs are more fine-grained than a handful of roles can express
- Roles that quietly accumulate permissions over time without review

### ABAC

**In one line:** permissions are decided by evaluating attributes of the user, the resource, and the context at request time, instead of a fixed role name.

**How it works:** it's a library card that lets you check out more books during off-peak hours, or a support rule that only lets staff access customers in their own region. A rule looks like "allow if `user.department == support` AND `order.region == user.region` AND it's currently business hours." Where RBAC asks "what role do you have," ABAC asks "given everything true about you, this resource, and right now, is this allowed" — more flexible, and more work to reason about.

```mermaid
flowchart LR
    Req["Request: edit order #881"] --> Eval{"Evaluate attributes"}
    Eval --> A1["user.department == support"]
    Eval --> A2["order.region == user.region"]
    Eval --> A3["time is business hours"]
    A1 --> Combine{"All conditions true?"}
    A2 --> Combine
    A3 --> Combine
    Combine -->|yes| Allow[Allowed]
    Combine -->|no| Deny[Denied]
```

**Example:**

```json
{
  "effect": "allow",
  "action": "edit_order",
  "condition": "user.department == 'support' && order.region == user.region"
}
```

**Good for:**
- Fine-grained rules that genuinely depend on context, not just job title
- Regulated industries needing rules like "only during business hours" or "only from this region"

**Watch out for:**
- Rules can become hard to trace — "why was this allowed" may require replaying several attributes at once
- Needs good tooling to test and audit, or contradictory rules go unnoticed

### ReBAC (Google Zanzibar style)

**In one line:** access depends on your relationship to the specific resource, not your job title — "can edit if you're in the doc's editors group."

**How it works:** think of a shared Google Doc. Whether you can edit it has nothing to do with your role at the company — it depends entirely on whether someone explicitly added you (or a group you belong to) as an editor of *that particular document*. Relationship-based access control (ReBAC) models permissions as a graph of relationships — "user is in group," "group is editor of document" — and answers an access check by walking that graph rather than looking up a role or evaluating a rule. Google's **Zanzibar** paper describes exactly this system at the scale of Google Docs, Drive, and YouTube, and it's the model behind open-source systems like OpenFGA.

```mermaid
flowchart LR
    Doc["Document: order-notes-881"] -->|has editor| Group["Group: support-team"]
    Group -->|has member| U1["User: Priya"]
    Req["Can Priya edit order-notes-881?"] --> Walk["Walk the relationship graph:<br/>Priya is in support-team,<br/>support-team is editor of the doc"]
    Walk --> Allow[Allowed]
```

**Example:** a Zanzibar-style relationship tuple and the check it answers:

```
document:order-notes-881#editor@group:support-team#member
check(user:priya, edit, document:order-notes-881) → allowed
```

**Good for:**
- Sharing and collaboration features — docs, folders, projects — where access follows "who was invited," not "what's their job"
- Systems where the same person needs different access to different individual resources of the same type

**Watch out for:**
- More infrastructure to build or adopt than RBAC — usually reached for once RBAC genuinely can't express the sharing model needed
- Graph checks can get expensive at scale without careful caching, since a single check may need to walk several hops

### Row-level security in the database

**In one line:** the database itself enforces who can see or change which rows, so even a buggy app query can't leak another customer's data.

**How it works:** instead of trusting every waiter to bring food only to the right table — trusting every single app query to remember to filter by customer — the kitchen itself refuses to hand over a dish that isn't addressed to that table. Postgres's row-level security (RLS) does exactly this at the database layer: you enable RLS on a table, then write a policy that Postgres applies automatically to every query, filtering out rows the current session isn't allowed to see, regardless of what the application code asked for.

```mermaid
flowchart TD
    App["App runs: SELECT * FROM orders"] --> DB[("Postgres")]
    DB --> Policy["Row-level security policy:<br/>customer_id = current user?"]
    Policy -->|"row belongs to this user"| Return[Row included in result]
    Policy -->|"row belongs to someone else"| Hide[Row silently excluded]
```

**Example:**

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY customer_sees_own_orders
  ON orders
  FOR SELECT
  USING (customer_id = current_setting('app.current_customer_id')::int);
```

**Good for:**
- A last line of defence against authorization bugs in application code — the database enforces the rule even if a query forgets to
- Multi-tenant apps where one mistake in a `WHERE` clause could otherwise leak data across tenants

**Watch out for:**
- Easy to forget a policy on a new table and silently expose everything (or, with a wrong policy, hide everything)
- Adds a small amount of query overhead and complexity to reason about compared to application-level filtering alone

### Principle of least privilege

**In one line:** give every user, service, and API key only the access it needs to do its job — nothing more — by default.

**How it works:** the bookshop's stockroom key opens the stockroom, not the safe, even for staff who'd probably never touch the safe anyway. The point isn't distrust of any specific person or service — it's that the smaller the set of things something can do, the smaller the damage when a password leaks, a token gets stolen, or a bug misfires. Practically this means defaulting to deny and granting narrowly (a support tool gets `read_orders`, not `*`), reviewing access periodically rather than only at onboarding, and revoking anything unused.

**Example:**

```
# Narrow (good): this key can only read inventory
Authorization: Bearer bk_readonly_inventory_9f2a...

# Broad (risky): this key can do everything, including refunds and account deletion
Authorization: Bearer bk_admin_all_scopes_x82m...
```

**Good for:**
- Every credential in the system — user roles, API keys, service accounts, OAuth scopes alike
- Limiting blast radius when (not if) some credential eventually leaks

**Watch out for:**
- "Just give it admin, it's easier" is exactly the shortcut that turns a small leak into a large one
- Unused broad grants tend to accumulate silently unless someone actively audits them

## Common attacks and defences

| Attack | What happens | Defence |
|---|---|---|
| Credential stuffing | Attacker tries email/password pairs leaked from other, unrelated breaches, betting on password reuse | Rate limit login attempts, check new passwords against known-breached lists, require MFA |
| Brute force | Attacker tries many passwords against one account | Rate limiting and progressive backoff on failed attempts, CAPTCHAs, slow password hashing (bcrypt/argon2) |
| Session hijacking | Attacker steals a valid session ID or cookie and reuses it as if they were you | HttpOnly + Secure cookies, rotate the session ID after login, keep sessions short-lived |
| CSRF (cross-site request forgery) | A malicious site tricks your logged-in browser into submitting a request you never intended | `SameSite=Lax` or `Strict` cookies, CSRF tokens on state-changing requests |
| Token theft via XSS | A malicious script running on the page reads a token from JS-accessible storage and sends it to the attacker | Never store tokens in `localStorage`; keep them in HttpOnly cookies; fix the underlying XSS (escape output, set a Content Security Policy) |
| Phishing | A fake login page tricks you into typing your real credentials into it | Phishing-resistant auth (passkeys/WebAuthn, bound to the real domain), MFA, user education |

## Which setup should I use?

```mermaid
flowchart TD
    Start{"What are you building?"}
    Start -->|"Server-rendered web app"| S1["Server sessions + cookies<br/>(HttpOnly, Secure, SameSite)"]
    Start -->|"SPA calling a separate API"| S2["Short-lived JWT access token<br/>+ refresh token rotation"]
    Start -->|"Mobile app"| S3["OAuth 2.0 authorization code + PKCE,<br/>tokens stored in Keychain / Keystore"]
    Start -->|"Internal tool, staff already<br/>have Google or Microsoft accounts"| S4["OpenID Connect or SAML SSO<br/>against the company identity provider"]
```

**Simple web app with server rendering.** One backend renders the HTML and serves the API from the same origin. Server sessions with an HttpOnly, Secure, `SameSite=Lax` cookie are the simplest correct answer — no token storage problem to solve, and logout is a single database delete.

**SPA + API.** A JavaScript frontend talks to a separate API, possibly on a different subdomain. Short-lived JWT access tokens plus rotating refresh tokens are the common fit — but store the tokens in an HttpOnly cookie where the frontend and API share a registrable domain, rather than in `localStorage`, to keep them out of reach of any XSS bug.

**Mobile app.** No cookies to lean on in the same way. OAuth 2.0's authorization code flow with PKCE is the standard pattern (even for a first-party app talking to its own backend), with tokens stored in the platform's secure storage — Keychain on iOS, Keystore on Android — not in plain shared preferences or a plist.

**Company internal tools with Google/Microsoft login.** Staff already have a company identity in Google Workspace or Microsoft Entra ID. Let that identity provider do the work: OpenID Connect for simpler modern tools, or SAML for enterprise software that only speaks it. Either way, offboarding one employee at the IdP cuts off every connected tool at once.

## Common mistakes

- **Storing JWTs in `localStorage`.** Any XSS bug on the page can read `localStorage` and steal the token; an HttpOnly cookie is invisible to JavaScript entirely.
- **Long-lived access tokens with no expiry.** A stolen token is valid for as long as it lives — keep access tokens short (minutes to an hour) and lean on refresh tokens for longevity.
- **Rolling your own password hashing or crypto.** Use a vetted library (bcrypt, argon2) — hand-written crypto is where subtle, catastrophic bugs hide.
- **Checking auth only in the frontend.** Hiding a button doesn't stop a direct API call with the network tab open — every check has to be enforced server-side too.
- **Not rate-limiting login or token endpoints.** Makes brute force and credential stuffing cheap and largely invisible until it's too late.
- **Still using OAuth's implicit or password (ROPC) grants.** Both are deprecated by the current OAuth 2.0 Security BCP (RFC 9700) — use authorization code + PKCE instead.
- **Forgetting to invalidate sessions or tokens after a password change or logout.** Old sessions should die immediately, not just wait to expire on their own.
- **Granting broad scopes or roles "to be safe."** The opposite of least privilege — grant exactly what's needed, and review what's already been granted.

## Go deeper

- Tools: [web-dev-resources → Auth](https://github.com/alwintwk/web-dev-resources#auth)
- Related: [big-tech-system-design → Rate limiting](https://github.com/alwintwk/big-tech-system-design/blob/main/concepts/rate-limiting.md) (login throttling is rate limiting applied to auth endpoints)
- Authoritative references: [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html), [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [oauth.net](https://oauth.net/2/), [RFC 6749 (OAuth 2.0)](https://www.rfc-editor.org/rfc/rfc6749), [RFC 7636 (PKCE)](https://www.rfc-editor.org/rfc/rfc7636), [RFC 9700 (OAuth 2.0 Security Best Current Practice)](https://www.rfc-editor.org/rfc/rfc9700), [webauthn.guide](https://webauthn.guide/), [passkeys.dev](https://passkeys.dev/), [MDN — Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
