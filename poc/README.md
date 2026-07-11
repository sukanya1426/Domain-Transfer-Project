# Sigmoix Domain Onboarding POC

A proof-of-concept for the "add your domain" flow: a website goes live instantly on
an auto-generated platform subdomain, then can be mapped to a customer's own custom
domain — both served over HTTPS with automatic Let's Encrypt certificates via a shared
Traefik reverse proxy.

## Goal

Demonstrate the two-stage onboarding path used by platforms like Vercel/Netlify:

1. **Instant subdomain** — a site is reachable with a valid TLS cert the moment the
   container starts, with zero DNS configuration.
2. **Custom domain** — the customer points their domain's DNS at the VM and the same
   site is automatically served on their domain with its own cert.

## Input

- Static website served by Nginx (`website/index.html`).
- A VM (Hetzner Ubuntu, IP `37.27.203.157`) running Docker and a shared Traefik instance
  with a `letsencrypt` cert resolver, watching the external `traefik-net` Docker network.

## Output

The website reachable over HTTPS, with real Let's Encrypt certificates, on:

- `https://poc.37.27.203.157.sslip.io` — platform subdomain, works immediately.
- `https://sukanya1426.me` / `https://www.sukanya1426.me` — custom domain, after DNS is pointed at the VM.

## How it works

- **Platform subdomain** uses [sslip.io](https://sslip.io) wildcard DNS, which resolves any
  hostname containing an embedded IP straight to that IP. No domain, registrar, or nameserver
  setup is needed — this is the `*.vercel.app` equivalent.
- **Custom domains** only serve once the domain owner adds an A record pointing to the VM.
- Traefik routes each host to the website container and issues per-host certs, all driven by
  the Docker labels in `docker-compose.yml`. HTTP (port 80) is redirected to HTTPS.

## Configuration

Routing and TLS are defined by Docker labels in `docker-compose.yml`. To adapt this POC:

1. **Set the VM IP** — replace `37.27.203.157` in the platform subdomain host rule
   (`poc.<IP>.sslip.io`) with your VM's public IP.
2. **Set the custom domain** — replace `sukanya1426.me` / `www.sukanya1426.me` in the
   `custom` router's `Host(...)` rule with the customer's domain, and have the owner add an
   A record for it pointing to the VM IP.
3. **Confirm Traefik prerequisites** — the shared Traefik instance must have the `letsencrypt`
   cert resolver configured and watch the external `traefik-net` network.

## Deploy

Run on the VM, from the `poc` directory:

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f website
```

Requirements on a fresh VM: Docker + the Docker Compose plugin, and the external
`traefik-net` network created (`docker network inspect traefik-net`).

## Verify

```bash
# Platform subdomain — works immediately, no DNS setup
curl -I https://poc.37.27.203.157.sslip.io     # 200, valid Let's Encrypt cert

# Custom domain — only after the owner points DNS at the VM
curl -I https://sukanya1426.me
```

## Troubleshooting

- **Cert issuance fails** — check that port 80 is reachable from the internet, the
  `letsencrypt` resolver is configured, and (for custom domains) that DNS has propagated to the VM.
- **404** — confirm the `Host(...)` label rules on the website container match the URL.
- **Container unreachable** — confirm it is attached to `traefik-net` and Traefik uses the same network.
