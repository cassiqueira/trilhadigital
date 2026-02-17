# Trilha Digital

Software inteligente para operacoes comerciais. Dois produtos, uma infraestrutura — rodando inteiramente no Google Cloud Platform.

---

## Produtos

### Claria — Atendimento WhatsApp com IA

Plataforma SaaS multi-tenant que transforma canais de mensageria em operacoes de vendas autonomas. A IA atende, qualifica e converte leads 24/7 pelo WhatsApp, Instagram e Messenger.

**Problema:** Empresas perdem vendas porque nao conseguem responder rapido o suficiente. O tempo medio de primeira resposta no WhatsApp comercial e de 3+ horas. Leads esfriam, concorrentes respondem antes.

**Solucao:** Motor de IA conversacional que responde em segundos com a profundidade de um consultor senior. Nao e chatbot de arvore de decisao — e um pipeline de 21 etapas com raciocinio contextual, orquestrado por LangGraph e alimentado por LLMs de ultima geracao via Vertex AI.

#### Arquitetura do Motor de IA

O core e um grafo LangGraph com 21 nos que processa cada mensagem:

```
mensagem recebida
    |
    v
[rate limiter] -> [horario comercial] -> [transcrever audio]
    |
    v
[carregar estado] -> [selecionar persona] -> [build prompt]
    |
    v
[Inference Engine (Gemini/Vertex AI)] -> [processar resposta] -> [check objecao]
    |                                                                  |
    v                                                            [loop reforco]
[validar CEP] -> [check handoff] -----> [handoff humano]
    |
    v
[gerar TTS] -> [enviar resposta] -> [atualizar labels]
    |
    v
[atualizar deal stage] -> [disparar CAPI] -> [salvar estado]
```

Cada no e independente, testavel e configuravel por tenant. O grafo decide dinamicamente o caminho baseado no contexto da conversa.

#### Funcionalidades

| Funcionalidade | Descricao |
|---------------|-----------|
| **Persona IA customizavel** | Cada tenant define tom, regras e conhecimento da sua IA |
| **Multi-canal** | WhatsApp (WAHA Plus), Instagram, Messenger (Meta Cloud API) |
| **Lite Inbox** | CRM embutido com historico em tempo real (WebSocket) |
| **RAG semantico** | Base de conhecimento por tenant com pgvector (busca vetorial) |
| **Quiz interativo** | Formularios conversacionais via WhatsApp para captura de dados |
| **Validacao de CEP** | Verificacao automatica de area de entrega durante a conversa |
| **TTS (Text-to-Speech)** | Respostas em audio com voz sintetizada |
| **Transcricao de audio** | Audios recebidos transcritos automaticamente (Whisper API) |
| **Handoff humano** | Transferencia para operador quando a IA detecta necessidade |
| **Pipeline de vendas** | Kanban com stages customizaveis e movimentacao automatica |
| **Conversions API** | Eventos de conversao disparados direto para o Meta Ads |

#### Resultados

- Tempo de primeira resposta: de **3 horas** para **30 segundos**
- Conversoes: **+47%** apos implementacao
- Resolucao autonoma: **70%** das conversas sem intervencao humana

---

### Traffic OS — Mineracao de Anuncios com IA

Plataforma de gestao para agencias de trafego pago e equipes comerciais B2B. Automatiza o funil desde captura ate fechamento, com IA fazendo qualificacao, follow-up e agendamento.

#### Funcionalidades

| Funcionalidade | Descricao |
|---------------|-----------|
| **Analytics de campanhas** | Importacao de dados Facebook Ads com metricas consolidadas (spend, CPM, CPC, CTR, ROI) |
| **Qualificacao automatica** | IA analisa e pontua leads recebidos |
| **Follow-up inteligente** | Cadencias automatizadas com timing otimizado |
| **Agendamento integrado** | Cal.com self-hosted para reunioes |
| **Dashboard de pipeline** | Visibilidade completa do funil em tempo real |
| **Analise preditiva** | Planejamento de integracao com BigQuery para analise preditiva de dados de trafego e otimizacao de campanhas |

---

## Stack Tecnico

### Backend

| Tecnologia | Uso |
|-----------|-----|
| Python 3.11 | Linguagem principal |
| FastAPI | API REST, WebSockets, Webhooks |
| LangGraph | Orquestracao do pipeline de IA (21 nos) |
| Google Vertex AI + Gemini | Orquestracao multimodal de LLMs — inferencia de alta performance com Gemini 1.5 Pro/Flash |
| Claude (Anthropic) | Suporte a tarefas especificas de raciocinio logico complexo |
| Celery + Redis | Background jobs, agendamento, cache |
| Cloud SQL for PostgreSQL 15 + pgvector | Banco relacional gerenciado + busca vetorial (RAG) |
| Whisper API | Transcricao de audio |

### Frontend

