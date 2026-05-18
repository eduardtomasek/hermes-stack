# Hermes Docker Stack

Připravený self-hosted stack pro `Hermes Agent` v Dockeru s persistentním `HERMES_HOME` a pamětí přes `TencentDB-Agent-Memory`.

Stack je navržený pro vzdálené používání přes `Hermes Desktop` proti HTTPS reverse proxy a zachovává lokální:

- personu v `SOUL.md`
- skills v `skills/`
- vlastní Hermes pluginy v `plugins/`

## Co je součástí

- `hermes`:
  - běží `gateway run`
  - vystavuje OpenAI-compatible API na `127.0.0.1:8642`
  - uvnitř image má Node 22, `pnpm` a `@tencentdb-agent-memory/memory-tencentdb`
  - používá `memory_tencentdb` jako Hermes memory provider
  - ukládá Hermes data i memory data do `./hermes`
- `memory_tencentdb` gateway:
  - nespouští se jako separátní compose service
  - provider ho startuje uvnitř `hermes` kontejneru přes `MEMORY_TENCENTDB_GATEWAY_CMD`
  - startuje se lazy až při první skutečné memory operaci nebo prvním requestu, který memory použije
  - health endpoint je na `127.0.0.1:8420/health`
  - data ukládá do `./hermes/tdai-memory`

## Architektura řešení

```text
client / Hermes Desktop / curl
            |
            v
reverse proxy (volitelné TLS)
            |
            v
Hermes API + Gateway
127.0.0.1:8642
            |
            +--> HERMES_HOME=/opt/data
            |    - config.yaml
            |    - SOUL.md
            |    - skills/
            |    - plugins/
            |    - tdai-memory/
            |
            +--> memory provider: memory_tencentdb
                    |
                    +--> lazy supervisor start
                    |
                    +--> internal gateway
                         127.0.0.1:8420
```

Runtime tok:

1. Docker spustí vlastní wrapper entrypoint `hermes-stack-entrypoint`.
2. Wrapper vytvoří symlinky na `memory_tencentdb` plugin do `/opt/data/plugins/...` a `/opt/hermes/plugins/memory/...`.
3. Wrapper předá řízení upstream `/opt/hermes/docker/entrypoint.sh gateway run`.
4. Hermes naběhne na `8642`.
5. `memory_tencentdb` se nespouští eager při bootu. Naběhne až když Hermes zpracuje request, který memory provider inicializuje.
6. Po prvním memory-aware requestu začne odpovídat i `127.0.0.1:8420/health`.

## Struktura

```text
.
├── .env.example
├── docker-compose.yml
├── docker/
│   └── hermes/
│       ├── Dockerfile
│       └── hermes-stack-entrypoint
├── hermes/
│   └── config.yaml.example
└── templates/
    ├── SOUL.md.example
    └── skills/
        └── example-skill/
            └── SKILL.md
```

Po prvním spuštění bude důležitý hlavně adresář `./hermes`, protože se mountuje jako `/opt/data` a funguje jako `HERMES_HOME`.

## Rychlý start

### 1. Připrav adresář

```bash
mkdir -p /opt/hermes
cd /opt/hermes
```

Na většině Linux serverů bude `/opt` zapisovatelný jen pro `root`. Praktický postup je:

```bash
sudo mkdir -p /opt/hermes
sudo chown -R "$USER":"$USER" /opt/hermes
cd /opt/hermes
```

Tím zajistíš, že:

- repozitář můžeš spravovat jako běžný uživatel
- `.env`, `hermes/`, `templates/` a další soubory nemusíš editovat přes `root`
- Docker bude do bind mountu zapisovat s očekávaným ownershipem

Pokud už adresář existuje a omylem ho vlastní `root`, oprav to:

```bash
sudo chown -R "$USER":"$USER" /opt/hermes
```

### 2. Zkopíruj obsah tohoto repozitáře

Do `/opt/hermes` dej:

- `docker-compose.yml`
- `.env.example` jako základ pro `.env`
- `docker/`
- `hermes/`
- `templates/`

