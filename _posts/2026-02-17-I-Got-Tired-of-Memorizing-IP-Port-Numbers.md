---
layout: post
title: I Got Tired of Memorizing IP:Port Numbers
published: true
tags: homelab SSL nginx cloudflare letsencrypt dns docker pihole proxmox
excerpt_separator: <!--more-->
---

## The Problem

Every home lab starts the same way. You spin up a service. It gets an IP and a port. You bookmark it. Then you spin up another. Then another. Fast forward a few months and you're looking at this mess:

- `http://192.168.1.10:81` - Nginx Proxy Manager
- `http://192.168.1.5:8006` - Proxmox
- `http://192.168.1.15:9000` - Portainer
- `http://192.168.1.20:7889` - GoAccess

I used to pride myself on <a href="https://blog.mitchscobell.com/Refactoring-my-homelab-with-Github-Copilot/" target="_blank" rel="noopener noreferrer">memorizing IP addresses</a> (and honestly, I still do). Port numbers, though? That's where I draw the line!

<!--more-->

I even ran <a href="https://homarr.dev/" target="_blank" rel="noopener noreferrer">Homarr</a> for a while but the novelty wore off fast.

Like, is Portainer on :9000 or :9001? Was Proxmox :8006 or :8060? And good luck trying to explain to anyone else how to access your services: "Just type `http://192.168.1.27:32400`... no wait, that's the old IP... actually is it port 32400 or 8324? Hang on let me check..."

Every bookmark is a jumble of numbers that look like phone numbers from the 90s. Browsers throw security warnings constantly. On top of that, some modern web apps straight up refuse to work properly without SSL.

It's 2026. We have self-driving cars and AI that can write code. Why am I still memorizing IP:port combinations like it's 1999?

## The Solution

What if instead, you could just use normal URLs like a civilized human being?

- `https://proxy.mscob.com` ✅
- `https://proxmox.mscob.com` ✅
- `https://goaccess.mscob.com` ✅

**Plus:**

- Actual valid SSL certificates (green lock, no scary browser warnings)
- URLs you can actually remember
- Everything STILL completely internal (zero exposure to the internet)
- Auto-renewing certificates (set it and forget it)
- Costs literally $12/year (just the domain)

That's exactly what I built. Honestly? It was way easier than I expected. Let me show you how.

---

## Architecture Overview

Here's how all the pieces fit together:

