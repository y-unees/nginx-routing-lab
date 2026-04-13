# Phase 2: Nginx SSL/TLS & Security Hardening

> **Lab Series:** The Nginx Routing Lab
> **Phase 1 Foundation:** [Static Routing & Clean URIs](https://github.com/y-unees/nginx-routing-lab/blob/main/README.md)
> **Live Environment:** `https://lab.unishdhungana.com.np` *(demonstration domain; may be inactive post-lab)*

---

## Overview

Phase 2 transitions the Nginx infrastructure established in Phase 1 from unencrypted HTTP to production-grade HTTPS. This phase covers the full SSL/TLS provisioning lifecycle using Let's Encrypt and Certbot, dual-layer firewall configuration across Azure and the host OS, and the implementation of industry-standard HTTP security headers.

The routing logic built in Phase 1 (`try_files`, clean URIs, explicit file mapping) is fully preserved and migrated into the hardened HTTPS server block.

---

## Objectives

- Provision a trusted TLS certificate via the ACME protocol (Let's Encrypt)
- Configure both Azure NSG and Ubuntu UFW to support HTTPS traffic and automated certificate renewal
- Enforce HTTPS-only access via a permanent HTTP-to-HTTPS redirect
- Harden the server with a core set of HTTP security response headers
- Refactor the Nginx configuration into a clean, two-block architecture

---

## Architecture: Before & After

### Phase 1 — Single HTTP Block

```
Client ──── HTTP :80 ──── Nginx ──── /var/www/html
                         (try_files routing)
```

### Phase 2 — Dual-Block HTTPS Architecture

```
Client ──── HTTP :80  ──── Nginx [Block 1: Redirector] ──── 301 → HTTPS
       └─── HTTPS :443 ─── Nginx [Block 2: SSL Terminator] ─── /var/www/html
                                  (TLS, Security Headers, try_files routing)
```

---

## Table of Contents

1. [Infrastructure & Firewall Configuration](#1-infrastructure--firewall-configuration)
2. [SSL Certificate Provisioning (Certbot)](#2-ssl-certificate-provisioning-certbot)
3. [Automated Renewal Verification](#3-automated-renewal-verification)
4. [Nginx Configuration Refactor](#4-nginx-configuration-refactor)
5. [HTTP Security Headers](#5-http-security-headers)
6. [Verification Checklist](#6-verification-checklist)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Infrastructure & Firewall Configuration

HTTPS requires opening Port 443 at two distinct enforcement layers: the cloud-level network perimeter (Azure NSG) and the host-level firewall (UFW). Both layers must be configured — a misconfiguration at either level will block traffic regardless of the other.

Port 80 must remain open even after HTTPS is enabled. This is a hard operational requirement for two reasons:

1. **Let's Encrypt HTTP-01 Challenge:** The ACME protocol verifies domain ownership by making an outbound HTTP request to `http://your-domain.com/.well-known/acme-challenge/<token>`. If Port 80 is firewalled, certificate issuance and all future automated renewals will fail.
2. **HTTP-to-HTTPS Redirect:** The Port 80 server block serves a `301 Moved Permanently` response, redirecting all unencrypted clients to HTTPS. Closing Port 80 would silently break this for any client that navigates to `http://` directly.

### 1.1 Azure Network Security Group (NSG)

An inbound security rule was added to the NSG associated with the VM's network interface to permit both HTTP and HTTPS traffic from any source.

| Priority | Name          | Port | Protocol | Source | Action |
|----------|---------------|------|----------|--------|--------|
| 100      | Allow-HTTP    | 80   | TCP      | Any    | Allow  |
| 110      | Allow-HTTPS   | 443  | TCP      | Any    | Allow  |

### 1.2 Host-Based Firewall (UFW)

Ubuntu's UFW was configured using the `Nginx Full` application profile, which encapsulates rules for both Port 80 and Port 443 in a single command, avoiding partial configurations.

```bash
# Apply the Nginx Full profile (covers ports 80 and 443)
sudo ufw allow 'Nginx Full'

# Remove any pre-existing, now-redundant single-port rules
sudo ufw delete allow 'Nginx HTTP'

# Verify the active ruleset
sudo ufw status verbose
```

Expected output:

```
Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW IN    Anywhere
Nginx Full (v6)            ALLOW IN    Anywhere (v6)
```

---

## 2. SSL Certificate Provisioning (Certbot)

Certbot automates the full ACME certificate lifecycle: domain validation, certificate issuance, and Nginx configuration injection.

### 2.1 Installation

```bash
# Install Certbot and the Nginx plugin
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

The `python3-certbot-nginx` plugin is critical. It enables Certbot to read the existing Nginx configuration, automatically locate the correct `server_name` directive, inject the SSL certificate paths (`ssl_certificate`, `ssl_certificate_key`), and reload Nginx — all without requiring manual file editing.

### 2.2 Certificate Issuance

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

**What happens during this command:**

1. Certbot identifies the server block in Nginx matching `your-domain.com`.
2. It generates a temporary file at `/.well-known/acme-challenge/<token>` on the local filesystem.
3. The Let's Encrypt CA makes an outbound HTTP request to `http://your-domain.com/.well-known/acme-challenge/<token>` to verify domain control (HTTP-01 challenge).
4. Upon successful validation, the CA issues a certificate chain, which Certbot stores at a deterministic path:

```
/etc/letsencrypt/live/your-domain.com/fullchain.pem   # Certificate + chain
/etc/letsencrypt/live/your-domain.com/privkey.pem     # Private key
```

5. Certbot injects the SSL directives into the Nginx configuration and performs a graceful reload.

---

## 3. Automated Renewal Verification

Let's Encrypt certificates have a 90-day validity window. Certbot installs a `systemd` timer to handle renewals automatically, with no manual intervention required.

### 3.1 Verify the Timer is Active

```bash
# Check the status of the Certbot renewal timer
sudo systemctl status certbot.timer
```

Expected output:

```
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since ...
    Trigger: ...
   Triggers: ● certbot.service
```

The `Active: active (waiting)` and `enabled` states confirm the timer is armed and will survive reboots.

### 3.2 Verify the Renewal Hook

```bash
# List the post-renewal hooks directory
ls /etc/letsencrypt/renewal-hooks/deploy/
```

Certbot's Nginx plugin registers a deploy hook in this directory that reloads Nginx after each successful renewal, ensuring the new certificate is served without any service interruption.

### 3.3 Perform a Dry-Run Renewal Test

```bash
# Simulate a full renewal without modifying any files
sudo certbot renew --dry-run
```

A successful dry-run output confirms that the renewal pipeline — ACME challenge, certificate fetch, Nginx reload hook — is fully functional before the actual 60-day renewal trigger.

---

## 4. Nginx Configuration Refactor

The original single-block HTTP configuration from Phase 1 was refactored into two distinct, purpose-separated server blocks.

### 4.1 Block 1 — HTTP Redirector (Port 80)

This block's sole responsibility is to issue a `301 Moved Permanently` redirect to the HTTPS equivalent of every incoming request. It contains no `root`, no `try_files`, and no static file serving logic — it is strictly a redirector.

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    # Redirect all HTTP traffic to HTTPS with a 301 (permanent) redirect.
    # The $host variable preserves the original hostname.
    # The $request_uri variable preserves the full path and query string.
    return 301 https://$host$request_uri;
}
```

**Design rationale for `return 301` over `rewrite`:** The `return` directive is evaluated at the server block level before any location matching occurs, making it more efficient. It also avoids the regex overhead of `rewrite` for what is a straightforward, unconditional redirect.

### 4.2 Block 2 — HTTPS SSL Terminator (Port 443)

This block handles TLS termination, enforces the security header policy, and contains the preserved `try_files` routing logic from Phase 1.

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com www.your-domain.com;

    # --- SSL/TLS Certificate Configuration ---
    # Paths are managed and injected by Certbot.
    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # --- HTTP Security Headers ---
    # Enforce HTTPS for 6 months; apply to all subdomains.
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;
    # Prevent MIME-type sniffing by the browser.
    add_header X-Content-Type-Options "nosniff" always;
    # Disallow this site from being embedded in any iframe.
    add_header X-Frame-Options "DENY" always;

    # --- Static File Serving (preserved from Phase 1) ---
    root  /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Directory-based routing (Phase 1)
    location /abc/ {
        try_files $uri $uri/ =404;
    }

    # Explicit file mapping (Phase 1)
    location /xyz/ {
        try_files /xyz.html =404;
    }

    location /pqr/ {
        try_files /random_text.html =404;
    }
}
```

### 4.3 Apply and Reload

```bash
# Validate configuration syntax before applying
sudo nginx -t

# Gracefully reload Nginx to apply changes without downtime
sudo systemctl reload nginx
```

---

## 5. HTTP Security Headers

Each security header serves a distinct defense function at the browser enforcement layer. These are transport-level and rendering-level controls that no amount of application-layer hardening can substitute.

### `Strict-Transport-Security` (HSTS)

```nginx
add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;
```

| Parameter | Value | Meaning |
|---|---|---|
| `max-age` | `15768000` | Browser caches the HTTPS-only policy for 6 months (in seconds) |
| `includeSubDomains` | — | Policy applies to all subdomains of the origin |

**Mechanism:** Once a browser receives this header, it will refuse to make any HTTP connection to this domain for the duration of `max-age`, even if the user explicitly types `http://`. This eliminates the window of vulnerability present on a user's *first* visit before the `301` redirect fires, which is otherwise susceptible to SSL-stripping attacks.

**Important:** HSTS should only be deployed after confirming that HTTPS is fully functional. Serving this header on a misconfigured HTTPS setup can lock clients out of the site for the entire `max-age` duration.

### `X-Content-Type-Options`

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Instructs browsers to strictly honor the `Content-Type` header returned by the server and not attempt to infer the MIME type from the content body. Without this header, a browser might execute a JavaScript payload disguised as a `.txt` file, a class of attack known as MIME-sniffing or content-type confusion.

### `X-Frame-Options`

```nginx
add_header X-Frame-Options "DENY" always;
```

Instructs the browser to refuse to render this page inside any `<iframe>`, `<frame>`, or `<object>` element, regardless of origin. This prevents clickjacking attacks, where an attacker overlays a transparent iframe over a decoy page to trick a user into clicking on elements of the target site without their knowledge.

`DENY` is stricter than `SAMEORIGIN` and is appropriate for a site that has no legitimate reason to be embedded in an iframe.

> **Note on the `always` flag:** By default, Nginx only adds `add_header` directives to `2xx` and `3xx` responses. The `always` parameter ensures headers are also appended to error responses (`4xx`, `5xx`), which is the correct behavior for security-enforcing headers.

---

## 6. Verification Checklist

| Check | Command / Method | Expected Result |
|---|---|---|
| **Nginx syntax** | `sudo nginx -t` | `syntax is ok` and `test is successful` |
| **UFW ruleset** | `sudo ufw status verbose` | `Nginx Full` listed as `ALLOW IN` |
| **Certbot timer** | `sudo systemctl status certbot.timer` | `Active: active (waiting)` and `enabled` |
| **Dry-run renewal** | `sudo certbot renew --dry-run` | `Congratulations, all simulated renewals succeeded` |
| **HTTP redirect** | `curl -I http://your-domain.com` | `HTTP/1.1 301 Moved Permanently` with `Location: https://...` |
| **HTTPS response** | `curl -I https://your-domain.com` | `HTTP/2 200` |
| **HSTS header present** | `curl -sI https://your-domain.com \| grep -i strict` | `strict-transport-security: max-age=15768000; includeSubDomains` |
| **MIME header present** | `curl -sI https://your-domain.com \| grep -i x-content` | `x-content-type-options: nosniff` |
| **Framing header present** | `curl -sI https://your-domain.com \| grep -i x-frame` | `x-frame-options: DENY` |
| **Certificate validity** | `sudo certbot certificates` | Certificate listed with valid expiry and correct domain names |
| **SSL Labs grade** | [ssllabs.com/ssltest](https://www.ssllabs.com/ssltest/) | Grade **A** or higher |

---

## 7. Key Takeaways

- **Never close Port 80 after enabling HTTPS.** It serves two non-negotiable functions: Let's Encrypt HTTP-01 challenge verification and the permanent HTTP-to-HTTPS redirect entry point.
- **Certbot is a full lifecycle manager, not just an installer.** The `certbot.timer` systemd unit handles zero-touch 90-day renewals with Nginx reload hooks, requiring no manual intervention post-setup.
- **Separate server blocks enforce a clean separation of concerns.** The Port 80 block is a pure redirector; the Port 443 block handles all application logic. Mixing concerns in a single block (e.g., using `if ($scheme = http)`) is fragile and violates Nginx best practices.
- **The `always` flag on `add_header` is not optional for security headers.** Without it, headers are silently omitted from error responses, leaving gaps in coverage.
- **HSTS is a client-side enforcement mechanism.** Its protection is cumulative — it becomes stronger with each visit as browsers cache the policy. Deploy only after HTTPS is fully validated to avoid locking clients out.

---

## Repository Structure

```
.
├── README.md                  # Phase 1 documentation
├── README-phase2.md           # This document
├── configs/
│   ├── nginx.conf             # Global Nginx configuration
│   └── routing.conf           # Phase 1: Single HTTP server block
│   └── routing-ssl.conf       # Phase 2: Dual-block HTTPS configuration
├── web_root(/var/www/html)/
│   ├── index.html
│   ├── abc/  (index.html, test.html)
│   ├── xyz/  (xyz.html)
│   └── pqr/  (random_text.html)
└── images/
    └── ...
```

---

## Related Resources

- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/)
- [Certbot Documentation](https://certbot.eff.org/docs/)
- [MDN — Strict-Transport-Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)
- [MDN — X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options)
- [MDN — X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)
- [Qualys SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)

---

*This lab is part of an ongoing DevOps infrastructure series focused on production-grade Nginx configuration, security hardening, and automated operational workflows.*