### 3. Vytvoř `.env`

Zkopíruj šablonu a doplň hodnoty:

```bash
cp .env.example .env
```

Nejdůležitější proměnné:

```env
API_SERVER_KEY=sem_vloz_vygenerovany_klic
API_SERVER_CORS_ORIGINS=https://hermes.example.com
HERMES_UID=1000
HERMES_GID=1000
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
TELEGRAM_ALLOWED_USERS=123456789
OPENROUTER_API_KEY=sem_vloz_openrouter_api_klic
MEMORY_TENCENTDB_LLM_API_KEY=sem_vloz_llm_api_klic
MEMORY_TENCENTDB_LLM_BASE_URL=https://openrouter.ai/api/v1
MEMORY_TENCENTDB_LLM_MODEL=openai/gpt-4o-mini
```

Poznámky:

- `OPENROUTER_API_KEY` používá samotný Hermes agent pro hlavní chat/model provider.
- `MEMORY_TENCENTDB_LLM_API_KEY` je nutný pro L1/L2/L3 extrakci v Tencent memory gateway.
- `MEMORY_TENCENTDB_LLM_BASE_URL` může být libovolný OpenAI-compatible endpoint.
- `Hermes` model a `Tencent memory` model jsou oddělené konfigurace. Můžou mířit na stejný provider, ale nemusí.
- `API_SERVER_CORS_ORIGINS` je důležité hlavně pro browserové UI; pro `Hermes Desktop` typicky není potřeba.
- `TELEGRAM_ALLOWED_USERS` má být seznam číselných Telegram user ID oddělených čárkou, ne username.
- `HERMES_UID` a `HERMES_GID` mají odpovídat uživateli, který vlastní `/opt/hermes` na hostiteli. Ověříš je přes `id -u` a `id -g`.

Rychlé ověření:

```bash
id -u
id -g
ls -ld /opt/hermes
```

Pokud ownership nesedí, můžeš pak v bind mountu skončit se soubory, které vytvořil `root`, a další úpravy budou zbytečně nepříjemné.

### 4. Aktivuj `memory_tencentdb` v Hermes configu

```bash
cp hermes/config.yaml.example hermes/config.yaml
```

Šablona nastavuje:

```yaml
memory:
  provider: memory_tencentdb

model:
  provider: openrouter
  default: openai/gpt-4o-mini
```

To znamená:

- hlavní Hermes agent defaultně používá `OpenRouter`
- memory gateway defaultně taky používá `OpenRouter`, ale přes vlastní `MEMORY_TENCENTDB_*` proměnné

Další běžné varianty pro hlavní Hermes model:

```yaml
# OpenAI
model:
  provider: openai
  default: gpt-4o-mini
```

```yaml
# Custom OpenAI-compatible endpoint, třeba Ollama nebo vLLM
model:
  provider: custom
  default: qwen2.5-coder:14b
  base_url: http://host.docker.internal:11434/v1
  api_key: ollama
```

Pro custom/local endpoint uvnitř Dockeru obvykle použij `host.docker.internal`, ne `localhost`, protože `localhost` by v kontejneru ukazoval na samotný kontejner.

### 5. Volitelně přidej personu

```bash
cp templates/SOUL.md.example hermes/SOUL.md
```

### 6. Spusť stack

```bash
docker compose up -d --build
```

### 7. Sleduj logy

```bash
docker compose logs -f hermes
```

### 8. Udělej první warmup request

`memory_tencentdb` se spouští lazy. Samotný start kontejneru nestačí k tomu, aby už běžel proces na `8420`.

```bash
export API_SERVER_KEY='sem-vloz-stejny-klic-z-.env'
curl -sS http://127.0.0.1:8642/v1/chat/completions \
  -H "Authorization: Bearer $API_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek/deepseek-v3.2",
    "messages": [
      {"role":"user","content":"Say only ok."}
    ]
  }'
```

Po tomhle requestu už obvykle běží i interní Tencent gateway na `8420`.

## Jak zprovoznit Telegram