```
┌──────────────────────────────────────────────────────┐
│  Internet (Cloudflare DNS - Public DNS only)         │
│  - No traffic ever reaches your network              │
│  - Only used for DNS challenge verification          │
└──────────────────────────────────────────────────────┘
                          ↓
         DNS queries only (no data traffic)
                          ↓
┌──────────────────────────────────────────────────────┐
│  Home Network (192.168.1.0/24)                       │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Router/Pi-hole (192.168.1.1)                  │  │
│  │  - Local DNS server                            │  │
│  │  - Maps *.mscob.com → 192.168.1.10             │  │
│  │  - Firewall (blocks external access)           │  │
│  └────────────────────────────────────────────────┘  │
│                          ↓                           │
│         proxmox.mscob.com → 192.168.1.10             │
│                          ↓                           │
│  ┌────────────────────────────────────────────────┐  │
│  │  Nginx Proxy Manager (192.168.1.10)            │  │
│  │  - SSL termination (*.mscob.com cert)          │  │
│  │  - Reverse proxy routing                       │  │
│  │  - Auto cert renewal via DNS challenge         │  │
│  └────────────────────────────────────────────────┘  │
│                          ↓                           │
│         Routes to actual service                     │
│                          ↓                           │
│  ┌────────────────────────────────────────────────┐  │
│  │  Proxmox (192.168.1.5:8006)                    │  │
│  │  Portainer (192.168.1.15:9000)                 │  │
│  │  GoAccess (192.168.1.20:7889)                  │  │
│  │  ... (all other services)                      │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

**Key Points:**

1. **Cloudflare** - Free DNS hosting. No traffic actually goes through them, they just host the DNS records.
2. **Router/Pi-hole** - Your local DNS server that lies to your devices and says "yeah, goaccess.mscob.com is totally at 192.168.1.10 trust me bro" (I use a UDM Pro, but any router with DNS override works, or throw up a <a href="https://pi-hole.net/" target="_blank" rel="noopener noreferrer">Pi-hole</a>)
3. **Nginx Proxy Manager** - The SSL terminator and traffic cop (I'm calling it "Nginx Proxy Manager" throughout because as a Node.js dev, "NPM" makes me think of Node Package Manager and it messes with my head)
4. **Services** - Your actual apps, now with fancy domain names

---

## Why This Works (The Magic of DNS Challenge)

So here's the thing: normally, getting an SSL certificate from Let's Encrypt requires them to connect to your server on port 80 or 443 to verify you own the domain. Which means... exposing your stuff to the internet. Hard pass.

**DNS Challenge is different though, and it's basically magic:**

1. You request a cert for `*.mscob.com`
2. Let's Encrypt says: "Prove you own this domain by creating this specific DNS TXT record"
3. Nginx Proxy Manager automatically creates the TXT record via Cloudflare's API:
   ```
   _acme-challenge.mscob.com  TXT  "random-validation-token"
   ```
4. Let's Encrypt queries public DNS, sees the TXT record, and issues the certificate
5. Nginx Proxy Manager automatically removes the TXT record
6. **Your services never touched the internet**

The certificate is valid, trusted by all browsers, and renews automatically every 60 days using the same DNS challenge process.

---

## Prerequisites

**You'll need:**

- **A real domain name** - I use `mscob.com` throughout this guide as an example (runs about $12/year) - just swap in your own domain wherever you see mine
- **Cloudflare account** - Free tier is perfect, don't need to pay for anything
- **Router with DNS override** - I use a UniFi Dream Machine Pro because I'm extra like that, but literally any router **that can do custom DNS** works (DD-WRT, pfSense, OPNsense, or a Pi-hole and point your router to that for DNS)
- **A server to run Nginx Proxy Manager** - I'm using a Proxmox VM, but honestly anything that can run Docker works
- Basic Linux/Docker knowledge (can copy-paste terminal commands without panicking)

---

## Step-by-Step Setup

### Part 1: Cloudflare DNS Setup

**1. Create Cloudflare Account**

- Sign up at https://dash.cloudflare.com (free)
- Add your domain (e.g., `mscob.com`)
- Update your domain registrar's nameservers to Cloudflare's
- Wait 24 hours for DNS propagation

**2. Create API Token**

Cloudflare Dashboard → My Profile → API Tokens → Create Token

- Template: **Edit zone DNS**
- Permissions: Zone → DNS → Edit
- Zone Resources: Include → Specific zone → `mscob.com`
- Copy the token (you'll only see it once!)

**3. Add DNS Records**

Add A records pointing to any IP (it truly doesn't matter since we'll override locally, the record just needs to exist for the challenge):

```
Type: A
Name: *.mscob.com
Content: 127.0.0.1 (or your public IP, or 192.168.0.1, anything works)
Proxy: OFF (this is critical!)
```

**Note:** These records will never be used for actual traffic since your local DNS will override them. They're just placeholders for the DNS challenge to work.

---

### Part 2: Nginx Proxy Manager Installation

**Option 1: Proxmox LXC (The Easy Way™)**

If you're running Proxmox, use the community helper scripts. These things are legitimately magic:

1. Open Proxmox shell
2. Copy-paste this bad boy:
   ```bash
   bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/nginxproxymanager.sh)"
   ```
3. Hit enter, follow the prompts (defaults are usually fine)
4. Wait like 5 minutes and boom, you've got a fully configured LXC container running Nginx Proxy Manager

**More info:** https://community-scripts.github.io/ProxmoxVE/scripts?id=nginxproxymanager

_Pro tip: I run these in advanced mode because I'm picky about which storage pool gets used and I want my SSH keys injected. If you're just getting started, standard mode is totally fine._

**Option 2: Manual Docker Installation**

For non-Proxmox setups, use Docker Compose:

```yaml
version: "3.8"
services:
  nginx-proxy-manager:
    image: "jc21/nginx-proxy-manager:latest"
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80" # HTTP
      - "81:81" # Admin UI
      - "443:443" # HTTPS
    volumes:
      - /data/nginx-proxy-manager/data:/data
      - /data/nginx-proxy-manager/letsencrypt:/etc/letsencrypt
```

```bash
docker-compose up -d
```

(Note: In a fully internal setup like this, you can omit exposing ports 80 and 443 if you want to ensure no direct external access attempts succeed. Local DNS overrides handle everything anyway.)

**Access the Admin UI:**

```bash
# Access admin UI
http://<your-proxy-manager-ip>:81

