# Arquitetura — Agente IA Multi-Loja (Grupo Ray)

> 11 lojas, 1 agente, GHL + WhatsApp API Oficial + FastAPI/LangGraph
> **Visao SaaS:** Este projeto valida o produto replicavel da Triadeflow

## Visao Geral

```
┌─────────────────────────────────────────────────────────┐
│                    11 LOJAS (WhatsApp)                   │
│  Loja 1  Loja 2  Loja 3  ...  Loja 10  Loja 11        │
│    │       │       │              │        │            │
└────┼───────┼───────┼──────────────┼────────┼────────────┘
     └───────┴───────┴──────┬───────┴────────┘
                            ▼
              ┌──────────────────────────┐
              │   GHL (CRM Unificado)    │
              │   LC Phone — API Oficial │
              │                          │
              │  Workflow: Customer Reply │
              │    IF tag "bot_ativo"    │
              │      → Webhook POST      │
              └────────────┬─────────────┘
                           ▼
     ┌──────────────────────────────────────────┐
     │   HETZNER VPS (4vCPU, 8GB RAM)          │
     │   Docker Compose                         │
     │                                          │
     │  ┌──────────┐  ┌───────────────────────┐│
     │  │  Nginx   │→ │  FastAPI + LangGraph  ││
     │  │ (proxy)  │  │  4 workers uvicorn    ││
     │  └──────────┘  └───────────┬───────────┘│
     │                            │             │
     │  ┌──────────┐  ┌──────────┴──────────┐ │
     │  │  Redis   │  │  PostgreSQL         │ │
     │  │ (cache/  │  │  (historico +       │ │
     │  │  queue)  │  │   analytics)        │ │
     │  └──────────┘  └─────────────────────┘ │
     └──────────────────────────────────────────┘
```

## Stack de Producao

