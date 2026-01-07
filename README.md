# actual-budget-fsp
Deployment Setup for self-hosting [Actual Budget](https://github.com/actualbudget/actual) on a Raspberry Pi

This repository contains a minimal Docker Compose setup to run the Actual Budget server behind a [Caddy](https://github.com/caddyserver/caddy) reverse proxy on a small self-hosted machine (e.g. Raspberry Pi). The configuration is intentionally simple so a hobbyist engineer can inspect and extend it.

**Files**
- **docker-compose.yml**: Starts two services — `caddy` (the reverse proxy) and `actual_server` (the Actual Budget backend). See [docker-compose.yml](docker-compose.yml).
- **Caddyfile**: Caddy configuration that routes requests for `raspberrypi.local` to the Actual Budget container and enables automatic TLS using Caddy's internal CA. See [Caddyfile](Caddyfile).

**What the compose does (overview)**
- **Caddy service**:
	- Runs the official `caddy:latest` image and exposes ports `80` and `443` to the LAN.
	- Mounts the local `Caddyfile` into `/etc/caddy/Caddyfile` so the reverse-proxy rules are editable from the host.
	- Persists Caddy's certificates and runtime state in the `./caddy_data` and `./caddy_config` directories (so TLS state survives restarts).
	- Joins the custom `web` network so it can reach the `actual_server` container by name.

- **Actual Budget server (`actual_server`)**:
	- Uses the `docker.io/actualbudget/actual-server:latest` image.
	- Persists application data in `./actual-data` mapped to `/data` inside the container.
	- Has a `healthcheck` that runs the container's Node.js health-check script; Docker will mark the container unhealthy if the check fails repeatedly.
	- Also joins the `web` network so Caddy can proxy to it.

**What the Caddyfile does**
The provided `Caddyfile` contains a simple site definition:

```
raspberrypi.local {
		tls internal
		reverse_proxy actual_server:5006
}
```

- `raspberrypi.local` — the site hostname Caddy will serve. On a typical home LAN this resolves to the Pi's hostname; you can change it to a real domain if you prefer.
- `tls internal` — instructs Caddy to use its internal CA to mint a certificate. This gives you HTTPS on the LAN without Let's Encrypt, but you may need to trust Caddy's internal CA on client devices if browsers warn about the certificate.
- `reverse_proxy actual_server:5006` — forwards incoming requests to the `actual_server` container on port `5006` over the Docker `web` network.

**Persistent data and directories created**
- `./actual-data` — Actual Budget server data directory (database, attachments, etc.).
- `./caddy_data` and `./caddy_config` — Caddy runtime state and certificates.

**How to run**
1. Ensure Docker and Docker Compose are installed on your Raspberry Pi.
2. From this repository directory, bring the stack up:

```sh
docker compose up -d
# (or: docker-compose up -d on systems that use the legacy binary)
```

3. Check service status:

```sh
docker compose ps
docker compose logs -f caddy
docker compose logs -f actual_server
```

4. Visit `https://raspberrypi.local` from a device on the same network. If you see a certificate warning, you can either:
	- Trust Caddy's internal CA on your client device, or
	- Replace `tls internal` with a valid `tls` setup (Let’s Encrypt or your own certificate) and update the volumes accordingly.

**Notes and tips**
- The `healthcheck` in `docker-compose.yml` helps Docker restart the server automatically if its internal health script reports problems.
- If you change the `Caddyfile`, reload Caddy by restarting the `caddy` container:

```sh
docker compose restart caddy
```

- If you want to expose the service to the internet, replace `raspberrypi.local` with your public domain and configure DNS, firewall/NAT, and a trusted TLS setup.