# Default credentials (change immediately!)
Email: admin@example.com
Password: changeme
```

---

### Part 3: Wildcard SSL Certificate via DNS Challenge

**In Nginx Proxy Manager UI:**

1. Navigate to **SSL Certificates** → **Add SSL Certificate**
2. Choose **Let's Encrypt**
3. Configure:
   ```
   Domain Names: *.mscob.com mscob.com
   Email: your-email@example.com
   Use a DNS Challenge: ✓
   DNS Provider: Cloudflare
   Credentials:
     dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
   ```
4. Agree to Terms ✓
5. Click **Save**

Within 1-2 minutes, you should have a valid wildcard certificate for `*.mscob.com`. This single cert will work for all your subdomains.

---

### Part 4: Local DNS Override

**Option 1: UniFi Dream Machine Pro / Network Application**

Settings → Networks → Default → DHCP & DNS → Custom DNS Entries

**Option 2: Pi-hole**

Local DNS → DNS Records

**Option 3: Other Routers**

Look for "DNS Override", "Custom DNS", "Static DNS", or "Local DNS" in your router's settings. Most modern routers support this feature.

Add entries for each service (all pointing to your Nginx Proxy Manager server):

```
proxy.mscob.com          → 192.168.1.10
proxmox.mscob.com        → 192.168.1.10
goaccess.mscob.com       → 192.168.1.10
octoprint.mscob.com      → 192.168.1.10
portainer.mscob.com      → 192.168.1.10
```

**What this does:** When any device on your network queries `proxmox.mscob.com`, your local DNS server returns your Nginx Proxy Manager server's IP instead of querying Cloudflare. Traffic never leaves your network.

**Test it:**

```bash
nslookup proxmox.mscob.com
# Should return: <your Nginx Proxy Manager server IP>
```

---

### Part 5: Add Proxy Hosts in Nginx Proxy Manager

For each service, create a proxy host:

**Example: Proxmox**

Nginx Proxy Manager UI → Hosts → Proxy Hosts → Add Proxy Host

**Details Tab:**

```
Domain Names: proxmox.mscob.com
Scheme: https
Forward Hostname/IP: <proxmox-server-ip>
Forward Port: 8006
Cache Assets: ✓
Block Common Exploits: ✓
Websockets Support: ✓
```

**SSL Tab:**

```
SSL Certificate: *.mscob.com (from dropdown)
Force SSL: ✓
HTTP/2 Support: ✓
HSTS Enabled: ✓
```

**Save**

Repeat for all services. Here's an example list:

| Service       | Port | URL                 |
| ------------- | ---- | ------------------- |
| Proxmox       | 8006 | proxmox.mscob.com   |
| Portainer     | 9000 | portainer.mscob.com |
| GoAccess      | 7889 | goaccess.mscob.com  |
| OctoPrint     | 5000 | octoprint.mscob.com |
| Tautulli      | 8181 | tautulli.mscob.com  |
| Uptime Kuma   | 3001 | uptime.mscob.com    |
| Manyfold      | 3214 | manyfold.mscob.com  |
| Proxy Manager | 81   | proxy.mscob.com     |

---

## Real-World Example: GoAccess Analytics (The Hard Part)

_Spoiler: I spent WAY too many hours debugging WebSockets..._

**CRITICAL:** Replace all instances of `goaccess.yourdomain.com` in the configuration below with your actual subdomain (e.g., `goaccess.mscob.com`). If you don't, WebSockets will break spectacularly.

**GoAccess** was hands down the most frustrating piece of this entire setup. It's a real-time web analytics tool that uses WebSockets for live updates. Which sounds simple until you try to proxy WebSockets through multiple layers of nginx with SSL termination.

Then it becomes a very special kind of hell.

**The Challenge:**

GoAccess runs a WebSocket server for real-time updates. This meant:

1. Setting up a custom Docker container with embedded nginx config
2. Properly proxying WebSocket connections through Nginx Proxy Manager
3. Getting Origin headers and authentication working correctly
4. Making the WebSocket URLs match the SSL domain (wss://)

**Before:** `http://<ip-address>:7889` - Mixed content warnings, WebSocket authentication failures, broken real-time updates

**After:** `https://goaccess.mscob.com` - Clean SSL, WebSockets authenticated and working flawlessly, real-time updates streaming

The SSL setup was crucial here. Without proper SSL termination and header forwarding, the WebSocket handshake would fail.

**The Solution: Custom Docker Compose with Embedded Nginx**

