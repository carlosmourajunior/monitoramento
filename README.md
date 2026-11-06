# Monitoramento

Um painel só para todos os servidores e sistemas: uso de CPU, memória e disco, containers, uso das APIs, bloqueios do CrowdSec, logins por SSH e alertas no Telegram.

```
 servidor olt ───────┐                          ┌─ Grafana  (monitor.rlcsolucoes.com.br)
   agente Alloy      │   HTTPS + senha          │
 servidor debian ────┼──▶ túnel Cloudflare ──▶ hub (no olt)
   agente Alloy      │   ingest.rlcsolucoes...  │  Prometheus (métricas, 30 dias)
 (próximos...) ──────┘                          │  Loki (logs, 30 dias)
                                                └─ Uptime Kuma (status.rlcsolucoes.com.br)
```

- **`hub/`** roda uma vez, no olt: Prometheus, Loki, Grafana, Uptime Kuma, um gateway com senha para os agentes e o `cloudflared`. Nenhuma porta é aberta.
- **`agente/`** roda em cada servidor, inclusive no olt: um container do Grafana Alloy que coleta tudo e envia para o hub. Funciona atrás de NAT, porque é o agente que envia.

## O que aparece no painel "Visão geral"

| Seção | Conteúdo |
|---|---|
| Servidores | se está mandando dados, há quanto tempo está ligado, disco, CPU, memória, rede, IPs bloqueados pelo CrowdSec |
| Containers | CPU e memória por container, containers que reiniciaram nas últimas 24 h |
| Uso das APIs e sites | requisições por site, respostas por status (2xx/4xx/5xx), endpoints mais chamados (IDs viram `:id`), quem mais usa a API do ISP (por IP), erros 5xx por endpoint |
| Segurança | alertas das aplicações (logins errados, login de admin, bloqueios por IP) e logins por SSH |

Os dados de API vêm dos logs de acesso do nginx (frontends) e do Caddy (ISP). Nada precisa mudar nos sistemas.

## Alertas (Telegram)

| Alerta | Quando |
|---|---|
| Servidor parou de mandar dados | 3 min sem métricas (desligado, sem rede, agente parado) |
| Container sumiu | um container que rodava na última hora não existe mais. Foi o que aconteceu com o ControleFinanceiro no upgrade do Debian |
| Disco quase cheio | > 85% por 10 min |
| Memória alta | > 90% por 10 min |
| CPU alta | > 90% por 15 min |
| Muitos erros 5xx | > 20 em 5 min num site |
| CrowdSec sem dados | o CrowdSec parou de responder |

As regras ficam em `hub/grafana/provisioning/alerting/regras.yml`. Os avisos de IP bloqueado continuam vindo direto do CrowdSec.

## Instalação

### 1. Túnel Cloudflare (painel Zero Trust)

Em **Networks → Tunnels → Create a tunnel**, crie o túnel `monitoramento` (tipo Cloudflared) e copie o token. Em **Public hostnames**, adicione:

| Hostname | Service |
|---|---|
| `monitor.rlcsolucoes.com.br` | `http://grafana:3000` |
| `status.rlcsolucoes.com.br` | `http://uptime-kuma:3001` |
| `ingest.rlcsolucoes.com.br` | `http://gateway:80` |

### 2. Hub (no olt)

```bash
cd /root && git clone https://github.com/carlosmourajunior/monitoramento.git
cd monitoramento/hub && cp .env.example .env && chmod 600 .env
docker run --rm caddy:2.9.1-alpine caddy hash-password --plaintext 'UMA-SENHA-FORTE-PARA-OS-AGENTES'
nano .env     # GRAFANA_ADMIN_PASSWORD, TELEGRAM_BOT_TOKEN, INGEST_PASSWORD_HASH (entre aspas simples), CLOUDFLARE_TUNNEL_TOKEN
docker compose up -d
docker compose ps
```

Abra `https://monitor.rlcsolucoes.com.br` e entre com o usuário admin e a senha do `.env`.

### 3. Agente (em cada servidor, inclusive no olt)

```bash
cd /root && git clone https://github.com/carlosmourajunior/monitoramento.git   # no olt já existe
cd monitoramento/agente && cp .env.example .env && chmod 600 .env
nano .env     # SERVIDOR=olt (ou debian), INGEST_PASSWORD=a senha em texto do passo 2
docker compose up -d
docker compose logs --tail 20 alloy    # não deve aparecer 401 nem "connection refused"
```

Em 1 ou 2 minutos, o servidor aparece no painel.

### 4. Uptime Kuma

Abra `https://status.rlcsolucoes.com.br` e crie o usuário admin no primeiro acesso. Depois:

1. **Settings → Notifications → Setup Notification → Telegram**: cole o token do bot e o chat id `1471597086`, marque *Default enabled* e *Apply on all existing monitors*.
2. **Add New Monitor → HTTP(s)** para cada site: `https://interno.via01.com.br/api/health/`, `https://financeiro.rlcsolucoes.com.br`, os do FisioPilates, do rlc, do financas, do sponsorview e do meuemprego. Use intervalo de 60 s.

O Uptime Kuma testa os sites pelo lado de fora. Ele avisa quando o site cai, mesmo que o servidor esteja de pé.

## Manutenção

- **Atualizar:** `git pull && docker compose up -d` na pasta `hub/` ou `agente/`. As versões das imagens estão fixadas nos composes.
- **Retenção:** 30 dias de métricas (`RETENCAO_METRICAS`) e 30 dias de logs (`hub/loki/loki.yml`).
- **Painel:** o "Visão geral" vem de `hub/grafana/dashboards/visao-geral.json` e não é editável pela interface. Para criar um painel novo, use *Save as*. Para mudar o oficial, edite o JSON e rode `docker compose restart grafana`.
- **CrowdSec:** o agente lê as métricas em `127.0.0.1:6060`, a porta padrão do CrowdSec. Sem o CrowdSec instalado, o painel só mostra "No data".
