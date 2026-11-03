# infra

Server-level configuration shared by the apps that run on my server. The apps themselves live in their own repos; this repo holds only what none of them owns.

Never commit secrets, IP addresses or `.env` files.

## Layout

```
/opt/infra/proxy        Caddy: the only container that publishes ports (80/443)
/opt/easy-admin         app stack, its web service joins the "proxy" network
/opt/family-finances    same pattern
```

```
internet ──80/443──▶ caddy ──"proxy" network──▶ easy-admin:3000
                                            └─▶ family-finances:8000
```

- **One Caddy for the whole server.** Only one process can listen on 443, so every domain goes through this one. It gets and renews the Let's Encrypt certificates and keeps them in the `caddy_data` volume.
- **Each app owns its site block** in its repo under `deploy/caddy/<app>.caddy`. `proxy/docker-compose.yml` mounts that directory into `/etc/caddy/sites/<app>/`, and `proxy/Caddyfile` imports each one (one line per app).
- **Apps reach the proxy through the external `proxy` network**, created by this stack. Each app joins it with a network alias equal to its name (`easy-admin`), because several apps have a service called `web` and the alias keeps them apart. Databases and other internal services stay on each app's default network.

## Proxy

```sh
cd /opt/infra/proxy
cp .env.example .env              # one *_DOMAIN per app
docker compose up -d
```

Start the proxy before the apps: their compose files expect the `proxy` network to exist.

After changing a site block or `.env`, reload without downtime:

```sh
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile   # site blocks
docker compose up -d                                                   # .env or compose changes
```

## Adding an app

1. In the app repo: add `deploy/caddy/<app>.caddy` with a site block using `{$<APP>_DOMAIN}` and `reverse_proxy <app>:<port>`, and join its web service to the external `proxy` network with the alias `<app>`.
2. Here: add the mount in `proxy/docker-compose.yml`, its `import` line in `proxy/Caddyfile` and the domain to `.env.example`.
3. On the server: point the domain's DNS A record at the server, set the domain in `proxy/.env`, `docker compose up -d` here, then start the app.
