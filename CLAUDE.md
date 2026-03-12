# Mundo Ray — CRM Atacado B2B (Triadeflow)

## O que eh
Implementacao de CRM profissional para Mundo Ray — atacado B2B de roupas infantis em Fortaleza/CE. Faz parte do mesmo grupo do Sushi da Hora.
Processo comercial desenhado, WhatsApp centralizado, automacoes de follow-up, dashboard de metricas e integracao Bling.

## Cliente
- **Empresa:** Mundo Ray | Atacado (@mundoray_)
- **Segmento:** Roupas infantis — atacado B2B para lojistas
- **Local:** Fortaleza/CE
- **Site:** https://www.lojaatacadomundoray.com.br/
- **Instagram:** @mundoray_
- **Plataforma e-commerce:** Loja Integrada (AWS CDN)
- **ERP:** Bling
- **Grupo:** Mesmo grupo do Sushi da Hora (Ray)

## Operacao Atual
- 2+ vendedores atendendo via WhatsApp pessoal
- 5+ canais de venda (Shopee, TikTok, Mercado Livre, site, WhatsApp)
- 15+ grupos ativos de lojistas
- Ja teve 6 numeros de WhatsApp, sem controle
- Base de clientes no Bling subutilizada
- Sem CRM, sem pipeline, sem metricas

## Proposta Comercial
- **URL:** https://mundoray.triadeflow.ai
- **Repo:** triadeflow-ia/mundo-ray (publico — GitHub Pages)
- **Branch:** main
- **Tech:** HTML single-page, dark theme, Poppins font
- **Design:** Split hero (texto esquerda + imagem direita), padrao Hiro Bot

### Identidade Visual Mundo Ray
- **Rosa/Pink:** #fa72a5 (primaria)
- **Verde Teal:** #30a77e (accent)
- **Ciano:** #50d3cc (acentos secundarios)
- **Verde Claro:** #9dd466 (botoes secundarios)
- **Dark text:** #201f1f
- **Fonte:** Poppins
- **Estilo:** Light + Moderno + Infantil (site deles), dark theme na proposta

### Pricing Proposto
- **Implementacao:** R$5.000 (unico)
  - Onboard, processo comercial, CRM, importacao Bling, automacoes, dashboard, treinamento
  - 3 meses de acompanhamento
- **Plataforma:** R$97/usuario/mes
  - 3 usuarios iniciais (2 vendedores + 1 admin) = R$291/mes
  - Opcao com integracao completa: R$120/usuario/mes
  - IA opcional: R$150/mes ilimitado

### 6 Modulos
1. CRM & Pipeline de Vendas (atacado B2B customizado)
2. WhatsApp Centralizado (chat unificado, app mobile)
3. Automacoes & Follow-up (reativacao base, lembretes)
4. Dashboard & Metricas (vendas por vendedor, conversao)
5. Integracao Bling (clientes + pedidos)
6. Publicacao Multi-Canal (redes sociais — bonus)

### Roadmap Implementacao (4 semanas)
- S1: Discovery & Arquitetura
- S2: CRM & Integracoes (Bling)
- S3: Automacoes & Dashboard
- S4: Treinamento & Go-Live
- +3 meses acompanhamento

## Stack (DECISAO 2026-03-11)
- **CRM:** GoHighLevel (LC Phone + Conversations API)
- **ERP:** Bling (integracao)
- **WhatsApp:** API Oficial via GHL LC Phone — numero fica no GHL
- **Agente IA:** FastAPI + LangGraph (Railway) — multi-tenant 11 lojas
- **Trigger:** GHL Workflow → Webhook direto pro agente (sem n8n)
- **Modelo:** GPT-4.1-mini (cost-efficient pra volume)
- **Base:** 80K+ clientes
- **Arquitetura completa:** `ARQUITETURA-AGENTE-GRUPO-RAY.md`

## Estado (2026-03-11)
- [x] Proposta comercial criada e publicada
- [x] Reuniao realizada — cliente quer iniciar segunda-feira
- [x] Dominio mundoray.triadeflow.ai configurado (Cloudflare CNAME + GitHub Pages)
- [x] SSL em provisionamento (GitHub Pages auto)
- [ ] **PROXIMA SESSAO (Segunda-feira):** Arquitetura completa do projeto
  - Definir GHL Location (criar ou usar existente)
  - Desenhar pipeline de atacado B2B
  - Mapear campos, tags, automacoes
  - Definir integracao Bling
  - Planejar importacao da base de clientes
  - Definir WhatsApp (qual provedor)

## Arquivos
```
mundo-ray-site/          (repo: triadeflow-ia/mundo-ray)
├── index.html           # Proposta comercial (dark theme, split hero)
├── CNAME                # mundoray.triadeflow.ai
├── triadeflow-logo.png  # Logo Triadeflow (branca — nao usada, SVG inline no footer)
└── CLAUDE.md            # Este arquivo
```

## Comandos
```bash
# Abrir proposta local
start "" "C:\tmp\mundo-ray-proposta.html"

# Push atualizacoes
cp /c/tmp/mundo-ray-proposta.html /c/tmp/mundo-ray-site/index.html
cd /c/tmp/mundo-ray-site && git add -A && git commit -m "desc" && git push origin main
```