Okay so this setup is a bit weird - instead of mounting nginx config files from the filesystem (which gets wiped on reboot if you're using /tmp like I initially was... don't ask), I embedded the ENTIRE nginx configuration directly in the Docker Compose file using Docker configs.

Why? Because it makes the whole thing completely portable and self-contained. Copy one YAML file, deploy anywhere. Chef's kiss. 👌

Here's the full config (don't forget to update `goaccess.yourdomain.com` with your actual domain, otherwise you're gonna have a bad time):

```yaml
version: "3.8"

services:
  goaccess:
    image: allinurl/goaccess:latest
    container_name: goaccess
    restart: unless-stopped
    volumes:
      - /var/log/nginx:/var/log/nginx:ro
      - goaccess-data:/srv/data
      - goaccess-html:/srv/report
    entrypoint: /bin/sh
    command:
      - -c
      - |
        echo 'Loading historical logs...';
        zcat -f /var/log/nginx/access.log.*.gz 2>/dev/null | goaccess - --log-format=COMBINED --db-path=/srv/data --persist || true;
        echo 'Starting real-time monitoring...';
        exec goaccess --log-file=/var/log/nginx/access.log --log-format=COMBINED --real-time-html --ws-url=wss://goaccess.yourdomain.com/ws --port=443 --addr=0.0.0.0 --output=/srv/report/index.html --db-path=/srv/data --persist --restore --ignore-crawlers
    networks:
      - goaccess

  nginx:
    image: nginx:alpine
    container_name: goaccess-web
    restart: unless-stopped
    ports:
      - "7889:80"
    volumes:
      - goaccess-html:/usr/share/nginx/html:ro
    configs:
      - source: nginx_config
        target: /etc/nginx/conf.d/default.conf
    networks:
      - goaccess

volumes:
  goaccess-data:
  goaccess-html:

networks:
  goaccess:

configs:
  nginx_config:
    content: |
      server {
          listen                   80;
          server_name              _;

          root                     /usr/share/nginx/html;
          index                    index.html;

          location / {
              try_files            $$uri $$uri/ /index.html;
          }

          location /ws {
              proxy_pass           http://goaccess:443/;
              proxy_http_version   1.1;
              proxy_set_header     Upgrade $$http_upgrade;
              proxy_set_header     Connection "Upgrade";
              proxy_set_header     Host $$host;
              proxy_set_header     Origin $$http_origin;
              proxy_set_header     X-Real-IP $$remote_addr;
              proxy_set_header     X-Forwarded-For $$proxy_add_x_forwarded_for;
              proxy_buffering      off;
              proxy_read_timeout   86400;
              proxy_send_timeout   86400;
          }
      }
```

**Key Features:**

1. **Historical Data Loading:** The startup script processes all compressed nginx logs (`access.log.*.gz`) before starting real-time monitoring. This means you get historical data from day one, not just current traffic. The line `zcat -f /var/log/nginx/access.log.*.gz 2>/dev/null | goaccess - --log-format=COMBINED --db-path=/srv/data --persist || true;` handles this. **Note:** This assumes your nginx uses the standard `COMBINED` log format and that the logs are mounted at `/var/log/nginx`. If your setup differs, adjust the path and/or `--log-format` accordingly.

2. **Embedded Nginx Config:** Instead of mounting config files, the nginx configuration is embedded directly in the compose file using Docker `configs`. Notice the `$$` escaping for nginx variables - Docker Compose requires double `$` to pass a single `$` through to the config file.

3. **WebSocket Proxying:** The internal nginx container handles WebSocket proxying from `/ws` to the GoAccess WebSocket server on port 443.

4. **Persistent Database:** The `goaccess-data` volume stores the persistent database, so your stats survive restarts.

5. **Port Configuration:** GoAccess listens internally on port 443 (`--port=443`), and the nginx container exposes port 7889 externally. This REALLY confused me for like 3 hours until I realized the `--port` flag does double duty - it controls BOTH the internal listening port AND the port number that gets hardcoded into the JavaScript WebSocket URL. Once I figured that out, everything clicked. You can change the external port by modifying `"7889:80"` in the YAML to whatever you want.

**Deploy it:**

1. Save as `docker-compose.yml` (or use in Portainer stacks)
2. Update `--ws-url=wss://goaccess.yourdomain.com/ws` with your actual domain
3. Update the volume mount `/var/log/nginx:/var/log/nginx:ro` to point to your actual nginx logs directory
4. (Optional) Change port 7889 to your preferred port in the `ports:` section
5. Run: `docker-compose up -d` (or deploy in Portainer)
6. Add proxy host in Nginx Proxy Manager pointing to your chosen port with WebSocket support enabled

