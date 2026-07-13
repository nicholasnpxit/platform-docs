# Runbook — npx-platform

Procedimentos operacionais do dia a dia. Última atualização: 2026-07-12.

## Subir/derrubar um stack

Cada stack é independente (Traefik, Portainer, cada cliente):

```bash
cd /opt/npx-platform/traefik      && docker compose up -d   # ou down
cd /opt/npx-platform/portainer    && docker compose up -d
cd /opt/npx-platform/clients/demo && docker compose up -d
```

`docker compose down -v` remove volumes também — só usar se realmente
quiser apagar os dados (bancos, dashboards salvos, etc.).

## Adicionar um novo cliente

1. `mkdir -p /opt/npx-platform/clients/<nome>`
2. Copiar a estrutura de `/opt/npx-platform/clients/demo/docker-compose.yml`
   como referência (rede `internal` própria para banco, containers web na
   rede `edge`).
3. Labels Traefik obrigatórias em cada container que precisa ser roteado:
   ```yaml
   labels:
     - "traefik.enable=true"
     - "traefik.docker.network=edge"
     - "traefik.http.routers.<nome>-<servico>.rule=Host(`<servico>.<nome>.npxit.com.br`)"
     - "traefik.http.routers.<nome>-<servico>.entrypoints=websecure"
     - "traefik.http.routers.<nome>-<servico>.tls=true"
     - "traefik.http.routers.<nome>-<servico>.tls.certresolver=letsencrypt"
     - "traefik.http.services.<nome>-<servico>.loadbalancer.server.port=<porta>"
   ```
4. Criar o DNS do(s) subdomínio(s) novo(s) apontando para o IP público
   correto **antes** de subir (senão o Traefik vai tentar e falhar a
   emissão do certificado repetidamente até funcionar — inofensivo, mas
   gera ruído no log).
5. **Atualizar `docs/ACCESS.md` e `docs/ARCHITECTURE.md` na mesma sessão**
   — não é opcional, ver `CLAUDE.md`.

## Trocar Let's Encrypt de staging para produção

Já foi feito em 2026-07-12 (ver `docs/STATE.md`) — os 4 hosts (`traefik`,
`portainer`, `zabbix.demo`, `grafana.demo`) estão em produção. Passo a passo
para repetir (ex: se algum host novo precisar do mesmo processo, validando
staging antes de virar para produção):

1. Configurar o certresolver com `acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`.
2. Subir/reiniciar o Traefik, confirmar emissão em staging via
   `openssl s_client -connect 127.0.0.1:443 -servername <host> | openssl x509 -noout -issuer -subject`
   — issuer deve conter `(STAGING)` e subject deve bater com o host.
3. Remover a flag `acme.caserver` do compose (produção é o padrão quando
   ausente).
4. **Zerar o `acme.json`** antes de reiniciar — senão o Traefik reaproveita
   os certificados de staging já válidos em vez de pedir novos à produção.
   O arquivo é `root:root 600` (criado pelo próprio Traefik), então zere via
   um container temporário como root:
   ```bash
   docker run --rm -v /opt/npx-platform/traefik/letsencrypt:/le alpine \
     sh -c "cp /le/acme.json /le/acme.json.staging-backup && : > /le/acme.json"
   ```
5. `docker restart traefik` e confirmar no log:
   `acmeCA=https://acme-v02.api.letsencrypt.org/directory` + `Register...`
   sem erro.
6. Disparar uma requisição para cada host (`curl -sk --resolve <host>:443:127.0.0.1 https://<host>/`)
   para forçar o pedido do certificado (Traefik pede sob demanda).
7. Validar com `curl` **sem `-k`** — se fechar sem erro de certificado, é
   produção real. Confirmar issuer sem `(STAGING)` via `openssl x509`.

## Verificar se um host está roteando corretamente

Sem depender de DNS público (útil para testar antes do DNS existir, ou a
partir da própria VM):

```bash
curl -sk -o /dev/null -w "%{http_code}\n" --resolve <host>:443:127.0.0.1 https://<host>/
```

Verificar qual certificado está sendo servido (self-signed de fallback vs
Let's Encrypt real):

```bash
echo | openssl s_client -connect 127.0.0.1:443 -servername <host> 2>/dev/null | openssl x509 -noout -issuer -subject
```

Se `issuer=CN = TRAEFIK DEFAULT CERT`, o Let's Encrypt ainda não emitiu para
aquele host (checar `docker logs traefik | grep -i acme`).

## Diagnosticar problema de emissão Let's Encrypt

```bash
docker logs traefik --tail 100 | grep -iE "acme|certificate|challenge"
```

Erros comuns:
- `DNS problem: NXDOMAIN` → o host não tem registro DNS público ainda.
- `Timeout during connect` / `Connection refused` → porta 80 não está
  alcançável de fora (checar FortiGate/firewall), Let's Encrypt precisa
  conectar na porta 80 (HTTP-01) para validar.
- Rate limit (`too many certificates`/`too many failed authorizations`) →
  esperar a janela do rate limit da Let's Encrypt passar; por isso staging
  é usado primeiro para não gastar tentativas de produção com configuração
  ainda não validada.

## Adicionar/trocar uma senha de algum serviço

1. Gerar senha forte: `openssl rand -base64 18 | tr -d '/+=' | cut -c1-20`
2. Aplicar no serviço (variável de ambiente do compose, ou via API/UI do
   serviço, dependendo do caso).
3. **Atualizar `docs/ACCESS.md` imediatamente**, na mesma sessão — regra
   permanente, ver `CLAUDE.md`.

## Containers e o que cada um faz (referência rápida)

| Container | Stack | Função |
|---|---|---|
| `traefik` | traefik | Reverse proxy / TLS |
| `docker-shim` | traefik | Proxy de rewrite do socket Docker (ver DECISIONS.md) |
| `portainer` | portainer | UI de administração Docker |
| `demo-zabbix-server` | clients/demo | Zabbix server |
| `demo-zabbix-web` | clients/demo | Zabbix web (nginx+php) |
| `demo-mysql` | clients/demo | Banco do Zabbix |
| `demo-grafana` | clients/demo | Grafana |
