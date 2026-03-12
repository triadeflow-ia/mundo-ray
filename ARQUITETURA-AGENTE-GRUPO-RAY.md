# Arquitetura — Agente IA Multi-Loja (Grupo Ray)

> 11 lojas, 1 agente, GHL + WhatsApp API Oficial + FastAPI/LangGraph

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
              ┌──────────────────────────┐
              │   AGENTE (Railway)       │
              │   FastAPI + LangGraph    │
              │                          │
              │  POST /webhook/ghl       │
              │    → Identifica loja     │
              │    → Carrega prompt      │
              │    → Processa (ReAct)    │
              │    → Responde via GHL API│
              └──────────────────────────┘
```

## Stack

| Camada | Tech | Detalhes |
|--------|------|----------|
| **CRM** | GoHighLevel | 1 Location ou Agency (11 lojas como sub) |
| **WhatsApp** | GHL LC Phone (API Oficial) | 1 numero por loja OU 1 numero central |
| **Trigger** | GHL Workflow | "Customer Replied" → Webhook |
| **Agente** | FastAPI + LangGraph | Railway (Docker) |
| **Motor IA** | GPT-4.1-mini ou Claude Haiku | Cost-efficient pra volume |
| **Deploy** | Railway | 1 instancia, multi-tenant |

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

### Opcao B: Agency (escala futura)
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
     URL: https://{railway-url}/webhook/ghl
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
    prompt = load_store_prompt(store)
    agent = create_react_agent(model, tools, prompt)
    response = await run_agent(agent, payload.message, payload.contact_id)
    await ghl_send_message(payload.conversation_id, response)
```

### Identificacao de loja
```python
# Cada numero de WhatsApp mapeia pra uma loja
STORE_MAP = {
    "558599210061": {"id": "barra", "brand": "sushi", "name": "Sushi da Hora - Barra do Ceará"},
    "558598405232": {"id": "parquelandia", "brand": "sushi", "name": "Sushi da Hora - Parquelândia"},
    "558598554849": {"id": "maraponga", "brand": "sushi", "name": "Sushi da Hora - Maraponga"},
    "558598661202": {"id": "maracanau", "brand": "sushi", "name": "Sushi da Hora - Maracanaú"},
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
| `consultar_catalogo` | Produtos/colecoes disponíveis |
| `consultar_pedido_bling` | Status pedido via Bling API |
| `consultar_estoque` | Disponibilidade via Bling |
| `calcular_frete` | Estimativa por regiao |

## Estrutura do Projeto

```
grupo-ray-bot/
├── src/
│   ├── main.py                  # FastAPI app
│   ├── config.py                # Settings (env vars)
│   ├── agent/
│   │   ├── graph.py             # LangGraph ReAct agent (multi-tenant)
│   │   ├── tools/
│   │   │   ├── shared.py        # Tools compartilhadas (GHL, mensagem)
│   │   │   ├── sushi.py         # Tools Sushi da Hora
│   │   │   └── mundoray.py      # Tools Mundo Ray
│   │   ├── prompts/
│   │   │   ├── base.py          # Prompt base
│   │   │   ├── sushi_da_hora.py # Prompt marca Sushi
│   │   │   ├── mundo_ray.py     # Prompt marca Mundo Ray
│   │   │   └── stores.py        # Overrides por loja
│   │   ├── store_map.py         # Mapeamento numero → loja
│   │   └── nodes.py             # Preprocessamento (audio, imagem)
│   ├── integrations/
│   │   ├── ghl.py               # GHL CRM + Conversations API
│   │   └── bling.py             # Bling ERP API (Mundo Ray)
│   └── models/
│       └── schemas.py           # Pydantic models
├── requirements.txt
├── Dockerfile
├── railway.toml
└── .env
```

## Env Vars (Railway)

```env
# IA
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4.1-mini

# GHL
GHL_API_TOKEN=pit-...
GHL_LOCATION_ID=...

# Bling (Mundo Ray)
BLING_API_TOKEN=...

# Config
PORT=8000
ENV=production
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
| Railway (agente) | ~$5-10 (1 instancia) |
| OpenAI GPT-4.1-mini | ~R$50-150 (depende do volume) |
| GHL (plataforma) | Ja incluso no plano do cliente |
| WhatsApp conversas | ~R$0.15/conversa service |
| **Total estimado** | **~R$200-400/mes** pra 11 lojas |

## Escalabilidade

- **1 instancia Railway** aguenta todas as 11 lojas (async, nao-blocking)
- Se volume crescer: Railway escala automatico (+ replicas)
- Adicionar loja nova = adicionar entrada no STORE_MAP + prompt
- Adicionar marca nova = novo arquivo de prompt + tools especificas

## Roadmap de Implementacao

### Semana 1: Base
- [ ] Criar repo `grupo-ray-bot`
- [ ] FastAPI + estrutura multi-tenant
- [ ] GHL integration (Conversations API send/receive)
- [ ] Webhook endpoint + store identification
- [ ] Prompt base + prompt Sushi da Hora (migrar do Hiro)

### Semana 2: Sushi da Hora (5 lojas)
- [ ] Migrar tools do Hiro (adaptar pra GHL messaging)
- [ ] Prompts por unidade (5 overrides)
- [ ] GHL Workflows (5 triggers)
- [ ] Testar nas 5 unidades

### Semana 3: Mundo Ray
- [ ] Prompt Mundo Ray (atacado B2B)
- [ ] Tools Bling (pedido, estoque, catalogo)
- [ ] GHL Workflow Mundo Ray
- [ ] Testar

### Semana 4: Demais lojas + Go-Live
- [ ] Adicionar lojas restantes
- [ ] Testes end-to-end todas as 11
- [ ] Go-live gradual
- [ ] Monitoramento

## Decisoes Tecnicas

| Decisao | Escolha | Motivo |
|---------|---------|--------|
| WhatsApp | API Oficial via GHL LC Phone | CRM unificado, vendedor ve tudo |
| Webhook | GHL Workflow → Agente direto | Sem n8n, menos latencia |
| Multi-tenant | 1 instancia, STORE_MAP | Simples, barato, escala |
| Modelo IA | GPT-4.1-mini | Cost-efficient pra volume 11 lojas |
| Deploy | Railway (Docker) | Ja usado nos outros projetos |
| Identificacao loja | Numero WhatsApp → STORE_MAP | Deterministico, sem ambiguidade |
