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
  - health endpoint je na `127.0.0.1:8420/health`
  - data ukládá do `./hermes/tdai-memory`

## Struktura

```text
.
├── .env.example
├── docker-compose.yml
├── docker/
│   └── hermes/
│       └── Dockerfile
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

## Ověření

### Hermes health

```bash
curl http://127.0.0.1:8642/health
```

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
- `curl http://127.0.0.1:8420/health`

### Gateway nenaběhne

Provider startuje gateway přímo uvnitř `hermes` kontejneru. Kritické věci jsou:

- Node runtime v image
- dostupný `pnpm`
- platný `MEMORY_TENCENTDB_GATEWAY_CMD`
- zapisovatelný `./hermes/tdai-memory`

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
