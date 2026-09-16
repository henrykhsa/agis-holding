<p align="center">
  <img src="https://img.shields.io/badge/Holding-Agis%20Group-000000?style=for-the-badge&logo=data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PHBhdGggZD0iTTEyIDJMMyA3djEwbDkgNSA5LTVWN2wtOS01eiIvPjwvc3ZnPg==" alt="Agis Group" />
  <img src="https://img.shields.io/badge/Status-Building-blue?style=for-the-badge" alt="Status" />
  <img src="https://img.shields.io/badge/Licença-Proprietária-red?style=for-the-badge" alt="Licença" />
</p>

<h1 align="center">Agis Group</h1>

<p align="center">
  <strong>Tecnologia que move negócios. Software que move pessoas.</strong>
</p>

<p align="center">
  Holding de tecnologia focada em soluções de gestão corporativa e finanças pessoais.<br/>
  Construímos sistemas robustos, interligados e escaláveis — do B2B ao B2C.
</p>

---

## Visão

Ser a infraestrutura digital que conecta empresas aos seus clientes finais — da operação interna à vida financeira de milhões de pessoas — com segurança, performance e design como pilares inegociáveis.

## Missão

Construir um ecossistema de produtos de software que resolva problemas reais de gestão e finanças, unindo:

- **Segurança** — Dados protegidos, RBAC contextual, auditoria total.
- **Performance** — Server-first, edge computing, experiências instantâneas.
- **Design** — Interfaces que respeitam o tempo e a inteligência do usuário.

---

## O Ecossistema

```
                        ┌─────────────────────┐
                        │     AGIS GROUP      │
                        │  Holding de Tech    │
                        └────────┬────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────▼────────────┐ ┌───────▼───────────────┐ ┌────▼────────────────────┐
│   LAURUS CLOUD      │ │   DELIVERY ENGINE     │ │         SALV            │
│      (B2B)          │ │      (B2B2B)          │ │        (B2C)            │
│                     │ │                       │ │                         │
│  SaaS de Gestão     │ │  Hub de Integração    │ │  Educação & Gestão      │
│  Corporativa + PMS  │ │  de Delivery          │ │  Financeira Pessoal     │
│  + POS              │ │  (API as a Service)   │ │                         │
└─────────────────────┘ └───────────────────────┘ └─────────────────────────┘
```

---

### Laurus Cloud — B2B

> Plataforma SaaS multi-tenant de gestão hoteleira e ponto de venda.

| Dimensão | Detalhe |
|----------|---------|
| Mercado | Hospitality, F&B, corporativo |
| Módulos | PMS, POS, Financeiro, Governança, CRM, Auditoria |
| Modelo | Assinatura SaaS |
| Status | Em produção |

PMS completo com reservas, check-in/out, turnos de caixa, RBAC com 16 roles, auditoria noturna e integração POS para alimentos e bebidas. Arquitetura multi-tenant com isolamento total por cliente.

**Em desenvolvimento futuro:** Motor de Integração de Delivery — produto SaaS independente (API as a Service) que centraliza pedidos de marketplaces (iFood, 99Food, Rappi) e os entrega via webhook padronizado para qualquer sistema de PDV. O Laurus Cloud será o Cliente Nº 1 a consumir essa API. ([Documentação técnica →](./docs/features/delivery-integration-engine.md))

**[Documentação completa →](./Laurus-Cloud/README.md)**

---

### Salv — B2C

> App de educação e gestão financeira pessoal. Freemium, gamificado, com roadmap para conta digital.

| Dimensão | Detalhe |
|----------|---------|
| Mercado | Massa — 80M+ brasileiros sem controle financeiro |
| Modelo | Freemium (ads) → Premium → Open Finance → BaaS |
| Estratégia | Cavalo de Troia: app de hábito → fintech → banco digital |
| Status | Concepção e arquitetura |

Acesso completo gratuito suportado por anúncios. Gamificação real (streaks, XP, ranks). Evolução planejada para conta digital via Banking as a Service.

**[Documentação completa →](./Salv/README.md)**

---

## Cultura de Engenharia

### Build in Public

Documentamos a jornada técnica publicamente. Cada decisão de arquitetura, cada pivô de produto e cada lição aprendida é registrada e compartilhada.

Mantemos um repositório centralizado de:

- Posts técnicos e artigos de engenharia
- Estratégias de mercado e posicionamento
- Decisões de arquitetura (ADRs)
- Roadmaps públicos por produto

> Transparência não é vulnerabilidade. É credibilidade.

### Princípios de Engenharia

| Princípio | Prática |
|-----------|---------|
| **Ship fast, ship safe** | CI/CD automatizado, deploys diários, zero downtime |
| **Convention over configuration** | Padrões claros reduzem decisões desnecessárias |
| **Server-first** | RSC, edge middleware, dados no servidor sempre que possível |
| **Type everything** | TypeScript strict em todo o ecossistema |
| **Own the data** | PostgreSQL como fonte de verdade, schemas por domínio |
| **Document as you build** | Código sem documentação é dívida silenciosa |

---

## Stack Tecnológico Global

Tecnologias que orbitam o ecossistema Agis — compartilhadas entre produtos para consistência, mobilidade de time e redução de custo cognitivo.

### Core

| Camada | Tecnologia |
|--------|------------|
| Linguagem | **TypeScript** (strict, em toda superfície) |
| Runtime | **Node.js** |
| ORM | **Prisma** (multi-schema, type-safe) |
| Banco de Dados | **PostgreSQL** |

### Web (Laurus Cloud)

| Camada | Tecnologia |
|--------|------------|
| Framework | **Next.js** (App Router, RSC) |
| UI | **React 19** + **Tailwind CSS** |
| Auth | **NextAuth.js** (JWT) |
| Hosting | **Vercel** |

### Mobile (Salv — projetado)

| Camada | Tecnologia |
|--------|------------|
| Framework | **React Native / Expo** |
| Estado | A definir na fase de arquitetura |
| Backend | **Node.js** (NestJS ou similar) |
| Infra | **AWS / GCP** |

### Compartilhado

| Serviço | Uso |
|---------|-----|
| **Resend** | E-mails transacionais |
| **Sentry** | Error tracking |
| **Vitest** | Testes unitários |
| **GitHub** | Versionamento + CI/CD |

---

## Estrutura do Repositório

```
agis-group/
├── README.md              ← Você está aqui
├── Laurus-Cloud/          ← Laurus Cloud (B2B — PMS/POS SaaS)
├── Salv/                  ← Salv (B2C — Finanças Pessoais)
└── docs/                  ← Posts, estratégias e decisões públicas
```

---

## Para Quem é Este Repositório

| Audiência | O que encontrar |
|-----------|-----------------|
| **Desenvolvedores** | Arquitetura, stack, padrões de código, como contribuir |
| **Parceiros** | Visão de produto, roadmap, capacidades técnicas |
| **Investidores** | Modelo de negócio, TAM, estratégia de crescimento |
| **Time interno** | Fonte de verdade do ecossistema, decisões documentadas |

---

<p align="center">
  <sub>
    &copy; 2024–2026 Agis Group. Todos os direitos reservados.<br/>
    Construído com convicção, café e TypeScript.
  </sub>
</p>
