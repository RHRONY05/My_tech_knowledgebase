---
tags:
  - devops
  - nginx
  - web-serving
  - architecture
last_reviewed: 2026-10-02
related_notes:
  - "[[docker-compose-production-patterns]]"
  - "[[ssl-tls-and-certbot]]"
  - "[[self-hosted-vps-vs-paas]]"
---

# Nginx Production Web Architecture: Reverse Proxying & SPA Routing

> **Active Recall Self-Test:**
> 1. Why does refreshing the browser on a React client-side route like `/movies/seat-selection` return a 404 Not Found error without Nginx's `try_files` directive?
> 2. How does an Nginx **Reverse Proxy** eliminate browser Cross-Origin Resource Sharing (CORS) issues between frontend and backend?
> 3. Why can static production assets (`dist/assets/*.js`) be cached with a 1-year `Cache-Control: max-age=31536000, immutable` header, while `index.html` must never be cached?
> 4. What critical HTTP headers (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) must Nginx pass when forwarding `/api/` traffic to Express?

---

## 1. The Core Problem
### The Frontend Reality & The SPA 404 Bug

When you write React applications using Vite or Webpack, you develop with dynamic components, JSX, and hot-module reloading.

#### Problem A: The Browser's Reality
Web browsers understand only three things: **plain HTML, plain CSS, and plain JavaScript**. Running `npm run build` discards Node.js and compiles your application into flat, static files inside a `dist/` directory:
- `index.html` (The single entry point HTML file)
- `assets/index-D7h2k9.js` (Hashed JavaScript bundle)
- `assets/index-B1x8q2.css` (Hashed stylesheet bundle)

#### Problem B: The Single Page Application (SPA) 404 Bug
In a traditional multi-page website, visiting `/about` maps to a physical file at `/var/www/html/about.html` on the server disk.
In a React SPA:
- Only **ONE physical HTML file** exists on the server: `index.html`.
- When a user navigates to `/movies/seat-selection`, React Router intercepts the click and updates the URL via the HTML5 History API without requesting a new page from the server.
- **The Catastrophe:** The user loves the movie page and presses **F5 (Refresh)**. The browser makes a brand-new GET request to `https://domain.com/movies/seat-selection`.
- Nginx searches its physical disk for a folder named `/movies/seat-selection`. Finding nothing, **Nginx returns HTTP 404 Not Found!**

#### Problem C: The CORS Nightmare
If the frontend runs on `http://localhost:5173` and the backend runs on `http://localhost:5000`, the browser treats them as distinct origins, triggering CORS preflight `OPTIONS` requests that fail if server headers are misconfigured.

#### The Architectural Solution: Nginx Edge Routing
1. **The `try_files` SPA Fallback:** Instruct Nginx to search for the physical file on disk; if it does not exist, silently serve `index.html` with status 200, allowing React Router to mount and render the requested view.
2. **Reverse Proxying `/api/`:** Colocate the frontend and backend behind a single domain (`domain.com`). Nginx serves static files on `/` and forwards `/api/` to the backend container, completely eliminating CORS.

---

## 2. The Mental Model

### The Nginx Traffic Ingress Controller
```
[ Incoming Browser Request ]
             │
             ├── Requests Static File: GET /assets/index-C87a.js
             │   └── Found on Disk ──► Serves directly off SSD (Cache: 1 Year)
             │
             ├── Requests SPA Route: GET /movies/seat-selection
             │   └── File NOT found on Disk!
             │   └── try_files $uri $uri/ /index.html;
             │       └── Serves dist/index.html (Cache: no-cache)
             │       └── Browser boots React Router ──► Renders SeatSelection view!
             │
             └── Requests Data: POST /api/bookings/initiate
                 └── Matches location /api/
                 └── Reverse Proxies to internal container: http://backend:5000
                 └── Strips CORS, preserves Client IP via X-Forwarded-For
```

---

## 3. Production Code Breakdown

### Complete Production Nginx Configuration (`frontend/nginx.conf`)
```nginx
# =========================================================================
# PRODUCTION NGINX CONFIGURATION FOR REACT SPA + REVERSE PROXY
# =========================================================================

server {
    listen 80;
    server_name localhost;

    # Root directory pointing to compiled Vite build output
    root /usr/share/nginx/html;
    index index.html;

    # Enable Gzip compression to minimize wire payload
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;

    # =====================================================================
    # 1. SPA CLIENT-SIDE ROUTING FALLBACK
    # =====================================================================
    location / {
        # Check if the requested URI is a physical file ($uri),
        # then check if it's a directory ($uri/).
        # If neither exists, FALL BACK TO /index.html!
        try_files $uri $uri/ /index.html;
    }

    # =====================================================================
    # 2. STATIC ASSET CACHING STRATEGY
    # =====================================================================
    # Vite bundles include content hashes (e.g. index-D7h2k9.js).
    # If code changes, the hash changes! Therefore, hashed assets can be
    # cached forever in the browser.
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
        access_log off;
    }

    # ⚠️ NEVER cache index.html!
    # The browser must always check the server to discover new hashed JS bundles!
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0";
    }

    # =====================================================================
    # 3. BACKEND API REVERSE PROXY
    # =====================================================================
    location /api/ {
        # Forward traffic to the Docker service name 'backend' on port 5000
        proxy_pass http://backend:5000;

        # Standard HTTP proxy headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;

        # ⚠️ CRITICAL: Pass original client IP to Express backend
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout configurations to prevent hanging socket leaks
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Security: Deny access to hidden dotfiles (.git, .env)
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

---

## 4. Production Gotchas & Best Practices

### ⚠️ Gotcha 1: The Trailing Slash Proxy Pass Trap
- **The Trap:** Writing `proxy_pass http://backend:5000/;` (with a trailing slash) versus `proxy_pass http://backend:5000;` (without a trailing slash).
- **The Bug:**
  - With trailing slash (`http://backend:5000/`): Nginx strips the matching `/api/` prefix. A client request to `/api/movies` arrives at Express as `GET /movies`.
  - Without trailing slash (`http://backend:5000`): Nginx preserves the full path. A client request to `/api/movies` arrives at Express as `GET /api/movies`.
- **The Fix:** If your Express routes are defined as `app.use('/api/movies', router)`, **omit the trailing slash** on `proxy_pass`!

### ⚠️ Gotcha 2: The Stale `index.html` Cache Trap
- **The Trap:** Allowing browsers to cache `index.html` for 1 hour or 1 day.
- **The Disaster:** You push a critical bug fix to production. Vite generates a new bundle `index-NEW.js`. However, existing users have `index.html` cached in their browser storage. Their browser requests the old `index-OLD.js`, which no longer exists on the server, resulting in a blank white screen of death (`Loading chunk failed`) for all active users.
- **The Rule:** **NEVER cache `index.html`**. Enforce `Cache-Control: "no-store, no-cache"` on `index.html`, and cache only the hashed files inside `/assets/`.

### ⚠️ Gotcha 3: Missing `trust proxy` in Express
- **The Trap:** When Express sits behind an Nginx reverse proxy, `req.ip` returns `127.0.0.1` or the Nginx container's internal IP (`172.20.0.2`) instead of the user's real public IP address.
- **The Bug:** Rate limiters and security audit logs block Nginx itself instead of the malicious attacker.
- **The Fix:** Enable proxy trust in your Express `app.js`:
  ```javascript
  app.set('trust proxy', 1);
  ```