Stack už běží s `gateway run`, takže pro Telegram není potřeba další compose service.

### 1. Vytvoř Telegram bota

V Telegramu napiš `@BotFather`, použij `/newbot` a ulož si token.

Výsledek vypadá zhruba takto:

```text
123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

### 2. Zjisti svoje Telegram user ID

Použij třeba `@userinfobot` nebo `@get_id_bot`.

Potřebuješ číselné ID, například:

```text
123456789
```

### 3. Doplň `.env`

```env
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
TELEGRAM_ALLOWED_USERS=123456789
```

Pro víc lidí:

```env
TELEGRAM_ALLOWED_USERS=123456789,987654321
```

Volitelně můžeš nastavit i cílový chat pro cron výstupy:

```env
TELEGRAM_HOME_CHANNEL=-1001234567890
TELEGRAM_HOME_CHANNEL_NAME=My Notes
```

### 4. Restartuj stack

```bash
docker compose up -d
docker compose logs -f hermes
```

### 5. Napiš botovi

Pošli botovi v Telegramu zprávu typu:

```text
hello
```

Pokud chceš, aby cron nebo proactive zprávy chodily do konkrétního chatu, pošli do toho chatu `/sethome`.

### Polling vs webhook

Default je long polling. To je nejjednodušší varianta pro VPS nebo domácí server.

Pokud chceš webhook režim, přidej:

```env
TELEGRAM_WEBHOOK_URL=https://hermes.example.com/telegram
```

Webhook dává smysl hlavně na platformách, kde se instance uspává a probouzí přes příchozí HTTP provoz.

### Ověření

Když je Telegram správně nastavený:

- warning o chybějícím allowlistu zmizí
- bot odpoví na zprávu v Telegramu
- v logu uvidíš příchozí turny gatewaye

## Ověření

### Hermes health

```bash
curl http://127.0.0.1:8642/health
```

Tohle je hlavní healthcheck pro Docker i operativní monitoring.

### Hermes autorizace

```bash
export API_SERVER_KEY='sem-vloz-stejny-klic-z-.env'
curl -H "Authorization: Bearer $API_SERVER_KEY" \
  http://127.0.0.1:8642/v1/models
```

### Tencent memory gateway health

```bash
curl http://127.0.0.1:8420/health
```

Důležité:

- `8420` není vhodný jako cold-start healthcheck kontejneru
- při čerstvém startu může vracet `connection refused`, i když je stack v pořádku
- použij ho až po prvním requestu, který aktivuje memory provider

## Přístup zvenku přes Hermes Desktop

Bezpečný pattern:

1. Hermes nech publikovaný jen na `127.0.0.1:8642`
2. před něj dej reverse proxy s TLS
3. v `Hermes Desktop` nastav:
   - Base URL: `https://hermes.tvoje-domena.cz`
   - API key: hodnota `API_SERVER_KEY`

### Minimal Caddy example

```caddyfile
hermes.tvoje-domena.cz {
  reverse_proxy 127.0.0.1:8642
}
```

### Minimal Nginx example

```nginx
server {
    listen 443 ssl http2;
    server_name hermes.tvoje-domena.cz;

    ssl_certificate /etc/letsencrypt/live/hermes.tvoje-domena.cz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hermes.tvoje-domena.cz/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8642;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Kam dát osobnost agenta

Primární persona patří do:

```text
./hermes/SOUL.md
```

Uvnitř kontejneru je to:

```text
/opt/data/SOUL.md
```

## Jak přidávat skills

Lokální skills patří do:

```text
./hermes/skills/<kategorie>/<nazev-skillu>/SKILL.md
```

Rychlý start:

```bash
mkdir -p hermes/skills/local/example-skill
cp templates/skills/example-skill/SKILL.md hermes/skills/local/example-skill/SKILL.md
```

Pokud chceš sdílet skills mimo `HERMES_HOME`, přidej další read-only cesty do `./hermes/config.yaml`:

```yaml
skills:
  external_dirs:
    - /opt/data/external-skills
