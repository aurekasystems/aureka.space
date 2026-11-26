# Deployment on Blossom

For now `aureka.space` is not a website of its own: the shared Blossom proxy
(`aureka-proxy-caddy`) answers both names with a **temporary redirect (302)**
to `https://aureka.at/products/space`. There is no container and no checkout on the server.

Why 302 and not 301: browsers cache a 301 indefinitely. Anyone who visits
`aureka.space` now would keep landing on `aureka.at` even after the real site
launches.

Like `aureka.at`, this runs on Blossom only temporarily (Blossom is a
preview/staging platform per `aurekasystems/infrastructure`), so the route is
kept here and not in the infrastructure repository.

## Files

- `deploy/blossom/aureka-space.caddy`: route for the shared proxy;
  `aureka.space` and `www.aureka.space` → `https://aureka.at/products/space` (302).

## 1. Route in the shared proxy (as `mjs`)

Copy `deploy/blossom/aureka-space.caddy` to Blossom (e.g. `scp` to `~/`),
then validate it in a check copy first; the running proxy is not touched:

```bash
C=/srv/proxy/caddy/data/config
sudo install -d -m 755 $C/aureka-space-check
sudo cp ~/aureka-space.caddy $C/aureka-space-check/
{ cat /srv/proxy/caddy/app/Caddyfile; echo 'import /config/aureka-space-check/*.caddy'; } | sudo tee $C/Caddyfile.aureka-space-check >/dev/null
sudo docker exec aureka-proxy-caddy-caddy-1 caddy validate --config /config/Caddyfile.aureka-space-check --adapter caddyfile
```

Activate it, with a backup of `sites-enabled`. Reload only after a successful
validation:

```bash
sudo cp -a $C/sites-enabled $C/backups/sites-enabled-$(date +%Y%m%d-%H%M%S)
sudo cp ~/aureka-space.caddy $C/sites-enabled/aureka-space.caddy
sudo docker exec aureka-proxy-caddy-caddy-1 caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo docker exec aureka-proxy-caddy-caddy-1 caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
sudo rm -r $C/aureka-space-check $C/Caddyfile.aureka-space-check
```

Then check that the other sites behind the proxy still answer as usual.

## 2. DNS and verification

Set at Infomaniak (lower the TTL to 300 beforehand); leave MX and mail
records unchanged:

| Name | Type | Value |
| --- | --- | --- |
| `aureka.space` | A | `179.237.76.73` |
| `aureka.space` | AAAA | `2001:1600:16:10::acf` |
| `www.aureka.space` | A | `179.237.76.73` |
| `www.aureka.space` | AAAA | `2001:1600:16:10::acf` |

Follow the certificate issuance until `certificate obtained successfully`
appears for both names (a redirect over HTTPS still needs a certificate):

```bash
sudo docker logs -f --since 2m aureka-proxy-caddy-caddy-1 2>&1 | grep -iE 'aureka\.photos|obtain|error'
```

Then:

```bash
curl -I https://aureka.space/
curl -I https://www.aureka.space/
```

Expected: `HTTP/2 302` with `location: https://aureka.at/products/space`.

## Rollback / launching the real site

Remove `sites-enabled/aureka-space.caddy`, validate and reload the proxy.
For the real launch, replace the route with a `reverse_proxy` to a site
container, following the pattern in `aurekasystems/aureka.at`
(`docs/DEPLOYMENT.md`).