First startup will take longer as it processes historical logs. Subsequent restarts are much faster.

**OctoPrint had similar challenges** - it also uses WebSockets for real-time printer status and camera feeds. Getting `https://octoprint.mscob.com` working required:

- Enabling WebSocket support in Nginx Proxy Manager proxy host
- Adding custom nginx headers for WebSocket upgrade
- Configuring OctoPrint to trust the proxy

These WebSocket-heavy applications were definitely the hardest part of the whole setup, but once you understand the pattern (proper headers, SSL termination, WebSocket-aware proxying), it becomes repeatable (which was nice, since I also had to do for OctoPrint).

---

## Testing Your Setup

**1. DNS Resolution**

```bash
nslookup proxmox.mscob.com
# Should return: <your Nginx Proxy Manager server IP>
```

**2. Certificate Validity**

```bash
openssl s_client -connect proxmox.mscob.com:443 -servername proxmox.mscob.com < /dev/null
# Look for:
# subject=CN = *.mscob.com
# issuer=C = US, O = Let's Encrypt
```

**3. Browser Test**

Open `https://proxmox.mscob.com`:

- ✅ Page loads
- ✅ Green lock icon
- ✅ Valid certificate (click lock to verify)
- ✅ No warnings

**4. External Access Test**

From outside your network:

```bash
nslookup proxmox.mscob.com
# Should return: Your placeholder IP (NOT your internal IP)
# Connection should fail - services are internal only
```

---

## Security Considerations

**What's Exposed:**