| Tecnologia | Uso |
|-----------|-----|
| React 18 + TypeScript | UI Library |
| Vite | Build tool |
| Tailwind CSS | Estilos |
| React Query | Data fetching + cache |

### Infraestrutura Google Cloud

| Componente | Servico |
|-----------|---------|
| Compute | Google Compute Engine — atualmente em instancias dedicadas, com roadmap de migracao para **Cloud Run** / **GKE** para escalabilidade horizontal automatica |
| Banco de dados | **Cloud SQL for PostgreSQL 15** (gerenciado, HA, backups automaticos) |
| Data Warehouse | **BigQuery** (planejado) — analise preditiva de dados de trafego e comportamento de leads |
| Containers | Docker Compose (7 servicos), migracao planejada para **Artifact Registry + Cloud Run** |
| AI/ML | **Vertex AI** — endpoint de inferencia para Gemini 1.5 Pro/Flash com suporte a multimodalidade |
| WhatsApp Gateway | WAHA Plus (GOWS engine) |
| Meta Channels | Cloud API (Instagram, Messenger, WhatsApp Business) |
| Object Storage | Cloudflare R2 (S3-compatible), avaliando migracao para **Cloud Storage** |
| TLS/CDN | Cloudflare |
| Reverse Proxy | Nginx (SNI routing, TLS termination) |

### Arquitetura

```
                    Cloudflare CDN
                        |
              +---------+---------+
              |                   |
        app.useclaria.com    api.useclaria.com
              |                   |
              v                   v
    +---------------------------------------------+
    |              Nginx (TLS, routing)            |
    |   app → React SPA (static)                   |
    |   api → FastAPI (proxy reverso)              |
    +---------------------------------------------+
                      |
            +-------------------+
            |   FastAPI :8000   |
            |   LangGraph (IA)  |
            |   Celery workers  |
            +--------+----------+
                     |
        +------------+------------+
        |            |            |
     Redis      Cloud SQL    Vertex AI
     (cache)   (PostgreSQL)  (Gemini LLM)
```

### Seguranca

- **Multi-tenant com RLS**: Row-Level Security nativo no PostgreSQL — 39 policies que isolam dados por tenant no nivel do banco
- **Dual pool**: Conexoes da aplicacao rodam como `app_user` (sem superuser), com RLS ativo por padrao
- **JWT + RBAC**: Autenticacao com refresh tokens, roles (admin, gerencial, operador)
- **Rate limiting**: Sliding window por IP/tenant com fallback em memoria (fail-closed)
- **SQL parametrizado**: 100% das queries — zero risco de injection
- **Input validation**: Pydantic em todas as rotas
- **Docker hardened**: Containers com `no-new-privileges`, `cap_drop: ALL`
- **Security headers**: CSP, HSTS, Referrer-Policy, Permissions-Policy

---

## Roadmap de Escalabilidade (GCP)

O projeto esta em fase de expansao da infraestrutura Google Cloud para suportar o crescimento de tenants e volume de mensagens:

| Fase | Migracao | Beneficio |
|------|----------|-----------|
| **Q1 2026** | Vertex AI como endpoint primario de inferencia | Latencia reduzida, billing unificado GCP |
| **Q2 2026** | Cloud Run para agente-api + celery workers | Auto-scaling horizontal, zero cold start |
| **Q2 2026** | BigQuery para analytics do Traffic OS | Queries analiticas em TB de dados sem impacto no OLTP |
| **Q3 2026** | Cloud Storage para midias (audio, imagens) | Integracao nativa com Vertex AI e Cloud CDN |
| **Q3 2026** | Artifact Registry + Cloud Build | CI/CD nativo GCP, deploy automatizado |
| **Q4 2026** | GKE para orquestracao multi-servico | Service mesh, observabilidade com Cloud Monitoring |

---

## Desenvolvimento

A abordagem de desenvolvimento assistido por IA permite que um desenvolvedor solo mantenha e evolua uma plataforma com a complexidade e qualidade de uma equipe de 5+ engenheiros.

### Metodologia

- **SDD (Software Development Discipline)**: Fluxo estruturado de spec → exec → test → commit
- **Memory Bank**: Contexto persistente entre sessoes de desenvolvimento
- **RAG interno**: Base de conhecimento indexada do proprio codebase com pgvector
- **Code review automatizado**: Cada alteracao passa por revisao do agente de IA

### Metricas do Codebase

| Metrica | Valor |
|---------|-------|
| Linguagens | Python, TypeScript, SQL |
| Arquivos Python (backend) | 60+ |
| Componentes React (frontend) | 40+ |
| Nos do grafo LangGraph | 21 |
| Tabelas PostgreSQL | 35+ |
| Policies RLS | 39 |
| Migrations | 60+ |
| Testes | Unitarios + E2E (Playwright) |

---

## Licenca

Proprietario. Todos os direitos reservados.

---

*Trilha Digital — Software inteligente para operacoes comerciais, powered by Google Cloud.*
