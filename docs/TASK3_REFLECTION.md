# Task 3 Reflection

>  Publishing the Service via a Domain (Written Description)

## Objective

Publish the AI Learning Assistant — already running on an Aliyun ECS instance with the container port `14242` exposed directly — as a secure HTTPS service behind a controlled domain name, with an access-control layer that is independent of the application's built-in BNBU login.

Architecture chain:

```
AI agent source code → Dockerfile → docker build → local image → local container test
→ Aliyun ACR → ECS pulls image → Docker container → Nginx + Certbot → public IP / domain → Internet user
```
## Executive Processes
### Step 1 — Register a domain and configure DNS

1. Register, or use, a domain or subdomain you control (e.g. `ai.example.com`).
2. Look up the ECS instance's public IPv4 address from the Alibaba Cloud console.
3. Create a DNS `A` record pointing the domain to that public IPv4 address.
4. Confirm the record resolves correctly with `dig +short ai.example.com` or `nslookup`,
   and wait for the DNS TTL to propagate.

### Step 2 — Adjust the security group for ports 80/443

- Add inbound rules to the Alibaba Cloud security group allowing TCP `80` (HTTP) and`443` (HTTPS).
- After verifying the reverse proxy works, **remove** the public inbound rule for port `14242`, so the application is no longer exposed directly on that port.

### Step 3 — Install and configure Nginx as a reverse proxy

```bash
sudo apt update
sudo apt install nginx
sudo nano /etc/nginx/sites-available/ai-agent
```

Write the following server block (replace `ai.example.com` with the real domain):

```nginx
server {
    listen 80;
    server_name ai.example.com;

    location / {
        proxy_pass http://127.0.0.1:14242;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 360s;
        proxy_send_timeout 360s;
    }
}
```

Enable the site, validate, and reload:

```bash
sudo ln -s /etc/nginx/sites-available/ai-agent /etc/nginx/sites-enabled/ai-agent
sudo nginx -t
sudo systemctl reload nginx
```

Then confirm the domain and reverse proxy work over HTTP at `http://ai.example.com/ui/`.

### Step 4 — Enable HTTPS with Certbot

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d ai.example.com
```

Follow Certbot's prompts to issue and install the Let's Encrypt certificate, then confirm the application loads over HTTPS at `https://ai.example.com/ui/`.

### Step 5 — Add an access-control layer at the reverse proxy

The BNBU login is not production-grade authentication, so a public deployment requires an
additional, independent access-control layer. Two options:

1. **HTTP Basic Auth (simplest)**:

```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd ai-admin
```

Append to the HTTPS server block:

```nginx
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

Then `sudo nginx -t && sudo systemctl reload nginx`.

2. **Identity-aware access proxy** (e.g. Cloudflare Access or OAuth2 Proxy): delegate
   entry authentication to a professional identity gateway — better suited to production.

Either way, this layer must be independent of the application's BNBU login.

### Step 6 — Acceptance screenshot

- Open `https://ai.example.com/ui/`; the access-control check (Basic Auth prompt) appears first, then the application loads after valid credentials are entered.
- The screenshot must show the browser's address bar, proving the HTTPS domain (not a raw IP) is in use.
- Confirm the certificate is valid (the lock icon in the address bar).

