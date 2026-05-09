# Quitte — Arquitetura Técnica e Stack de Desenvolvimento

> **Documento de referência técnica** para uso como contexto em ferramentas de IA (Gemini, Claude Code).
> Gerado com base na visão de produto da Quitte: plataforma de Recuperação Financeira Assistida.

---

## 1. Visão Geral do Produto

A **Quitte** é uma plataforma de "Recuperação Financeira Assistida" focada em resolver o ciclo crônico de endividamento no Brasil. Utiliza a metodologia **50-30-20** como espinha dorsal:

- **50%** — Necessidades (gastos essenciais)
- **30%** — Desejos (gastos flexíveis)
- **20%** — Dívidas e Futuro (no MVP: 100% amortização de débitos priorizados)

### Modelo de Negócio
- **Primário (B2B2C):** Venda para RHs como benefício corporativo (PEPM — Per Employee Per Month)
- **Secundário (B2C):** Freemium com assinatura para recursos avançados

### Pilares Funcionais
1. Diagnóstico Inteligente (Anamnese financeira)
2. Ingestão de dados multinível: Open Finance → Upload de extratos → Chat assistido por LLM
3. Plano de Ação Semanal via WhatsApp (micro-tarefas)
4. RAG sobre legislação financeira brasileira (CDC, Lei 14.181/2021, normativas BCB)

---

## 2. Decisão Arquitetural

### Abordagem: Modular Monolith no MVP → Microserviços sob demanda

**Justificativa:**
- Reduz complexidade operacional no MVP
- Mantém velocidade de desenvolvimento para time enxuto
- Fronteiras de domínio claras permitem extração futura de serviços (AI Module, Notification Service)
- Escalabilidade validada antes de distribuição prematura

---

## 3. Diagrama de Arquitetura (Textual)

```
┌─────────────────────────────────────────────────────┐
│                     Clientes                        │
│  Web App (Next.js) │ Mobile (Expo) │ WhatsApp │ RH  │
└────────────┬────────────────────────────────────────┘
             │ HTTPS / WebSocket
┌────────────▼────────────────────────────────────────┐
│         API Gateway (Kong ou AWS API Gateway)        │
│      Auth · Rate Limit · Tenant Routing              │
└────────────┬────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────┐
│         Backend — NestJS (Modular Monolith)          │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │  Auth    │ │  Dívidas │ │  Planos  │ │Tenant  │ │
│  │  Module  │ │  Module  │ │  Module  │ │Module  │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │
│  │Open Fin. │ │WhatsApp  │ │   AI / RAG Module  │  │
│  │ Module   │ │ Module   │ │  (LangChain + MCP) │  │
│  └──────────┘ └──────────┘ └────────────────────┘  │
└────────────┬────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────┐
│                  Camada de Dados                     │
│  PostgreSQL (RLS multitenancy) │ Redis (cache/queue) │
│  pgvector (RAG embeddings)     │ S3 (extratos/docs)  │
└─────────────────────────────────────────────────────┘
```

---

## 4. Stack Técnica Detalhada

### 4.1 Backend

| Tecnologia | Uso |
|---|---|
| **Node.js + TypeScript** | Runtime principal |
| **NestJS** | Framework backend modular por domínio |
| **Prisma ORM** | Acesso ao banco, migrations, type-safety |
| **BullMQ + Redis** | Filas: extratos, WhatsApp scheduler, Open Finance jobs |
| **Passport.js** | Autenticação: JWT + OAuth2 (Open Finance) |
| **Zod** | Validação de schemas de entrada |

**Módulos NestJS por domínio:**
- `AuthModule` — JWT, refresh token, OAuth2 PKCE
- `DebtModule` — CRUD de dívidas, priorização por juros/risco
- `PlanModule` — Geração e acompanhamento do plano 50-30-20
- `TenantModule` — Multitenancy B2B2C, gestão de RHs
- `OpenFinanceModule` — Integração BCB, consentimentos, refresh
- `WhatsAppModule` — Envio de micro-tarefas, webhooks de resposta
- `AIModule` — RAG, prompts, MCP tools, orquestração LLM

### 4.2 Banco de Dados

| Tecnologia | Uso |
|---|---|
| **PostgreSQL** | Banco principal |
| **Row Level Security (RLS)** | Isolamento de tenants via `tenant_id` |
| **pgvector** | Embeddings para RAG (evita serviço externo no MVP) |
| **Redis** | Sessões, cache de planos, rate limit, filas BullMQ |
| **AWS S3** | Armazenamento de extratos bancários e faturas (PDFs) |

