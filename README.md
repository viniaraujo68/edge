# edge

Proxy de borda da VPS: um Caddy que roteia por hostname para as apps e um `cloudflared` que
traz o tráfego do Cloudflare por um **Cloudflare Tunnel**. A VPS não publica porta HTTP
nenhuma — o origin não é alcançável pelo IP, e o visitante só fala HTTPS com o Cloudflare.

```
Visitante ─HTTPS→ Cloudflare ─túnel→ cloudflared → edge (Caddy :80, rede web)
    ├── peladex.rastrovin.org  → /api/* → peladex-backend:8000 · / → peladex-frontend:3000
    ├── pokerdex.rastrovin.org → /api/* → pokerdex-backend:8000 · / → pokerdex-frontend:3000
    └── rastrovin.org          → rastro-frontend:80
```

| App | Repo | Bloco |
|-----|------|-------|
| Peladex | [viniaraujo68/peladex](https://github.com/viniaraujo68/peladex) | `sites/peladex.caddy` |
| Pokerdex | [viniaraujo68/pokerdex-v3](https://github.com/viniaraujo68/pokerdex-v3) | `sites/pokerdex.caddy` |
| Rastro | [viniaraujo68/rastro](https://github.com/viniaraujo68/rastro) | `sites/rastro.caddy` |

Cada app sobe o próprio `docker compose` e entra na rede externa `web` com aliases
(`peladex-backend`, `pokerdex-frontend`, `rastro-frontend`…). O edge só conhece esses nomes.

## IP do visitante

O Cloudflare manda o IP real em `Cf-Connecting-IP`. O Caddy só aceita esse header de quem
está numa faixa privada (o `cloudflared`, na rede `web`), e cada site o repassa como um
`X-Forwarded-For` de valor único (`header_up X-Forwarded-For {client_ip}`). É esse valor que
o rate limit dos backends (uvicorn `--proxy-headers`) e o `XFF_DEPTH: "1"` dos frontends
SvelteKit leem; no rastro, o nginx acrescenta o IP do edge e o gin resolve o visitante. Como o
edge não publica porta, ninguém de fora forja o header.

## Subir numa VPS zerada

```bash
docker network create web

git clone https://github.com/viniaraujo68/edge.git /opt/edge
cd /opt/edge
cp .env.example .env        # domínios + TUNNEL_TOKEN
docker compose up -d
docker compose logs tunnel | grep -i registered
```

Depois, cada app no seu diretório: `docker compose up -d --build`.

## Cloudflare

1. **Networking → Tunnels → Create a tunnel** (Cloudflared), no dashboard principal — não
   precisa do Zero Trust. O token é a string depois de `--token` no comando de instalação;
   vai para `TUNNEL_TOKEN` no `.env`.
2. **Public Hostname**: um por app, todos com tipo `HTTP` e URL `edge:80`. Se já existir
   registro A/AAAA com o mesmo nome no DNS, apague antes — o túnel cria um CNAME.
3. **SSL/TLS → Overview**: Full (strict).
4. **SSL/TLS → Edge Certificates**: Always Use HTTPS; HSTS (max-age 6 meses,
   includeSubDomains, sem preload); Minimum TLS 1.2.
5. Firewall da Hetzner: nenhuma regra de entrada além do SSH.

## Acrescentar uma app

1. Na app: `docker compose` com os serviços na rede externa `web` e aliases próprios.
2. Aqui: um `sites/<app>.caddy` no formato dos outros (com o `header_up`), e a variável de
   domínio em `.env.example`.
3. Na VPS:

   ```bash
   cd /opt/edge && git pull
   echo 'NOVA_DOMAIN=nova.rastrovin.org' >> .env
   docker compose up -d
   docker compose exec edge caddy reload --config /etc/caddy/Caddyfile
   ```

4. No túnel, um Public Hostname novo → `HTTP` `edge:80`.

O `reload` é a quente: as outras apps não caem. Se o Caddyfile tiver erro, o `reload` falha e
o Caddy segue com a config antiga; para conferir antes,
`docker compose exec edge caddy validate --config /etc/caddy/Caddyfile`. O bloco
`:80 { respond 404 }` recusa host desconhecido.

## Conferir

```bash
curl -sI http://peladex.rastrovin.org | head -1     # 301 para https
curl -sI https://peladex.rastrovin.org | head -1    # 200
curl -m 5 http://<IP-da-VPS>                        # timeout
```

No log do backend de uma app, a requisição que você fez deve aparecer com o seu IP, não com
um do Cloudflare.