```

## Jak přidávat vlastní pluginy

Pro vlastní Hermes pluginy používej:

```text
./hermes/plugins/<plugin-name>/
```

`memory_tencentdb` provider se vytváří při startu automaticky jako symlink do `./hermes/plugins/memory_tencentdb`, takže tenhle adresář neupravuj ručně.

Kvůli aktuálnímu chování upstream Hermes se stejný plugin při startu linkuje i do interní cesty `/opt/hermes/plugins/memory/memory_tencentdb`, aby ho provider discovery spolehlivě našel.

## Aktualizace

Po update stacku:

```bash
docker compose build --no-cache hermes
docker compose up -d
```

Tím se stáhne i nejnovější `@tencentdb-agent-memory/memory-tencentdb@latest`, protože je součástí buildu image.

## Troubleshooting

### `hermes` naběhne, ale paměť nefunguje

Zkontroluj:

- `hermes/config.yaml` opravdu existuje, ne jen `config.yaml.example`
- `memory.provider` je `memory_tencentdb`
- `model.provider` a `model.default` v `hermes/config.yaml` odpovídají tomu, co chceš pro hlavní Hermes model
- provider API key pro hlavní Hermes model je nastavený v `.env` (`OPENROUTER_API_KEY`, `OPENAI_API_KEY`, atd.)
- `MEMORY_TENCENTDB_LLM_API_KEY` je nastavený v `.env`
- `docker compose logs hermes`
- proběhl aspoň jeden request na `8642`, který memory provider aktivuje
- teprve potom `curl http://127.0.0.1:8420/health`

### Gateway nenaběhne

Provider startuje gateway přímo uvnitř `hermes` kontejneru. Kritické věci jsou:

- Node runtime v image
- dostupný `tsx` v `./node_modules/.bin/tsx`
- platný `MEMORY_TENCENTDB_GATEWAY_CMD`
- zapisovatelný `./hermes/tdai-memory`
- korektní symlink pluginu do `/opt/hermes/plugins/memory/memory_tencentdb`

### `8420/health` vrací `connection refused` hned po startu

To je očekávané chování, pokud ještě neproběhl první memory-aware request.

Postup:

1. ověř `curl http://127.0.0.1:8642/health`
2. pošli první request na `POST /v1/chat/completions`
3. teprve potom testuj `curl http://127.0.0.1:8420/health`

### Proč ten stack nepoužívá `pnpm exec tsx`

Balíček `@tencentdb-agent-memory/memory-tencentdb` se v image instaluje přes `npm`.

Při runtime se ukázalo, že `pnpm exec tsx ...` se pokoušel přeuspořádat `node_modules` a končil na `ERR_PNPM_IGNORED_BUILDS`.

Proto stack spouští gateway přímo přes:

```bash
./node_modules/.bin/tsx node_modules/@tencentdb-agent-memory/memory-tencentdb/src/gateway/server.ts
```

Tohle bylo ověřené jako funkční.

### Jak opravdu poznat, že memory funguje

Nejspolehlivější test není samotný health endpoint, ale round-trip přes Hermes API:

1. pošli request typu "Remember that my favorite editor is Helix"
2. pošli druhý request "What is my favorite editor?"
3. pokud odpoví `Helix`, funguje celý chain:
   Hermes API -> memory provider -> Tencent gateway -> persistent storage

### Chci úplně čistý rebuild

```bash
docker compose down
docker compose build --no-cache hermes
docker compose up -d
```

## Reference

- Hermes Docker docs: https://hermes-agent.nousresearch.com/docs/user-guide/docker/
- Hermes API server: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/api-server.md
- Hermes personality (`SOUL.md`): https://hermes-agent.nousresearch.com/docs/user-guide/features/personality/
- Hermes skills: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills
- Hermes plugins: https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins
- TencentDB-Agent-Memory repo: https://github.com/Tencent/TencentDB-Agent-Memory
- Hermes provider docs v upstreamu: https://github.com/Tencent/TencentDB-Agent-Memory/tree/main/hermes-plugin/memory/memory_tencentdb
