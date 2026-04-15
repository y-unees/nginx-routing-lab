# Phase 3: Next.js Deployment & Nginx Reverse Proxy

> **Lab Series:** The Nginx Routing Lab  
> **◀ Phase 2:** SSL/TLS & Security Hardening  
> **Live Environment:** https://vibe.unishdhungana.com.np 

>> Note: This domain is used for demonstration and may be inactive post-lab.

---

## Welcome to Phase 3 🎉

In this phase of my DevOps lab series, I moved beyond serving static files and stepped into a more production-like setup. I deployed a **dynamic Next.js application** and configured **Nginx as a reverse proxy** to expose it securely.

This phase felt like a major leap because I started managing a real application lifecycle — not just files.

### What I Successfully Achieved

- Deployed a production-ready Next.js app
- Managed it using **PM2** for reliability
- Configured **Nginx reverse proxying**
- Served **multiple applications on a single server/IP**

---

## What's New in This Phase

| Component | Phase 2 | Phase 3 |
|---|---|---|
| Content type | Static HTML | Dynamic Next.js |
| Nginx role | File server + SSL | Reverse proxy |
| Process management | None | PM2 |
| Subdomain | lab | + vibe.unishdhungana.com.np |

---

## Architecture Overview

Here’s the mental model I built while working through this:

```
                          MY SERVER
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Browser ─── HTTPS :443 ──► Nginx                           │
│                               │                             │
│                               ▼                             │
│                           (Routing)                         │
│                               │                             │
│                               ▼                             │
│                           PM2 Manager                       │
│                               │                             │
│                               ▼                             │
│                    Next.js App (port 3000)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Insight

My Next.js app runs internally on **port 3000**, and it is **never exposed directly to the internet**.

- Nginx handles HTTPS
- PM2 keeps the app alive
- My app just focuses on serving content

---

## 1. Setting Up Node.js with NVM

Instead of installing Node via `apt`, I used **NVM** so I could manage versions cleanly.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm alias default node
```

---

## 2. Creating and Building the Next.js App

```bash
cd ~
npx create-next-app@latest vibe-next-app
cd vibe-next-app

npm run build
```

I explicitly avoided `npm run dev` since this is a production setup.

---

## 3. Managing the App with PM2

To keep my app alive beyond SSH sessions:

```bash
npm install -g pm2

pm2 start npm --name "vibe-next-app" -- start
pm2 save
pm2 startup
```

PM2 became my process supervisor — ensuring uptime even after reboots.

---

## 4. Configuring Nginx Reverse Proxy

```bash
sudo vim /etc/nginx/sites-available/vibe.unishdhungana.com.np
```

### Nginx Config

```nginx
server {
    listen 80;
    server_name vibe.unishdhungana.com.np;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name vibe.unishdhungana.com.np;

    ssl_certificate     /etc/letsencrypt/live/vibe.unishdhungana.com.np/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vibe.unishdhungana.com.np/privkey.pem;

    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then I enabled it:

```bash
sudo ln -s /etc/nginx/sites-available/vibe.unishdhungana.com.np \
           /etc/nginx/sites-enabled/

sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. SSL Certificate with Certbot

```bash
sudo certbot --nginx -d vibe.unishdhungana.com.np
sudo systemctl reload nginx
```

---

## 6. Multi-Site Hosting

At this point, my server was serving:

- Static site → lab
- Dynamic app → vibe.unishdhungana.com.np

Nginx routes requests based on `server_name`.

---

## 7. Verification Checklist

```bash
pm2 list
curl http://127.0.0.1:3000
sudo nginx -t
curl -I https://vibe.unishdhungana.com.np
```

---

## 8. Troubleshooting

### EADDRINUSE Error

I ran into this issue multiple times:

```
EADDRINUSE: address already in use :::3000
```

This usually happened due to **zombie processes**.

Instead of manually killing PIDs every time, I used:

```bash
fuser -k 3000/tcp
```

This instantly cleared the port.

Then I restarted cleanly:

```bash
pm2 restart vibe-next-app
```

---

## Lessons Learned

- PM2 does not prevent duplicate processes automatically
- Ports can remain locked by zombie Node processes
- `fuser -k` is much faster than manual PID hunting
- Always check `pm2 list` before restarting apps
- Clean process management is critical in production

---

## Key Takeaways

- I now understand reverse proxy architecture in practice
- Nginx + PM2 is a powerful combination
- Separation of concerns (SSL vs app logic) simplifies everything
- Production != development (never use dev server)

---

## Repository Structure

```
.
├── README.md
├── SSL_TLS/
│   └── README.md
├── REVERSE_PROXY/
│   └── README.md
├── configs/
│   ├── routing.conf
│   ├── routing-ssl.conf
│   └── vibe-proxy.conf
└── images/
```

---

## Phase 4 Preview 🚀

In the next phase, I’m planning to explore:

- Dockerizing the Next.js app
- Reverse proxying a completely different stack (maybe Node API or Flask)
- Introducing container orchestration concepts

---

This phase marked a big shift in my learning — I moved from configuring servers to actually **running applications in a production-like environment**.
