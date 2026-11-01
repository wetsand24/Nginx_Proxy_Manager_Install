# Nginx Proxy Manager Install (Docker, 2026)

Nginx Proxy Manager in front of your homelab apps, with a free DuckDNS name and one Let's Encrypt wildcard
certificate. Every app gets its own name, like `https://jellyfin.YOURNAME.duckdns.org`, with a trusted certificate
and no browser warnings. It works on your home network, and away from home through Tailscale. You don't forward any
ports on your router. This gives you clean names and HTTPS. It doesn't secure the apps behind it: their logins,
updates and passwords are still up to you.

Video: [Nginx Proxy Manager Setup 2026: Local SSL Without Port Forwarding](https://youtu.be/U9tIwCmWctc) on [@kadirrangwala](https://www.youtube.com/@kadirrangwala)

Tested on Debian 13 with Nginx Proxy Manager 2.16.0.

Placeholders used below. Replace them with your own values:

| Placeholder | What it is | In the video |
|---|---|---|
| `PROXY-IP` | the local IP of the machine running Nginx Proxy Manager | `10.37.0.54` |
| `YOURNAME` | your DuckDNS subdomain | `kr-homelab` |
| `YOUR-TOKEN` | your DuckDNS token (or Cloudflare API token) | blurred |
| `APP-IP` | the local IP of the machine running the app | `10.37.0.53` |

---

## What you need

- A Linux machine or VM on your network for the proxy, with a fixed IP. A DHCP reservation in your router is fine.
  It needs `sudo` and `curl` (on Debian, as root: `apt install -y sudo curl`).
- Your apps already running.
- A free [DuckDNS](https://www.duckdns.org) account.
- A free [Tailscale](https://tailscale.com) account, for the remote part at the end.

## Files

| File | Where it goes | What to change |
|---|---|---|
| [`compose.yaml`](compose.yaml) | `~/npm/compose.yaml` on the proxy machine | nothing |

## 1. Install Docker and Nginx Proxy Manager

SSH into the proxy machine (type `yes` the first time to trust its key):

```bash
ssh YOURUSER@PROXY-IP
```

Update the package lists and install Docker with the official script:

```bash
sudo apt update
curl -fsSL https://get.docker.com | sudo sh
sudo docker --version
```

Skip the Docker lines if Docker is already installed.

Make a folder and create the compose file:

```bash
mkdir npm && cd npm
nano compose.yaml        # paste the file below, Ctrl+O, Enter, Ctrl+X
```

[`compose.yaml`](compose.yaml):

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

- `restart: unless-stopped`: it comes back after a reboot.
- Ports `80` and `443`: the normal web ports, where your apps are served.
- Port `81`: the admin page.
- The two volumes keep your settings and certificates in this folder, so nothing is lost when the container updates.

Start it and check it's running:

```bash
sudo docker compose up -d
sudo docker ps
```

## 2. Create the admin account

Open `http://PROXY-IP:81` in a browser.

On the current version (2.16) you create the admin account on this first page: name, email, password, Save.
Older guides use a default login (`admin@example.com` / `changeme`). That's gone now.

Use a real email address. Nginx Proxy Manager also gives it to Let's Encrypt when it asks for certificates, and
Let's Encrypt rejects addresses like `@example.com`.

## 3. Free domain with DuckDNS

1. Sign in at [duckdns.org](https://www.duckdns.org) (Google, GitHub and a few others).
2. Your **token** is at the top. Keep it secret. It works like a password for all your DuckDNS domains.
3. Type the subdomain you want (`YOURNAME`) and click **add domain**.
4. **DuckDNS fills in your public IP by default. Replace it** with the proxy's local IP, `PROXY-IP`.
5. Click **update ip**.

Don't run a DuckDNS updater script or container for this name. Those update it to your public IP again.

### Check it

Any name in front of `YOURNAME.duckdns.org` returns the proxy's IP (DuckDNS gives you a wildcard). This asks
Cloudflare's DNS at `1.1.1.1` and works on Windows too:

```bash
nslookup jellyfin.YOURNAME.duckdns.org 1.1.1.1
nslookup anything.YOURNAME.duckdns.org 1.1.1.1
```

Both should answer `PROXY-IP`.

### A private IP in public DNS?

A `10.x`, `192.168.x` or `172.16-31.x` address only works inside a local network. Anyone can look up the name and
see the IP, but they can't connect to it. The upside: every device at home finds the proxy with no extra setup, and
devices on Tailscale find it too. Some routers block names that point to a private IP. See
[If a name doesn't work](#if-a-name-doesnt-work).

## 4. Wildcard certificate

The certificate comes from Let's Encrypt with the DNS challenge. Nginx Proxy Manager uses your DuckDNS token to put
a code in a DNS record, Let's Encrypt checks it, and you get the certificate. Let's Encrypt never connects to your
network, so no ports are opened. Nginx Proxy Manager renews it by itself.

The other option, **Let's Encrypt via HTTP**, needs port 80 forwarded to the internet and the name pointed at your
public IP, and it can't do wildcards. Not used here.

In Nginx Proxy Manager: **Certificates** → **Add Certificate** → **Let's Encrypt via DNS**.

| Field | Value |
|---|---|
| Domain Names | `*.YOURNAME.duckdns.org` |
| Key Type | ECDSA (left as is) |
| DNS Provider | DuckDNS (type `duck` to search) |
| Credentials File Content | `dns_duckdns_token=YOUR-TOKEN` |
| Propagation Seconds | `120` |

Add **only the star name**. DuckDNS holds one challenge record at a time, so adding the plain
`YOURNAME.duckdns.org` next to it can make the request fail.

Nginx Proxy Manager warns that the token is stored in plain text. Never open the admin page (port 81) to the internet.

**Save**. It takes about two minutes (about 130 seconds in my tests). The certificate is valid for 90 days and renews
by itself.

## 5. Proxy hosts

**Hosts** → **Proxy Hosts** → **Add Proxy Host**.

Jellyfin, as in the video:

| Field | Value |
|---|---|
| Domain Names | `jellyfin.YOURNAME.duckdns.org` |
| Scheme | `http` |
| Forward Hostname / IP | `APP-IP` (in the video `10.37.0.53`) |
| Forward Port | `8096` |
| Block Common Exploits | on |
| Websockets Support | on |

**SSL** tab:

| Field | Value |
|---|---|
| SSL Certificate | `*.YOURNAME.duckdns.org` |
| Force SSL | on |
| HTTP/2 Support | on |

**Save**. The host shows **Online**, which means Nginx loaded the new config without errors. Open
`https://jellyfin.YOURNAME.duckdns.org`: the padlock shows, and the certificate is valid.

Notes:

- **Scheme** is the hop from the proxy to the app on your network. Leave it `http` unless the app only speaks HTTPS
  itself (Proxmox, for example, on port 8006), then pick `https`.
- If the app runs in Docker on the proxy machine itself, don't use `localhost`. Inside the container that means the
  proxy container. Use the machine's IP.
- Websockets: many apps use them for live updates, so turn it on.

### Vaultwarden

Vaultwarden needs HTTPS. The browser blocks its encryption features on a plain HTTP page, which is why it showed an
error at the start of the video. Same steps:

| Field | Value |
|---|---|
| Domain Names | `vault.YOURNAME.duckdns.org` |
| Scheme | `http` |
| Forward Hostname / IP | `APP-IP` |
| Forward Port | `8080` (the port you mapped for Vaultwarden) |
| Block Common Exploits, Websockets Support | on |
| SSL tab | the wildcard certificate, Force SSL, HTTP/2 Support |

Same certificate again. Every new app is one more proxy host.

## 6. Remote access with Tailscale

The names point to a private IP, so they only work at home. Tailscale lets your own devices reach the proxy from
anywhere, without forwarding ports. The proxy machine shares a route to its own IP only (`/32`), so devices on
your Tailscale network reach the proxy and nothing else on your home network.

**Proxmox LXC container?** Do the [one extra step on the host](#proxmox-lxc-container) first.

On the proxy machine:

```bash
curl -fsSL https://tailscale.com/install.sh | sudo sh
sudo tailscale up --advertise-routes=PROXY-IP/32
```

In the video: `sudo tailscale up --advertise-routes=10.37.0.54/32`.

Open the sign-in link it prints and click **Connect**. You'll see two warnings above the link (IPv6 forwarding is
disabled, and one about UDP GRO). Both are safe to ignore for this IPv4 `/32` route.

The route needs IPv4 forwarding. Docker already turns it on, so nothing to do if you followed step 1. See
[IP forwarding](#ip-forwarding) otherwise.

Check the machine's Tailscale address:

```bash
tailscale ip -4
```

### Approve the route

In the [Tailscale admin console](https://login.tailscale.com/admin/machines), **Machines**: the proxy machine shows a
**Subnets** badge, which means its route is waiting for approval.

1. Click the three dots (⋯) on that machine → **Edit route settings**.
2. Tick `PROXY-IP/32` → **Save**.
3. Same menu → **Disable key expiry**. Otherwise the machine gets logged out after 180 days and remote access stops.

## 7. Test from another network

On a device that's away from home (or on another network, like mobile data), the name still resolves, but the
connection times out. That's expected until Tailscale is on.

**Windows, Mac, phones:** install the Tailscale app and sign in with the same account. Routes are accepted by default.

**Linux:** routes are not accepted by default, so add `--accept-routes`:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --accept-routes
```

If Tailscale is already up on that machine: `sudo tailscale set --accept-routes`.

Open the sign-in link, **Connect**. Then open `https://jellyfin.YOURNAME.duckdns.org`: same padlock, same wildcard
certificate, through Tailscale.

---

## Proxmox LXC container

Tailscale needs `/dev/net/tun`, which an LXC container doesn't get by default. On the **Proxmox host**, add these two
lines to `/etc/pve/lxc/CTID.conf` (`CTID` = the container's ID, for example `154`):

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Then restart the container:

```bash
pct shutdown CTID
pct start CTID
```

This is from [Tailscale's LXC guide](https://tailscale.com/kb/1130/lxc-unprivileged) (Proxmox 7 and later).
Docker inside an LXC container also needs **Nesting** on (container → Options → Features).

## IP forwarding

The subnet route needs IPv4 forwarding on the proxy machine. Docker already turns `net.ipv4.ip_forward` on (tested
2026-11-01), so the Docker install from step 1 needs nothing. Check it:

```bash
sysctl net.ipv4.ip_forward     # 1 = on
```

Without Docker, turn it on as in [Tailscale's subnet router docs](https://tailscale.com/kb/1019/subnets):

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

The two warnings from `tailscale up` (IPv6 forwarding disabled, UDP GRO) are safe to ignore for this IPv4 `/32` route.

## Own a domain on Cloudflare instead?

Everything works the same with a domain on Cloudflare. Only the DNS record and the token change.

1. **DNS record.** Cloudflare → your domain → **DNS** → **Records** → **Add record**:

   | Type | Name | IPv4 address | Proxy status |
   |---|---|---|---|
   | A | `*` | `PROXY-IP` | **DNS only** (grey cloud) |

   It must be the grey cloud. With the orange cloud the name points to Cloudflare instead of your proxy, and it
   breaks. Records you already have (like `www`) keep working: a wildcard only answers names without their own record.

2. **API token.** Profile → **API Tokens** → **Create Token** → **Edit zone DNS** → **Use template**. Under
   **Zone Resources** pick **Specific zone** and your domain, so the token only works on that one domain.
   **Continue to summary** → **Create Token** → copy it.

3. **Certificate.** In Nginx Proxy Manager, same as step 4, with:

   | Field | Value |
   |---|---|
   | Domain Names | `*.YOURDOMAIN.com` |
   | DNS Provider | Cloudflare |
   | Credentials File Content | `dns_cloudflare_api_token=YOUR-TOKEN` |

Check with `nslookup jellyfin.YOURDOMAIN.com 1.1.1.1`, then add proxy hosts as `jellyfin.YOURDOMAIN.com`.

## Home Assistant behind Nginx Proxy Manager

Home Assistant returns an HTTP **400** error through the proxy until you tell it to trust the proxy. On Home Assistant
2026.10 this is done in the UI. The old `http:` block in `configuration.yaml` is ignored there (Home Assistant shows
a repair: "HTTP YAML configuration is ignored after migration").

1. Open Home Assistant on its own IP: `http://APP-IP:8123`.
2. **Settings** → **System** → **Network** → **HTTP server** → **Reverse proxy**.
3. Turn on **Trust X-Forwarded-For**.
4. **Add Trusted proxies**: `PROXY-IP`.
5. **Save** → **Save and restart**.
6. After the restart, Home Assistant asks you to confirm the new HTTP server settings. Click **Confirm**. If you don't
   confirm within 5 minutes, it goes back to the old settings.

Then add the proxy host: `homeassistant.YOURNAME.duckdns.org`, scheme `http`, `APP-IP`, port `8123`, Block Common
Exploits on, **Websockets Support on** (Home Assistant needs it), and the same SSL tab settings.

Tested on Home Assistant 2026.10.4.

## If a name doesn't work

**The name doesn't resolve at home.** The `nslookup` in step 3 asked Cloudflare directly, but your devices normally
ask your router. Some routers and DNS filters have **DNS rebind protection**, which blocks public names that point to
a private IP. Compare:

```bash
nslookup jellyfin.YOURNAME.duckdns.org 1.1.1.1    # asks Cloudflare
nslookup jellyfin.YOURNAME.duckdns.org            # asks your own network
```

If the first answers and the second doesn't, something on your network is blocking it. Fixes:

- Find DNS rebind protection in your router or DNS filter and add an exception for your full name,
  `YOURNAME.duckdns.org`, not just `duckdns.org`. That allows only your domain, and the protection stays on for
  everything else.
- If your router has no exception (common on ISP routers), add the names as **local DNS records** in Pi-hole or your
  router's local DNS: one record per app pointing to `PROXY-IP`, or a wildcard if yours supports it. The certificate
  works the same, because the DNS challenge doesn't care where the name points. This also keeps the names working at
  home if DuckDNS is down. Over Tailscale, those devices then need that local DNS too. My Pi-hole setup video shows the
  local DNS records part: https://youtu.be/k9GPXPp1KU0

**The certificate fails.**

- A typo in the token or the domain name. The line must be exactly `dns_duckdns_token=YOUR-TOKEN`.
- Propagation seconds too short. Use `120`.
- The plain `YOURNAME.duckdns.org` added next to `*.YOURNAME.duckdns.org`. DuckDNS holds one challenge record at a
  time. Use only the star name.
- The admin email is an `@example.com` address. Let's Encrypt rejects it.
- Let's Encrypt rate limits: at most 5 certificates for the exact same names per 7 days. If you hit it, wait. See
  [letsencrypt.org/docs/rate-limits](https://letsencrypt.org/docs/rate-limits/).

**502 Bad Gateway.** Wrong forward IP, port or scheme on the proxy host, or `localhost` used for an app on the same
machine. Check the app opens on `http://APP-IP:PORT` first.

**Works at home, not away.**

- The route isn't approved: admin console → Machines → proxy machine → ⋯ → **Edit route settings**.
- A Linux client isn't accepting routes: `sudo tailscale set --accept-routes`.
- The proxy machine's key expired: disable key expiry for it.

**Keep port 81 off the internet.** It's the admin page, and your DNS token is stored there in plain text. You don't
need to forward any ports for anything in this guide.

## Updating later

```bash
cd ~/npm && sudo docker compose pull && sudo docker compose up -d
```
