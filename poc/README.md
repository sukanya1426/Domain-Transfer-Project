# Sigmoix Domain Onboarding POC

This workspace contains a small proof-of-concept for onboarding a website from a platform-managed subdomain and later mapping it to a custom domain using Docker, an existing shared Traefik instance, and Let's Encrypt.

## Directory Structure

- `poc/website/index.html` - static demo homepage
- `poc/website/Dockerfile` - production image for the website
- `poc/website/nginx.conf` - Nginx site configuration
- `poc/docker-compose.yml` - website deployment that joins the shared Traefik network
- `poc/traefik-data/traefik.yml` - Traefik static configuration
- `poc/traefik-data/acme.json` - Let's Encrypt certificate store

## What the demo serves

- `Sigmoix Domain Onboarding Demo`
- the current hostname
- the current timestamp
- a success message

## Domains

### Platform subdomain — no DNS configuration required

The platform subdomain uses **sslip.io wildcard DNS**, which resolves any
hostname containing an embedded IP straight to that IP. No domain, registrar,
or nameserver setup is ever needed.

- `poc.37.27.203.157.sslip.io` -> `37.27.203.157` (automatic, always)

This is the zero-config equivalent of a `*.vercel.app` subdomain: it works the
instant the container is up and Let's Encrypt issues a real certificate for it
over the HTTP-01 challenge.

> We deliberately do **not** use `sigmoix.com` for the platform subdomain — its
> authoritative nameservers are Afternic's (`ns1/ns2.afternic.com`), not a zone
> we control, so records added in Namecheap are never served publicly.

### Custom domains — customer points DNS to the VM

These only begin serving once the domain owner adds an A record to the VM. This
is the real "add your domain" onboarding step.

- `sukanya1426.me` -> `37.27.203.157`
- `www.sukanya1426.me` -> `37.27.203.157`

## VM setup commands

Run these on the Hetzner Ubuntu VM:

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://get.docker.com | sh
apt-get install -y docker-compose-plugin
systemctl enable docker
systemctl start docker
docker version
docker compose version
docker network inspect traefik-net
```

## Deployment commands

From the `poc` directory on the VM:

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f website
```

Before the first public deploy, confirm that the shared Traefik instance already has the `letsencrypt` certificate resolver configured and that it watches the `traefik-net` Docker network.

## Verification commands

```bash
# Platform subdomain — should work immediately, no DNS setup
curl -I http://poc.37.27.203.157.sslip.io      # 308 redirect to HTTPS
curl -I https://poc.37.27.203.157.sslip.io     # 200, valid Let's Encrypt cert

# Custom domains — only after the owner points DNS to 37.27.203.157
curl -I https://sukanya1426.me
curl -I https://www.sukanya1426.me
```

## Traefik notes

- HTTP traffic on port 80 is redirected to HTTPS.
- The website container must join the same Docker network as Traefik.
- Traefik routes are defined by Docker labels on the website container.
- The shared Traefik instance should already be responsible for dashboard access and certificate storage.

## Troubleshooting

- The platform subdomain (`*.sslip.io`) needs no DNS work; if its cert fails, check that port 80 is reachable from the internet and the shared Traefik instance has the `letsencrypt` resolver.
- If a custom-domain certificate issuance fails, verify that DNS has propagated to `37.27.203.157`, port 80 is reachable, and the shared Traefik instance is listening for the `letsencrypt` resolver.
- If the website returns 404, confirm the host rule labels on the website container (and, for custom domains, the DNS records).
- If the website container is not reachable, check that the container is attached to `traefik-net` and that Traefik is using the same network.