| Camada | Tech | Detalhes |
|--------|------|----------|
| **Servidor** | Hetzner VPS | 4vCPU, 8GB RAM, SSD — ~€15/mes |
| **Orquestracao** | Docker Compose | Nginx + App + Redis + PostgreSQL |
| **CRM** | GoHighLevel | 1 Location ou Agency (11 lojas como sub) |
| **WhatsApp** | GHL LC Phone (API Oficial) | 1 numero por loja, CRM unificado |
| **Trigger** | GHL Workflow | "Customer Replied" → Webhook direto |
| **App** | FastAPI + LangGraph | 4 workers uvicorn, async, ReAct agent |
| **Cache/Queue** | Redis | Cache FAQ, fila de mensagens, rate limiting |
| **Banco** | PostgreSQL | Historico conversas, analytics, metricas |
| **Motor IA** | GPT-4.1-mini | Com routing por complexidade + cache |
| **Proxy** | Nginx | SSL (Let's Encrypt), rate limit, gzip |

## Otimizacao de Latencia LLM

```
Mensagem recebida
    │
    ▼
┌─ Redis Cache ─────────────────────┐
│ FAQ match? → Resposta instantanea │  (~50ms)
└───────────────┬───────────────────┘
                │ (cache miss)
                ▼
┌─ Model Routing ───────────────────┐
│ Simples (saudacao, FAQ)           │
│   → GPT-4.1-nano (~200ms)        │
│ Medio (info produto, horario)     │
│   → GPT-4.1-mini (~500ms)        │
│ Complexo (reclamacao, negociacao) │
│   → GPT-4.1-mini full (~800ms)   │
└───────────────────────────────────┘
                │
                ▼
┌─ Streaming Response ─────────────┐
│ Resposta enviada em chunks       │
│ via GHL Conversations API        │
└───────────────────────────────────┘
```

### Tecnicas de otimizacao
- **Prompt caching:** Prompts base/marca cacheados (nao reprocessa a cada msg)
- **Redis FAQ cache:** Respostas pra perguntas frequentes (horario, endereco, cardapio)
- **Model routing:** Classifica complexidade antes de chamar LLM
- **Connection pooling:** Pool de conexoes HTTP pra GHL API
- **Async everywhere:** FastAPI async + httpx async client

## GHL — Estrutura

### Opcao A: 1 Location (recomendado pra comecar)
```
Location: Grupo Ray
  ├── Pipeline Sushi da Hora (5 unidades)
  ├── Pipeline Mundo Ray (atacado B2B)
  ├── Pipeline [Outras marcas]
  ├── Tags por loja: loja_barra, loja_parquelandia, loja_mundoray...
  ├── Custom Fields: loja_origem, tipo_cliente
  └── 11 numeros WhatsApp (LC Phone)
```

### Opcao B: Agency (escala futura / SaaS)
```
Agency: Grupo Ray
  ├── Sub-Account: Sushi da Hora
  ├── Sub-Account: Mundo Ray
  └── Sub-Account: [Outras marcas]
```

## GHL Workflow (por loja)

```yaml
Trigger: Customer Replied
Filters:
  - Channel: WhatsApp
  - Contact Tag contains: "bot_ativo"
Actions:
  1. Webhook POST:
     URL: https://{servidor}/webhook/ghl
     Body:
       contactId: {{contact.id}}
       contactName: {{contact.name}}
       phone: {{contact.phone}}
       message: {{message.body}}
       conversationId: {{conversation.id}}
       locationId: {{location.id}}
       storeId: "loja_xxx"  # tag ou custom field
```

## Agente FastAPI — Multi-Tenant

### Endpoint unico
```python
@app.post("/webhook/ghl")
async def webhook_ghl(payload: GHLWebhook):
    store = identify_store(payload.phone, payload.store_id, payload.location_id)

    # Cache check (Redis)
    cached = await redis.get(f"faq:{store['id']}:{hash(payload.message)}")
    if cached:
        await ghl_send_message(payload.conversation_id, cached)
        return

    # Model routing
    complexity = classify_complexity(payload.message)
    model = select_model(complexity)

    prompt = load_store_prompt(store)
    agent = create_react_agent(model, tools, prompt)
    response = await run_agent(agent, payload.message, payload.contact_id)

    # Persist to PostgreSQL
    await save_conversation(payload.contact_id, store, payload.message, response)

    await ghl_send_message(payload.conversation_id, response)
```

### Identificacao de loja
```python
# Cada numero de WhatsApp mapeia pra uma loja
STORE_MAP = {
    "558599210061": {"id": "barra", "brand": "sushi", "name": "Sushi da Hora - Barra do Ceara"},
    "558598405232": {"id": "parquelandia", "brand": "sushi", "name": "Sushi da Hora - Parquelandia"},
    "558598554849": {"id": "maraponga", "brand": "sushi", "name": "Sushi da Hora - Maraponga"},
    "558598661202": {"id": "maracanau", "brand": "sushi", "name": "Sushi da Hora - Maracanau"},
    "558598630747": {"id": "messejana", "brand": "sushi", "name": "Sushi da Hora - Messejana"},
    "55XXXXXXXXXX": {"id": "mundoray_1", "brand": "mundoray", "name": "Mundo Ray - Atacado"},
    # ... demais lojas
}
```

### Prompts por marca/loja
```
src/
├── prompts/
│   ├── base.py              # Regras gerais (tom, limites, transferencia)
│   ├── sushi_da_hora.py     # Prompt Sushi (cardapio, unidades, promos)
│   ├── mundo_ray.py         # Prompt Mundo Ray (atacado, Bling, lojistas)
│   └── store_overrides/
│       ├── barra.py         # Override especifico (horario, endereco)
│       ├── parquelandia.py
│       └── ...
```

### Composicao do prompt
```python
def load_store_prompt(store):
    base = load_base_prompt()
    brand = load_brand_prompt(store["brand"])  # sushi ou mundoray
    override = load_store_override(store["id"])  # especifico da loja
    return f"{base}\n\n{brand}\n\n{override}"
```

## Tools (compartilhadas + especificas)

### Tools compartilhadas (todas as lojas)
| Tool | Descricao |
|------|-----------|
| `enviar_mensagem` | Responde via GHL Conversations API |
| `buscar_contato` | Busca no GHL CRM |
| `adicionar_tags` | Tags no contato |
| `preencher_campos` | Custom fields |
| `adicionar_nota` | Notas internas |
| `transferir_humano` | Remove tag bot + notifica vendedor |
| `identificar_loja` | Mapeia bairro/regiao → loja mais proxima |

### Tools Sushi da Hora
| Tool | Descricao |
|------|-----------|
| `consultar_promo_dia` | Promo por dia da semana |
| `consultar_cardapio` | Link cardapio da unidade |
| `consultar_status_pedido` | Status via Agilizone (futuro) |

### Tools Mundo Ray
| Tool | Descricao |
|------|-----------|
| `consultar_catalogo` | Produtos/colecoes disponiveis |
| `consultar_pedido_bling` | Status pedido via Bling API |
| `consultar_estoque` | Disponibilidade via Bling |
| `calcular_frete` | Estimativa por regiao |

## Banco de Dados (PostgreSQL)

### Schema principal
```sql
-- Historico de conversas
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id VARCHAR(50) NOT NULL,
    store_id VARCHAR(50) NOT NULL,
    brand VARCHAR(30) NOT NULL,
    message_in TEXT NOT NULL,
    message_out TEXT NOT NULL,
    model_used VARCHAR(30),
    tokens_in INT,
    tokens_out INT,
    latency_ms INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Metricas por loja
CREATE TABLE store_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id VARCHAR(50) NOT NULL,
    date DATE NOT NULL,
    total_messages INT DEFAULT 0,
    total_transfers INT DEFAULT 0,
    avg_latency_ms INT,
    total_tokens INT DEFAULT 0,
    cost_usd DECIMAL(10,4),
    UNIQUE(store_id, date)
);

-- Cache FAQ
CREATE TABLE faq_cache (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id VARCHAR(50) NOT NULL,
    question_hash VARCHAR(64) NOT NULL,
    answer TEXT NOT NULL,
    hit_count INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    UNIQUE(store_id, question_hash)
);

-- Indice pra queries de analytics
CREATE INDEX idx_conversations_store_date ON conversations(store_id, created_at);
CREATE INDEX idx_metrics_store_date ON store_metrics(store_id, date);
```

## Estrutura do Projeto

```
grupo-ray-bot/
├── docker-compose.yml          # Nginx + App + Redis + PostgreSQL
├── nginx/
│   └── nginx.conf              # Proxy reverso + SSL + rate limit
├── src/
│   ├── main.py                 # FastAPI app (4 workers uvicorn)
│   ├── config.py               # Settings (env vars, Pydantic BaseSettings)
│   ├── database.py             # PostgreSQL connection pool (asyncpg)
│   ├── cache.py                # Redis client (aioredis)
│   ├── agent/
│   │   ├── graph.py            # LangGraph ReAct agent (multi-tenant)
│   │   ├── router.py           # Classificador complexidade → modelo
│   │   ├── tools/
│   │   │   ├── shared.py       # Tools compartilhadas (GHL, mensagem)
│   │   │   ├── sushi.py        # Tools Sushi da Hora
│   │   │   └── mundoray.py     # Tools Mundo Ray
│   │   ├── prompts/
│   │   │   ├── base.py         # Prompt base
│   │   │   ├── sushi_da_hora.py # Prompt marca Sushi
│   │   │   ├── mundo_ray.py    # Prompt marca Mundo Ray
│   │   │   └── stores.py       # Overrides por loja
│   │   ├── store_map.py        # Mapeamento numero → loja
│   │   └── nodes.py            # Preprocessamento (audio, imagem)
│   ├── integrations/
│   │   ├── ghl.py              # GHL CRM + Conversations API
│   │   └── bling.py            # Bling ERP API (Mundo Ray)
│   └── models/
│       └── schemas.py          # Pydantic models
├── requirements.txt
├── Dockerfile
└── .env
```

## Docker Compose

```yaml
version: "3.8"
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - app
    restart: always

  app:
    build: .
    command: uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 4
    env_file: .env
    depends_on:
      - redis
      - postgres
    restart: always

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: always

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: grupo_ray
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
    restart: always

volumes:
  redis_data:
  pg_data:
```

## Env Vars (Hetzner VPS)

```env
# IA
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4.1-mini
OPENAI_MODEL_NANO=gpt-4.1-nano

# GHL
GHL_API_TOKEN=pit-...
GHL_LOCATION_ID=...

# Bling (Mundo Ray)
BLING_API_TOKEN=...

# Database
DB_HOST=postgres
DB_PORT=5432
DB_USER=grupo_ray
DB_PASSWORD=...
DB_NAME=grupo_ray
DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}

# Redis
REDIS_URL=redis://redis:6379/0

# Config
PORT=8000
ENV=production
LOG_LEVEL=info
```

## Fluxo de Transferencia Humano

```
1. Cliente pede humano (ou agente detecta necessidade)
2. Agente:
   - Remove tag "bot_ativo" do contato (GHL API)
   - Adiciona tag "atendimento_humano"
   - Adiciona nota: "Transferido: [motivo]"
   - Envia msg: "Vou te transferir para um atendente..."
3. GHL Workflow para de enviar pro agente (sem tag "bot_ativo")
4. Vendedor ve a notificacao no GHL e assume
5. Quando vendedor termina, adiciona tag "bot_ativo" de volta (manual ou automatico)
```

## Custo Estimado (11 lojas)

| Item | Custo/mes |
|------|-----------|
| Hetzner VPS (4vCPU/8GB) | ~R$80 (~€15) |
| OpenAI GPT-4.1-mini | ~R$50-150 (depende do volume) |
| GHL (plataforma) | Ja incluso no plano do cliente |
| WhatsApp conversas | ~R$0.15/conversa service |
| Dominio + SSL | Gratuito (Let's Encrypt) |
| **Total estimado** | **~R$150-300/mes** pra 11 lojas |

## Escalabilidade

- **1 VPS Hetzner** aguenta todas as 11 lojas (4 workers async, nao-blocking)
- Se volume crescer: upgrade VPS (8vCPU/16GB = ~€30/mes) ou adicionar segundo VPS com load balancer
- Adicionar loja nova = adicionar entrada no STORE_MAP + prompt
- Adicionar marca nova = novo arquivo de prompt + tools especificas
- PostgreSQL aguenta milhoes de registros com indices adequados
- Redis cache reduz chamadas LLM em ~40-60% (FAQ repetidas)

## Visao SaaS (Pos-Validacao)

### O que o Grupo Ray ja valida
- Arquitetura multi-tenant (STORE_MAP, prompts por marca, tools modulares)
- Dois segmentos diferentes (food delivery + atacado B2B) = versatilidade
- 11 lojas + 80K clientes = escala real
- Custo por loja baixo (~R$15-25/loja/mes de infra)

### O que precisa adicionar pro SaaS
- Dashboard admin (criar lojas, editar prompts, ver metricas)
- Onboarding self-service (wizard de configuracao)
- Billing (Stripe/Asaas — cobranca automatica)
- API de configuracao (CRUD lojas/prompts via API)
- Multi-GHL (suportar N locations de N clientes)
- Isolamento de dados por cliente (schema separation ou row-level)

### Modelo de Negocio Projetado
- Starter: R$297/mes (1 loja)
- Pro: R$697/mes (5 lojas)
- Business: R$1.497/mes (ilimitado)
- Setup: R$2.000-5.000

### Roadmap SaaS
1. **Mar/Abr 2026:** Validar com Grupo Ray (11 lojas) — FASE ATUAL
2. **Mai 2026:** Dashboard admin + 2-3 clientes beta
3. **Jun/Jul 2026:** SaaS self-service + billing

## Roadmap de Implementacao (Grupo Ray)

### Semana 1: Base
- [ ] Criar repo `grupo-ray-bot`
- [ ] Setup Hetzner VPS + Docker Compose
- [ ] FastAPI + estrutura multi-tenant
- [ ] PostgreSQL schema + Redis config
- [ ] GHL integration (Conversations API send/receive)
- [ ] Webhook endpoint + store identification
- [ ] Prompt base + prompt Sushi da Hora (migrar do Hiro)

### Semana 2: Sushi da Hora (5 lojas)
- [ ] Migrar tools do Hiro (adaptar pra GHL messaging)
- [ ] Prompts por unidade (5 overrides)
- [ ] GHL Workflows (5 triggers)
- [ ] Model routing + Redis cache
- [ ] Testar nas 5 unidades

### Semana 3: Mundo Ray
- [ ] Prompt Mundo Ray (atacado B2B)
- [ ] Tools Bling (pedido, estoque, catalogo)
- [ ] GHL Workflow Mundo Ray
- [ ] Testar

### Semana 4: Demais lojas + Go-Live
- [ ] Adicionar lojas restantes
- [ ] Testes end-to-end todas as 11
- [ ] Dashboard metricas (PostgreSQL queries)
- [ ] Go-live gradual
- [ ] Monitoramento (logs + alertas)

## Decisoes Tecnicas

| Decisao | Escolha | Motivo |
|---------|---------|--------|
| WhatsApp | API Oficial via GHL LC Phone | CRM unificado, vendedor ve tudo |
| Webhook | GHL Workflow → Agente direto | Sem n8n, menos latencia, menos ponto de falha |
| Multi-tenant | 1 instancia, STORE_MAP | Simples, barato, escala |
| Modelo IA | GPT-4.1-mini (com routing) | Cost-efficient + nano pra simples |
| Deploy | Hetzner VPS + Docker Compose | Mais robusto, menor latencia, custo fixo |
| Banco | PostgreSQL | Historico, analytics, metricas — dados sao o ativo |
| Cache | Redis | FAQ cache, rate limiting, filas |
| Proxy | Nginx | SSL, rate limit, gzip, robustez |
| Identificacao loja | Numero WhatsApp → STORE_MAP | Deterministico, sem ambiguidade |
