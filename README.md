# Traefik Router

Router centralizado para o VPS. Gerencia todo o tráfego HTTP/HTTPS via portas 80 e 443, com SSL automático via Let's Encrypt e rate limiting global por IP.

## Arquitetura

```
Internet
   │
   ├─ :80  → redirect HTTPS
   └─ :443 → Traefik → containers via rede traefik_public
```

Cada aplicação roda em seu próprio `docker-compose.yml` e se integra ao Traefik através de **labels** e da **rede compartilhada** `traefik_public`.

## Subindo o Router

```bash
docker compose up -d
```

O router deve estar **sempre rodando antes** das outras aplicações, pois ele cria a rede `traefik_public`.

## Integrando uma Aplicação

### 1. Conectar à rede compartilhada

No `docker-compose.yml` da sua app, declare a rede `traefik_public` como externa:

```yaml
networks:
  traefik_public:
    external: true
```

### 2. Adicionar labels e rede ao serviço

```yaml
services:
  app:
    image: minha-app
    networks:
      - traefik_public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.minha-app.rule=Host(`meudominio.com`)"
      - "traefik.http.routers.minha-app.entrypoints=websecure"
      - "traefik.http.routers.minha-app.tls.certresolver=letsencrypt"
      - "traefik.http.services.minha-app.loadbalancer.server.port=3000"
```

> **Não exponha portas** (`ports:`) nas suas aplicações. O Traefik acessa os containers diretamente pela rede interna.

### 3. Exemplo completo

```yaml
services:
  app:
    image: minha-app:latest
    restart: unless-stopped
    networks:
      - traefik_public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.minha-app.rule=Host(`timero.me`)"
      - "traefik.http.routers.minha-app.entrypoints=websecure"
      - "traefik.http.routers.minha-app.tls.certresolver=letsencrypt"
      - "traefik.http.services.minha-app.loadbalancer.server.port=3000"

networks:
  traefik_public:
    external: true
```

## Labels de Referência

| Label | Descrição |
|---|---|
| `traefik.enable=true` | Habilita o Traefik para este container |
| `traefik.http.routers.<nome>.rule` | Regra de roteamento (ex: `Host(\`dominio.com\`)`) |
| `traefik.http.routers.<nome>.entrypoints` | Entrypoint — use sempre `websecure` |
| `traefik.http.routers.<nome>.tls.certresolver` | Resolver SSL — use sempre `letsencrypt` |
| `traefik.http.services.<nome>.loadbalancer.server.port` | Porta interna do container |

> O `<nome>` nas labels deve ser único por aplicação no VPS.

## Múltiplos Domínios no Mesmo Serviço

```yaml
labels:
  - "traefik.http.routers.minha-app.rule=Host(`timero.me`) || Host(`www.timero.me`)"
```

## Subdomínios

```yaml
labels:
  - "traefik.http.routers.api.rule=Host(`api.timero.me`)"
  - "traefik.http.routers.api.entrypoints=websecure"
  - "traefik.http.routers.api.tls.certresolver=letsencrypt"
  - "traefik.http.services.api.loadbalancer.server.port=8080"
```

## Rate Limiting

Um rate limit de **100 req/s por IP** (burst de 50) é aplicado automaticamente a todas as rotas. Nenhuma configuração adicional é necessária nas apps.

Para ajustar os limites, edite `dynamic.yml` — o Traefik recarrega automaticamente sem restart.

## Pré-requisitos no VPS

- Portas **80** e **443** abertas no firewall
- DNS dos domínios apontando para o IP do VPS antes de subir as apps (necessário para o Let's Encrypt emitir os certificados)

## Troubleshooting

**502 Bad Gateway** — o container da app não está acessível na rede `traefik_public`. Verifique se a rede está declarada no serviço e se o container está rodando.

**Certificado não emitido** — confirme que o DNS do domínio aponta para o VPS e que a porta 80 está acessível publicamente (usada pelo HTTP Challenge do Let's Encrypt).

**429 Too Many Requests** — o rate limit foi atingido. Ajuste os valores em `dynamic.yml` se necessário.