- ✅ DNS records (public, but that's normal)
- ✅ Domain ownership (anyone can see you own mscob.com)

**What's NOT Exposed:**

- ❌ Service IPs
- ❌ Open ports
- ❌ Service types
- ❌ Any actual traffic
- ❌ Network topology

**Additional Security:**

1. **Cloudflare API Token** - Scope to single zone, DNS edit only
2. **Nginx Proxy Manager Admin UI** - Only accessible on local network (port 81)
3. **Firewall Rules** - Router blocks all external access to internal network
4. **HTTPS Only** - Force SSL enabled on all proxy hosts
5. **HSTS** - Prevents downgrade attacks

---

## Maintenance

**Certificate Renewal:**

Let's Encrypt certificates expire every 90 days. Nginx Proxy Manager automatically renews them at 60 days using the same DNS challenge process. You'll get email notifications before expiry.

**To manually renew:**
Nginx Proxy Manager UI → SSL Certificates → Click ... menu → Renew

**Adding New Services:**

1. Add DNS entry in your router/<a href="https://pi-hole.net/" target="_blank" rel="noopener noreferrer">Pi-hole</a>: `newservice.mscob.com → <proxy-manager-ip>`
2. Add proxy host in Nginx Proxy Manager pointing to actual service
3. Use existing `*.mscob.com` certificate
4. Done!

---

## Troubleshooting

### DNS Not Resolving

**Problem:** `nslookup proxmox.mscob.com` doesn't return your Nginx Proxy Manager server IP

**Fix:**

1. Verify custom DNS entries in your router/Pi-hole
2. Check device is getting DNS from your local DNS server
3. Flush DNS cache: `sudo dscacheutil -flushcache` (macOS) or `ipconfig /flushdns` (Windows)

### Certificate Invalid

**Problem:** Browser shows "Not Secure" or cert warnings

**Fix:**

1. Verify cert was issued: Nginx Proxy Manager → SSL Certificates
2. Check proxy host is using correct cert (SSL tab)
3. Ensure Cloudflare proxy is OFF for DNS records
4. Check Cloudflare API token has DNS edit permission

### WebSocket Connections Failing

**Problem:** Services like GoAccess or OctoPrint show "Unable to authenticate WebSocket" or real-time updates don't work

**Fix:**

1. Enable "Websockets Support" in Nginx Proxy Manager proxy host (Details tab)
2. Add custom WebSocket headers in Nginx Proxy Manager Advanced tab:
   ```nginx
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "Upgrade";
   proxy_set_header Host $host;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto $scheme;
   ```
3. Ensure the backend service:
   - Listens on all interfaces (0.0.0.0), not just localhost
   - Uses correct WebSocket URL scheme (wss:// for HTTPS domains)
   - Trusts the proxy (some apps like OctoPrint have "trust proxy" settings)

**Note:** This was legitimately the hardest part of my entire setup. I spent more time debugging WebSocket authentication failures than I'd like to admit. GoAccess needed the custom Docker setup you saw above. OctoPrint needed a bunch of specific headers. The debugging pain is a one-time cost though.

---

## Cost Breakdown

- **Domain Registration:** ~$12/year (varies by TLD)
- **Cloudflare:** **Free** (free tier is sufficient)
- **SSL Certificates:** **Free** (Let's Encrypt)
- **Nginx Proxy Manager:** **Free** (open source)
- **Router/DNS:** Already owned for network infrastructure

**Total ongoing cost: $12/year**

---

## Lessons Learned

**1. DNS Challenge is a Game Changer**

Before I figured out DNS challenge, I tried all the wrong things:

- **Self-signed certs** - Browser warnings everywhere, plus you have to manually install your CA certificate on every single device. Your phone, tablet, laptop, your partner's devices, guest devices... it never ends. Hard pass.
- **Exposing port 80 temporarily** - Yeah, let me just open my internal services to the internet real quick, what could go wrong?
- **mDNS/Bonjour** - Works great until you try it on literally anything that isn't a Mac or iPhone.

DNS challenge is the ONLY way to get valid, trusted certificates without exposing anything to the internet AND without installing certs on every device. It just works.

**2. Local DNS Override is Critical**

Without local DNS override, traffic would route through Cloudflare proxy or fail entirely. Your router's (or Pi-hole's) ability to override DNS for internal IPs makes the whole system work. Most modern routers support this, and it's one of Pi-hole's core features.

**3. Wildcard Certs Simplify Everything**

Getting a separate cert for each service is possible but painful. A single `*.mscob.com` wildcard cert covers unlimited subdomains and simplifies management.

**4. WebSockets Need Special Handling (The Hardest Part)**

Look, if you're running apps with WebSockets (GoAccess, OctoPrint, Proxmox, Portainer, etc.), buckle up. This is where you'll spend most of your troubleshooting time.

WebSockets are picky about:

- **Headers** - It's not just enabling "WebSocket support" checkbox in Nginx Proxy Manager. You need the RIGHT headers or it just silently fails.
- **The Upgrade and Connection headers** - These need to be forwarded correctly or the handshake fails
- **Binding** - Services need to bind to 0.0.0.0, not 127.0.0.1 (found this out the hard way)
- **URL scheme** - Must use wss:// (not ws://) when accessing via HTTPS
- **Trust settings** - Some apps (looking at you, OctoPrint) need explicit "trust the proxy" configuration

GoAccess required that completely custom Docker setup you saw earlier with embedded nginx config. OctoPrint needed trust proxy settings AND specific headers. These weren't "just add a proxy host and you're done" situations - they required actual debugging and nginx config work.

The good news is once you figure out the pattern for one WebSocket app, it applies to the rest. Small victories. 🎉

---

## Conclusion

This setup completely transformed my home lab. No more memorizing IP addresses and port numbers. No more browser security warnings. No more explaining to family members how to type `http://192.168.1.whatever:port` into their phones.

Just clean URLs with valid SSL certificates. Everything stays completely internal. It auto-renews too.

**What I actually got out of this:**

- ✅ Actual memorable URLs (proxy.mscob.com beats 192.168.1.10:81 every time)
- ✅ Valid SSL certificates with the green lock (no more clicking through warnings)
- ✅ Can actually share links with people without explaining port numbers
- ✅ Set-it-and-forget-it automatic certificate renewal
- ✅ Some apps literally require HTTPS to work properly, so this unlocked new functionality
- ✅ Looks professional AF
- ✅ Zero internet exposure (firewall stays locked down)

The best part? Total ongoing cost is $12/year for the domain. That's it. Everything else is free.

---

## Additional Resources

- <a href="https://nginxproxymanager.com/" target="_blank" rel="noopener noreferrer">Nginx Proxy Manager Official Docs</a>
- <a href="https://letsencrypt.org/docs/" target="_blank" rel="noopener noreferrer">Let's Encrypt Documentation</a>
- <a href="https://developers.cloudflare.com/api/" target="_blank" rel="noopener noreferrer">Cloudflare API Documentation</a>
- <a href="https://community-scripts.github.io/ProxmoxVE/" target="_blank" rel="noopener noreferrer">Proxmox Community Scripts</a>

---

**Questions? Found this helpful?** Feel free to reach out or adapt this setup for your own home lab!

_Last Updated: February 17, 2026_
