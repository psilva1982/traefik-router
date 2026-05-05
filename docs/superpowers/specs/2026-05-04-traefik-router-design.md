---
title: Traefik Router — Design Spec
date: 2026-05-04
status: approved
---

## Context

Router centralizado Traefik v3 para um VPS independente do projeto `agendify-traefik`.
Gerencia tráfego HTTP/HTTPS para domínios `timero.me`, `api.timero.me` e `rstec.dev.br`.
Roteamento dinâmico via Docker labels — sem roteamento estático no `traefik.yml`.

## Architecture

```
Internet
   │
   ├─ :80  ─→ redirect → :443
   └─ :443 ─→ Traefik ─→ containers via rede traefik_public
```

Modelo hub-and-spoke:
- **Hub**: este `traefik-router`
- **Spokes**: outros docker-compose no VPS, cada um com labels Traefik e conectados à rede `traefik_public` como externa

## Files

```
traefik-router/
├── docker-compose.yml   # Sobe o Traefik, define rede e volume ACME
└── traefik.yml          # Config estática: entrypoints, providers, ACME
```

## docker-compose.yml

- Imagem oficial `traefik:v3.6` (sem Dockerfile customizado)
- Monta `traefik.yml` via volume (read-only)
- Monta `/var/run/docker.sock` como read-only para provider Docker
- Volume nomeado `traefik-certificates` para persistir `acme.json`
- Rede `traefik_public` definida aqui, usada como `external: true` pelas apps
- `security_opt: no-new-privileges:true`
- `restart: unless-stopped`
- Logging com rotação (max 10m, 3 arquivos)

## traefik.yml (config estática)

- **log**: level INFO
- **accessLog**: bufferingSize 100
- **entryPoints**:
  - `web` (:80) → redireciona para `websecure` via HTTPS
  - `websecure` (:443)
- **providers.docker**:
  - `exposedByDefault: false` — apenas containers com `traefik.enable=true` são expostos
  - `network: traefik_public`
- **certificatesResolvers.letsencrypt**:
  - email: `support@timero.me`
  - storage: `/etc/traefik/acme/acme.json`
  - httpChallenge via entrypoint `web`
- **api**: `dashboard: false` (desabilitado em produção)

## Integração das Apps

Cada app no VPS deve:
1. Conectar à rede `traefik_public` como externa
2. Adicionar labels Traefik no serviço

Exemplo mínimo:
```yaml
services:
  app:
    image: my-app
    networks:
      - traefik_public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`timero.me`)"
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls.certresolver=letsencrypt"
      - "traefik.http.services.my-app.loadbalancer.server.port=3000"

networks:
  traefik_public:
    external: true
```

## Constraints

- Porta 80 deve estar aberta para o HTTP Challenge do Let's Encrypt funcionar
- Apenas um processo pode ouvir nas portas 80/443 no host
- O volume `traefik-certificates` deve persistir entre restarts para evitar rate limit do ACME