**Estratégia de multitenancy:**
- Uma única database com isolamento via `tenant_id` + RLS
- Mais simples que schema-per-tenant no MVP
- RLS garante que queries nunca vazam dados entre tenants

### 4.3 Camada de IA e RAG

| Tecnologia | Uso |
|---|---|
| **LangChain.js** ou **LlamaIndex.TS** | Orquestração de pipelines RAG |
| **MCP (Model Context Protocol)** | Tools contextuais: consultar dívidas, acionar Open Finance, gerar plano |
| **Claude API (Anthropic)** | LLM primário — raciocínio financeiro, PT-BR, dados sensíveis |
| **GPT-4o (OpenAI)** | Fallback de LLM |
| **pgvector** | Store de embeddings (legislação financeira brasileira) |
| **text-embedding-3-small** (OpenAI) ou **embed-multilingual-v3** (Cohere) | Embeddings em português |

**Base de conhecimento RAG:**
- Código de Defesa do Consumidor (CDC)
- Lei 14.181/2021 — Lei do Superendividamento
- Resoluções e normativas do Banco Central do Brasil (BCB)
- Jurisprudência relevante sobre renegociação de dívidas

**Fluxo RAG:**
1. Usuário descreve situação → LLM identifica intent
2. MCP Tool consulta dívidas do usuário no banco
3. RAG recupera legislação relevante via pgvector
4. LLM gera resposta contextualizada + próximo passo do plano
5. Resultado persiste no `PlanModule`

### 4.4 Open Finance

| Item | Detalhe |
|---|---|
| **SDK** | Opus Open Finance ou integração direta BCB |
| **Fluxo** | OAuth2 PKCE → consentimento → coleta de transações/dívidas → parsing → normalização |
| **Agendamento** | Jobs BullMQ respeitando TTL dos consentimentos |
| **Escopos** | Minimização LGPD: apenas escopos estritamente necessários |

### 4.5 Canal WhatsApp

| Opção | Recomendação |
|---|---|
| **Meta Cloud API** | Opção mais barata, integração direta |
| **Zenvia / Take Blip** | **Recomendado MVP** — suporte PT-BR, compliance nacional, menor atrito |
| **Twilio** | Alternativa internacional, maior custo |

**Fluxo de micro-tarefas:**
- BullMQ scheduler dispara tarefa semanal por usuário
- Template pré-aprovado Meta Business → entregue via WhatsApp
- Resposta do usuário capturada via webhook → atualiza progresso no `PlanModule`

### 4.6 Frontend Web

| Tecnologia | Uso |
|---|---|
| **Next.js 14+ (App Router)** | SSR para SEO (landing), RSC para dashboard |
| **TypeScript** | Type-safety end-to-end |
| **Tailwind CSS + shadcn/ui** | Design system, velocidade de desenvolvimento |
| **Zustand** | Estado global leve (sem Redux no MVP) |
| **TanStack Query (React Query)** | Cache e sincronização com a API |
| **Recharts ou Tremor** | Visualizações: progresso de dívidas, projeção 50-30-20 |

### 4.7 Mobile

| Tecnologia | Uso |
|---|---|
| **React Native + Expo SDK** | App iOS e Android (managed workflow) |
| **Expo Router** | Navegação file-based, alinhada ao Next.js |
| **Tamagui ou React Native Paper** | UI components mobile |
| **`@quitte/core`** | Pacote interno com lógica de negócio compartilhada com web |

### 4.8 Infra e DevOps

| Tecnologia | Uso |
|---|---|
| **Docker + Docker Compose** | Ambiente de desenvolvimento local |
| **AWS (sa-east-1)** | Cloud principal — conformidade LGPD, data residency no Brasil |
| **ECS (Fargate) ou Kubernetes** | Orquestração em produção |
| **GitHub Actions** | CI/CD — build, test, deploy |
| **Sentry** | Monitoramento de erros (frontend + backend) |
| **OpenTelemetry + Grafana** | Métricas, traces, observabilidade |
| **AWS Secrets Manager ou Doppler** | Gestão de segredos e variáveis de ambiente |

---

## 5. Estrutura de Repositório (Monorepo)

```
quitte/
├── apps/
│   ├── api/               # NestJS — Backend principal
│   ├── web/               # Next.js — Web app + Portal RH
│   └── mobile/            # Expo — React Native iOS/Android
├── packages/
│   ├── core/              # Lógica de negócio compartilhada (TypeScript)
│   ├── ui/                # Design system compartilhado (web + mobile)
│   └── ai/                # RAG, prompts, MCP tools, embeddings
├── infra/                 # Terraform / AWS CDK
├── docs/                  # Documentação técnica e de produto
│   ├── adr/               # Architecture Decision Records
│   └── api/               # OpenAPI / Swagger specs
└── .github/
    └── workflows/         # CI/CD pipelines
```

