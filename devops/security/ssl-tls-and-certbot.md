---
tags:
  - devops
  - security
  - ssl
  - tls
  - encryption
last_reviewed: 2026-10-02
related_notes:
  - "[[nginx-reverse-proxy-and-spa-routing]]"
  - "[[self-hosted-vps-vs-paas]]"
---

# SSL/TLS Architecture, Public-Key Cryptography & Certbot Deep-Dive

> **Active Recall Self-Test:**
> 1. What are the three foundational vulnerabilities of plain unencrypted HTTP that TLS (HTTPS) resolves?
> 2. How does the TLS handshake combine **Asymmetric Encryption** (Public/Private Keys) with **Symmetric Encryption** (Session Keys) to balance security and performance?
> 3. How does the Let's Encrypt **ACME HTTP-01 Challenge** mathematically prove domain ownership to issue a signed certificate?
> 4. Why must Let's Encrypt certificates be renewed every 90 days, and how does a Certbot cron / systemd timer automate this?

---

## 1. The Core Problem
### The Three Foundational Flaws of the Public Internet

When an application runs over plain HTTP (`http://domain.com`), traffic traverses dozens of untrusted intermediate routers, public Wi-Fi access points, and Internet Service Providers (ISPs).

#### Flaw 1: The Eavesdropping Problem (Sniffing)
Any intermediate node running a packet sniffer (like Wireshark) can read passwords, session cookies, credit card numbers, and authorization headers in clear, unencrypted ASCII text.

#### Flaw 2: The Impersonation Problem (Man-in-the-Middle)
Anyone on a public network can perform DNS spoofing or ARP poisoning, routing traffic intended for your bank or app to an imposter server. Plain HTTP provides zero cryptographic verification that the server you connected to is genuinely who it claims to be.

#### Flaw 3: The Data Tampering Problem (Integrity)
Rogue Wi-Fi routers or ISPs can inject third-party ads or malicious scripts directly into the HTML response stream before it reaches the user's browser.

