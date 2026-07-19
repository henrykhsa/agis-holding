<p align="center">
  <img src="https://img.shields.io/badge/Status-Em%20Desenvolvimento-blue?style=for-the-badge" alt="Status" />
  <img src="https://img.shields.io/badge/Licença-Proprietária-red?style=for-the-badge" alt="Licença" />
  <img src="https://img.shields.io/badge/Versão-0.1.0-green?style=for-the-badge" alt="Versão" />
</p>

<h1 align="center">Laurus Cloud</h1>

<p align="center">
  <strong>Plataforma SaaS de Gestão Hoteleira — PMS + POS</strong><br/>
  Solução modular multi-tenant para operações de front-office hoteleiro e ponto de venda (F&B).
</p>

<p align="center">
  <em>Desenvolvido e mantido pela <strong>Agis Group</strong></em>
</p>

---

## Indice

- [Visao Geral](#visao-geral)
- [Stack Tecnologico](#stack-tecnologico)
- [Arquitetura](#arquitetura)
- [Modulos e Estado Atual](#modulos-e-estado-atual)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Guia de Execucao Local](#guia-de-execucao-local)
- [Variaveis de Ambiente](#variaveis-de-ambiente)
- [Seguranca](#seguranca)
- [Roadmap](#roadmap)
- [Contribuicao](#contribuicao)

---

## Visao Geral

**Laurus Cloud** e uma plataforma SaaS modular que combina um **PMS (Property Management System)** completo com um modulo de **POS (Point of Sale)**, construida sobre uma unica aplicacao Next.js com arquitetura multi-tenant.

### Modelo de Tenancy

- Cada cliente da Laurus e representado por um **Tenant** (tabela `auth.laurus_company`) — raiz de isolamento de dados.
- Todo registro operacional carrega `tenantId`.
- `pms.Hotel` representa uma unidade operacional PMS subordinada ao Tenant (propriedade fisica).
- Clientes finais do PMS vivem em `pms.Guest` e `pms.Company`.
- Clientes do POS vivem em `pos.Customer`.

### Modulos Operacionais

| Modulo | Descricao |
|--------|-----------|
| **PMS** | Reservas, check-in/check-out, governanca de quartos, financeiro, auditoria noturna |
| **POS** | Ponto de venda, mesas, pedidos (F&B), integracao com folio PMS |

---

## Stack Tecnologico

### Frontend & Backend

| Tecnologia | Versao | Uso |
|------------|--------|-----|
| **Next.js** | 16.x | App Router, React Server Components |
| **React** | 19 | UI com React DOM |
| **TypeScript** | 5 | Tipagem estatica em todo o projeto |
| **Tailwind CSS** | 4.x | Estilizacao utilitaria |

### Banco de Dados & ORM

| Tecnologia | Versao | Uso |
|------------|--------|-----|
| **PostgreSQL** | 14+ | Banco relacional com multi-schema (`auth`, `pms`, `pos`) |
| **Prisma** | 5.x | ORM com schemas fragmentados por dominio |

### Autenticacao

| Tecnologia | Uso |
|------------|-----|
| **NextAuth.js 4.x** | JWT com `CredentialsProvider` e `PrismaAdapter` |
| **bcryptjs** | Hashing de senhas |

### Bibliotecas Principais

| Biblioteca | Uso |
|------------|-----|
| **date-fns** | Manipulacao de datas |
| **@hello-pangea/dnd** | Drag-and-drop (board de governanca) |
| **Resend** | E-mails transacionais |
| **react-icons** | Icones |
| **Vitest** | Testes unitarios |

### Infraestrutura

| Servico | Uso |
|---------|-----|
| **Vercel** | Hosting e deploy continuo |
| **Vercel Analytics** | Analise de uso |
| **Vercel Speed Insights** | Monitoramento de performance |

---

## Arquitetura

Arquitetura **modular monolitica** organizada por dominios de negocio, com separacao clara de camadas.

```
┌──────────────────────────────────────────────────────────────┐
│                       LAURUS CLOUD                           │
├──────────────────────────────────────────────────────────────┤
│  AUTH & RBAC       │  PORTAL (Tenants)  │  MULTI-TENANT      │
├──────────────────────────────────────────────────────────────┤
│  RESERVAS (PMS)    │  FINANCEIRO        │  GOVERNANCA        │
├──────────────────────────────────────────────────────────────┤
│  POS(Mesas/Pedidos)│  CRM (Hospedes)    │  CONFIGURACOES     │
├──────────────────────────────────────────────────────────────┤
│  AUDITORIA         │  TURNOS DE CAIXA   │  NIGHT AUDIT       │
└──────────────────────────────────────────────────────────────┘
```

### Diagrama de Alto Nivel

```
Navegador → Next.js App Router (Frontend + Backend)
              → Prisma Client → PostgreSQL (schemas: auth, pms, pos)
              → NextAuth.js (JWT)
```

### Principios de Design

1. **Tenant como raiz** — Nenhuma query operacional acessa dados sem filtro `tenantId`.
2. **Server-First** — React Server Components para performance; Client Components apenas para interatividade.
3. **Modulos ativados por `activeModules`** — Flags `hasPMS`/`hasPOS` derivam de `Tenant.activeModules`.
4. **Auditoria total** — Acoes criticas logadas em `pms.AuditLog`.
5. **RBAC contextual** — Cargo efetivo resolvido por `tenantId`, nunca de campo global.

---

## Modulos e Estado Atual

### Implementados e Funcionais

| Modulo | Funcionalidades |
|--------|-----------------|
| **Portal Admin** | CRUD de Tenants, vinculo usuario/tenant, selecao de tenant ativo |
| **Reservas (PMS)** | Tape chart (14 dias), calendário mensal, criacao/edicao, busca, check-in/check-out, atribuicao de quarto |
| **Financeiro** | Folio por reserva, lancamentos (CHARGE/PAYMENT), soft-delete, catalogo de produtos, metodos de pagamento com taxas de parcelamento |
| **Turnos de Caixa** | Abertura/fechamento, sangrias/suprimentos, contagem às cegas, auditoria de transacoes |
| **Hospedes/CRM** | CRUD de Guest (CPF) e Company (CNPJ), busca por tenant |
| **Governanca** | Board de status de quartos (CLEAN, DIRTY, INSPECT, OUT_OF_ORDER, status customizados) |
| **POS** | Gestao de mesas, criacao de pedidos, itens, pagamento, integracao POS → Folio PMS |
| **Auditoria Noturna** | Avanco de `businessDate`, validacao de check-ins pendentes |
| **Configuracoes** | CRUD de quartos, categorias, planos tarifarios, impostos, produtos, usuarios |

### Parcialmente Implementados

| Item | Status |
|------|--------|
| `pos.Customer` (CRUD UI) | Sem tela |
| `DailyRate` (tarifas flutuantes) | Schema existe, sem actions/UI |
| `Booking.source`, `adults`, `children` | Campos no schema, sem captura no form |
| `AuditLog` em todas actions criticas | Parcial (Night Audit e bloqueio de quarto OK) |

---

## Estrutura do Projeto

```
/
├── prisma/
│   ├── schema/              # Schemas Prisma fragmentados por dominio
│   │   ├── auth.prisma      # Tenant, User, Membership, Lead
│   │   ├── pms.prisma       # Hotel, Room, Booking, Folio, Transaction, ...
│   │   └── pos.prisma       # Customer, POSTable, POSOrder, POSOrderItem
│   ├── schema.tmp.prisma    # Schema unificado gerado (nao editar)
│   └── migrations/          # Migracoes SQL versionadas
│
├── src/
│   ├── app/                 # App Router do Next.js
│   │   ├── api/             # Rotas API (NextAuth, leads, session-valid)
│   │   └── app/             # Aplicacao protegida (/app/*)
│   │       ├── layout.tsx   # Layout raiz — contexto do Tenant
│   │       ├── dashboard/
│   │       ├── portal/      # Gestao de Tenants e usuarios
│   │       ├── modules/     # Selecao de modulo ativo
│   │       ├── availability/
│   │       ├── bookings/
│   │       ├── guests/
│   │       ├── financial/
│   │       ├── front-desk/
│   │       ├── housekeeping/
│   │       ├── governanca/
│   │       ├── pos/
│   │       ├── inventory/
│   │       └── settings/
│   │
│   ├── modules/             # Logica de negocio por dominio
│   │   ├── bookings/        # Reservas: tape chart, check-in/out, busca
│   │   ├── dashboard/       # KPIs e estatisticas
│   │   ├── finance/         # Lancamentos, pagamentos, turnos, auditoria
│   │   ├── guests/          # CRM de hospedes e empresas
│   │   ├── housekeeping/    # Board de governanca
│   │   ├── portal/          # Actions de gestao de Tenants
│   │   ├── pos/             # Ponto de venda
│   │   ├── settings/        # Configuracoes operacionais
│   │   └── shifts/          # Turnos de caixa
│   │
│   ├── shared/
│   │   ├── components/      # MainLayout, Sidebar, Providers
│   │   └── lib/
│   │       ├── db.ts                # Singleton Prisma Client
│   │       ├── auth.ts              # Configuracao NextAuth
│   │       ├── tenant-context.ts    # getActiveTenantContext, setTenantCookies
│   │       ├── server-rbac.ts       # requireTenantPermission, requirePagePermission
│   │       ├── permissions.ts       # Matriz PERMISSIONS + hasPermission()
│   │       ├── roles.ts             # getEffectiveRoleForTenant
│   │       ├── audit.ts             # createAuditLog()
│   │       └── password-*.ts        # Politica e reset de senha
│   │
│   └── types/
│       └── next-auth.d.ts   # Extensao de tipos NextAuth
│
├── scripts/                 # Utilitarios de desenvolvimento
├── docs/                    # Documentacao tecnica
├── middleware.ts            # Middleware Edge: auth + redirecionamento por modulo
└── package.json
```

### Fluxo de Dependencias

```
middleware.ts          (Edge — sem acesso ao Prisma)
    ↓
src/app/app/layout.tsx (Server Component — contexto global de UI)
    ↓
src/app/app/*/page.tsx (Server Components — paginas)
    ↓
src/modules/*/         (Logica de negocio + actions)
    ↓
src/shared/lib/        (Infra compartilhada: db, auth, rbac, permissions)
    ↓
prisma/                (ORM + schema + migrations)
```

---

## Guia de Execucao Local

### Pre-requisitos

| Ferramenta | Versao Minima |
|------------|---------------|
| Node.js | 20+ |
| PostgreSQL | 14+ |
| npm | 9.x+ |

### 1. Clonar o Repositorio

```bash
git clone https://github.com/agis-group/laurus-cloud.git
cd laurus-cloud
```

### 2. Instalar Dependencias

```bash
npm install
```

### 3. Configurar Variaveis de Ambiente

```bash
cp .env.example .env
```

### 4. Configurar Banco de Dados

```bash
# Gerar schema unificado + Prisma Client
npm run postinstall

# Executar migracoes
npx prisma migrate dev
```

> O banco requer a extensao `unaccent`. O usuario precisa de permissao `CREATE EXTENSION` ou um DBA deve habilita-la previamente.

### 5. Bootstrap (primeiro usuario)

```bash
# Via Prisma Studio
npx prisma studio
```

Criar: 1 Tenant → 1 User (com senha bcrypt) → 1 Membership (role OWNER/ADMIN).

### 6. Iniciar o Servidor

```bash
npm run dev
```

Acesse: `http://localhost:3000`

---

## Variaveis de Ambiente

```env
# Banco de Dados
DATABASE_URL="postgresql://user:password@localhost:5432/laurus_db"

# Autenticacao
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="gerar-com-openssl-rand-base64-32"

# Aplicacao
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_APP_NAME="Laurus Cloud"

# E-mail Transacional
RESEND_API_KEY="re_xxxxxxxxxxxx"
EMAIL_FROM="noreply@laurus.cloud"
```

---

## Seguranca

### Autenticacao

- JWT com chave de invalidacao derivada do hash da senha.
- Sessoes anteriores invalidadas apos redefinicao de senha.
- Recuperacao de senha com token aleatorio, hash SHA-256, expiracao de 1h.
- Politica de senha: min 8 chars, maiuscula, minuscula, numero, caractere especial.

### RBAC Contextual

O cargo efetivo e resolvido por `tenantId` — um usuario pode ter roles diferentes em Tenants distintos.

**16 roles disponiveis:** SUPER_ADMIN, OWNER, ADMIN, MANAGER, EVENT_MANAGER, AUDITOR, RESERVATIONS, RECEPTIONIST, CONCIERGE, HOUSEKEEPER, MAINTENANCE, SECURITY, WAITER, KITCHEN, VALET.

**Grupos de acesso:**

| Grupo | Roles |
|-------|-------|
| Alta Gestao | SUPER_ADMIN, OWNER, ADMIN, MANAGER |
| Administrativo | EVENT_MANAGER, AUDITOR, RESERVATIONS |
| Recepcao | RECEPTIONIST, CONCIERGE, VALET |
| Operacional | HOUSEKEEPER, MAINTENANCE, SECURITY |
| A&B | WAITER, KITCHEN |

### Middleware Edge

- Protege todas as rotas `/app/*`
- Verifica token JWT
- Le cookies `tenant_has_pms`/`tenant_has_pos` para redirecionar ao modulo correto
- Redireciona nao-autenticados para `/app/login`

---

## Roadmap

### Curto Prazo (em andamento)

- [ ] Rate limit em forgot-password e reset-password
- [ ] Completar AuditLog em todas actions criticas
- [ ] Tela CRUD de `pos.Customer`
- [ ] Full Text Search com suporte a acentos (extensao `unaccent`)
- [ ] Captura de `source`, `adults`, `children` nas reservas
- [ ] Remover logs de console em producao

### Medio Prazo

- [ ] Motor de tarifas flutuantes (`DailyRate`)
- [ ] Limite de desconto por cargo
- [ ] Estorno com politica temporal
- [ ] Fluxo completo de tarefas de governanca e manutencao
- [ ] Relatorios diarios por perfil de acesso
- [ ] API publica documentada (OpenAPI 3.1)
- [ ] Sistema de notificacoes em tempo real

### Longo Prazo

- [ ] MFA para Alta Gestao
- [ ] Integracao com channel managers (OTAs)
- [ ] App mobile (React Native)
- [ ] Multi-idioma (PT-BR / EN / ES)
- [ ] KDS real para cozinha
- [ ] Dashboard executivo multi-unidade
- [ ] Marketplace de integracoes (ERP, fiscal)

---

## Rotas Principais

| Rota | Modulo | Descricao |
|------|--------|-----------|
| `/app/dashboard` | PMS | Dashboard com KPIs |
| `/app/availability` | PMS | Tape Chart + calendario mensal |
| `/app/bookings` | PMS | Lista e busca de reservas |
| `/app/front-desk` | PMS | Operacoes de recepcao |
| `/app/governanca` | PMS | Board de governanca |
| `/app/guests` | PMS | CRM de hospedes e empresas |
| `/app/financial` | PMS | Visao financeira + metodos de pagamento |
| `/app/financial/payments` | PMS | Gestao de turnos de caixa |
| `/app/pos` | POS | Ponto de venda |
| `/app/inventory` | POS | Inventario |
| `/app/settings` | PMS/POS | Configuracoes |
| `/app/portal` | Admin | Gestao de Tenants |
| `/app/modules` | — | Selecao de modulo ativo |

---

## Contribuicao

Projeto **proprietario** da Agis Group. Contribuicoes restritas a equipe interna.

### Fluxo de Desenvolvimento

```
main ← staging ← feature/LAURUS-XXX-descricao
```

1. Branch a partir de `staging`: `feature/LAURUS-{ticket}-descricao`
2. Commits seguindo [Conventional Commits](https://www.conventionalcommits.org/)
3. Pull Request para `staging`
4. Review de pelo menos 1 membro senior
5. QA valida em staging antes do release para `main`

### Padroes de Codigo

- **Linting:** ESLint com config Next.js
- **Tipagem:** TypeScript strict
- **Commits:** `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`
- **Testes:** Vitest

---

## Documentacao Tecnica

Para detalhes aprofundados, consulte a pasta `docs/`:

- [Arquitetura do Sistema](./docs/ARCHITECTURE.md)
- [Funcionalidades Completas](./docs/SYSTEM_ARCHITECTURE_AND_FEATURES.md)
- [Estrutura do Projeto](./docs/PROJECT_STRUCTURE.md)
- [Matriz RBAC](./docs/RBAC_MATRIX.md)
- [Status de Seguranca](./docs/SECURITY_STATUS.md)
- [Roadmap de Melhorias](./docs/SYSTEM_IMPROVEMENTS_ROADMAP.md)
- [Debito Tecnico](./docs/TECH_DEBT_AND_TODO.md)

---

<p align="center">
  <sub>
    &copy; 2024–2026 Agis Group. Todos os direitos reservados.<br/>
    <strong>Laurus Cloud</strong> — Plataforma SaaS de gestao hoteleira.
  </sub>
</p>