**Tooling de monorepo:** [Turborepo](https://turbo.build/) — build cache, pipelines paralelas, DX otimizada para time enxuto.

---

## 6. Conformidade LGPD por Design

| Requisito LGPD | Implementação Técnica |
|---|---|
| **Consentimento granular** | Tabela `consents` com versioning e audit log imutável |
| **Direito ao esquecimento** | Soft delete + job de anonimização agendado (BullMQ) |
| **Portabilidade de dados** | Endpoint `GET /me/export` → JSON/CSV assinado |
| **Minimização de dados** | Open Finance: apenas escopos estritamente necessários |
| **Segurança em repouso** | RDS Encrypt at rest + colunas sensíveis com `pgcrypto` |
| **Logs de acesso** | Audit trail em tabela append-only para dados financeiros |
| **DPO e relatórios** | Painel de privacidade no Portal RH (TenantModule) |

---

## 7. Decisões Arquiteturais Registradas (ADRs)

### ADR-001: Modular Monolith no MVP
- **Decisão:** Iniciar com monolito modular, não microserviços
- **Motivo:** Velocidade de desenvolvimento, menor complexidade operacional
- **Consequência:** Extração de serviços (AI, Notifications) quando escala justificar

### ADR-002: pgvector em vez de Pinecone
- **Decisão:** Usar extensão pgvector no PostgreSQL existente
- **Motivo:** Elimina serviço externo pago no MVP; migração cirúrgica quando necessário
- **Consequência:** Avaliar Pinecone/Weaviate se volume de embeddings escalar significativamente

### ADR-003: Claude API como LLM primário
- **Decisão:** Anthropic Claude como provider principal de LLM
- **Motivo:** Melhor raciocínio financeiro estruturado, seguimento de instruções em PT-BR, adequado para dados financeiros sensíveis — alinhado à filosofia de "Autoridade Benevolente" do produto
- **Consequência:** GPT-4o como fallback; arquitetura de providers abstraída via LangChain

### ADR-004: Zenvia/Take Blip para WhatsApp
- **Decisão:** BSP nacional em vez de Meta Cloud direto no MVP
- **Motivo:** Suporte PT-BR, compliance com regulações brasileiras, menor atrito de onboarding
- **Consequência:** Migrar para Meta Cloud direto quando volume justificar custo de operação própria

### ADR-005: AWS sa-east-1 como região principal
- **Decisão:** Todos os dados residem em São Paulo (sa-east-1)
- **Motivo:** Conformidade LGPD, latência para usuários brasileiros, requisitos de soberania de dados financeiros
- **Consequência:** Multi-region somente se produto expandir para LATAM

---

## 8. Roadmap Técnico — MVP (16 semanas)

| Fase | Semanas | Entregas |
|---|---|---|
| **Fundação** | 1–3 | Monorepo, CI/CD, Auth (JWT + OAuth2), banco + RLS, multitenancy base |
| **Core Domain** | 4–7 | Módulos Dívidas e Planos, lógica 50-30-20, API REST documentada |
| **Integrações** | 8–11 | Open Finance (consentimento + coleta), WhatsApp (micro-tarefas), upload de extratos |
| **IA / RAG** | 10–13 | Pipeline RAG, MCP tools, anamnese assistida por LLM, base legislação |
| **Frontend** | 6–14 | Web app (Next.js), Mobile (Expo), Portal RH |
| **Qualidade e Go-live** | 14–16 | Testes E2E, pentest, LGPD audit, deploy produção AWS |

---

## 9. Variáveis de Ambiente Críticas (Referência)

```env
# Database
DATABASE_URL=postgresql://...
REDIS_URL=redis://...

# Auth
JWT_SECRET=
JWT_REFRESH_SECRET=

# Open Finance
OPEN_FINANCE_CLIENT_ID=
OPEN_FINANCE_CLIENT_SECRET=
OPEN_FINANCE_REDIRECT_URI=

# AI / LLM
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
COHERE_API_KEY=

# WhatsApp
WHATSAPP_BSP_API_KEY=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_WEBHOOK_VERIFY_TOKEN=

# AWS
AWS_REGION=sa-east-1
AWS_S3_BUCKET=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

# Observabilidade
SENTRY_DSN=
```

---

*Documento gerado para uso como contexto técnico em Gemini e Claude Code.*
*Versão 1.0 — Quitte Platform — Antigravity Ecosystem*