#### The Architectural Solution: Transport Layer Security (TLS)
1. **Confidentiality:** Military-grade encryption renders intercepted traffic unreadable garbage.
2. **Authentication:** Certificate Authorities (CAs like Let's Encrypt) digitally sign public keys to guarantee server identity.
3. **Integrity:** Message Authentication Codes (MACs) ensure data has not been altered in transit.

---

## 2. The Mental Model

### The Hybrid TLS Handshake: Asymmetric + Symmetric Encryption
Asymmetric encryption (RSA / Elliptic Curve) is computationally expensive. Symmetric encryption (AES-GCM) is blazing fast. The TLS handshake intelligently uses asymmetric encryption **only once** to negotiate a fast symmetric key:

```
[ Browser / Client ]                                   [ Server (Nginx) ]
        │                                                       │
        │ 1. ClientHello: "I support TLS 1.3, AES-256"          │
        ├──────────────────────────────────────────────────────►│
        │                                                       │
        │ 2. ServerHello + Signed Certificate (Public Key)      │
        │◄──────────────────────────────────────────────────────┤
        │                                                       │
   ─── Browser validates Certificate against Root CAs ──────────
        │ (Verifies domain signature; proves authenticity)      │
        │                                                       │
        │ 3. Generates a random, temporary SESSION KEY.         │
        │    Encrypts it using the Server's PUBLIC KEY.         │
        │    Sends encrypted session key across the wire.       │
        ├──────────────────────────────────────────────────────►│
        │                                                       │
   ─── Server decrypts message using its secret PRIVATE KEY ────
        │ (Now BOTH parties hold the identical SESSION KEY!)   │
        │                                                       │
        │ 4. Fast Two-Way Symmetric Encryption (AES-GCM)        │
        │◄═════════════════════════════════════════════════════►│
        │ All HTTP requests & responses encrypted with the      │
        │ shared Session Key. Zero overhead! Green padlock! 🔒 │
```

### The Certbot ACME HTTP-01 Challenge
How does Let's Encrypt know you actually own `domain.com` without human intervention?
```
1. You run Certbot on your VPS:
   certbot certonly --webroot -w /var/www/certbot -d domain.com
        │
        ▼
2. Certbot contacts Let's Encrypt ACME server:
   "I want a certificate for domain.com"
        │
        ▼
3. Let's Encrypt generates a cryptographic challenge:
   "Prove it! Put this secret string XYZ into http://domain.com/.well-known/acme-challenge/XYZ"
        │
        ▼
4. Certbot drops the token file into /var/www/certbot/.well-known/acme-challenge/
        │
        ▼
5. Let's Encrypt makes an HTTP request to http://domain.com/.well-known/acme-challenge/XYZ
   (If the token matches, domain ownership is mathematically proven!)
        │
        ▼
6. Let's Encrypt delivers signed fullchain.pem and privkey.pem to your server!
```

---

## 3. Production Code Breakdown

### A. Certbot Challenge & HTTPS Nginx Configuration (`/etc/nginx/sites-available/app.conf`)
```nginx
# =========================================================================
# 1. HTTP SERVER: ACME Challenge & Universal HTTPS Redirect
# =========================================================================
server {
    listen 80;
    server_name rhrony05.me www.rhrony05.me;

    # Allow Let's Encrypt ACME challenge verification over HTTP port 80
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect all other HTTP traffic permanently to HTTPS port 443
    location / {
        return 301 https://$host$request_uri;
    }
}

# =========================================================================
# 2. HTTPS SERVER: TLS Termination & Reverse Proxy
# =========================================================================
server {
    listen 443 ssl http2;
    server_name rhrony05.me www.rhrony05.me;

    # Paths to Let's Encrypt certificates managed by Certbot
    ssl_certificate /etc/letsencrypt/live/rhrony05.me/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rhrony05.me/privkey.pem;

    # Hardened TLS Protocols & Ciphers (Qualys SSL Labs A+ Grade)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # SSL Session Caching for reduced handshake latency
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # HTTP Strict Transport Security (HSTS) - Forces HTTPS for 1 year
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # Static frontend and reverse proxy locations
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### B. Automated Renewal Verification
```bash
# Test automated renewal without modifying live certificates
sudo certbot renew --dry-run

# Inspect certificate expiration dates
sudo certbot certificates
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The 90-Day Expiration Trap & Systemd Timers
- **The Trap:** Let's Encrypt certificates expire strictly after **90 days** (designed to limit the blast radius of compromised keys). If you forget to configure automated renewal, your site goes down with severe browser security warning screens (`NET::ERR_CERT_DATE_INVALID`).
- **The Fix:** Certbot installs a systemd timer (`certbot.timer`) or cron job that runs twice daily. Always verify its status:
  ```bash
  systemctl status certbot.timer
  ```

### ⚠️ Gotcha 2: The Port 80 Blocking Trap
- **The Trap:** Disabling Port 80 in your firewall (`ufw deny 80`) under the mistaken belief that *"we only use HTTPS Port 443 now"*.
- **The Breakdown:** Let's Encrypt's ACME HTTP-01 challenge connects **exclusively over Port 80** to verify domain ownership. If Port 80 is blocked, certificate renewal fails silently until the certificate expires.
- **The Rule:** Always leave Port 80 open on your host firewall. Use Nginx to handle the ACME challenge, then issue an HTTP 301 redirect to HTTPS for all user requests.

### ⚠️ Gotcha 3: The Private Key Leak (`privkey.pem`)
- **The Trap:** Accidentally committing `/etc/letsencrypt/` files to Git.
- **The Disaster:** Anyone who gains possession of `privkey.pem` can decrypt past recorded traffic and impersonate your domain seamlessly.
- **The Rule:** `privkey.pem` must have strict read permissions (`chmod 600`) restricted exclusively to the `root` or `nginx` user, and must NEVER leave the host machine.
