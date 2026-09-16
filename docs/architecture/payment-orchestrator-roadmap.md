# Payment Orchestrator — Roadmap Arquitetural

> Microserviço de orquestração de pagamentos do ecossistema AGIS Holding.  
> Responsável por abstrair gateways externos, maximizar margem financeira e garantir alta disponibilidade de cobrança para todos os tenants conectados ao Laurus Cloud (e futuros clientes B2B).

---

## 1. Visão Geral e Arquitetura Desacoplada

### Princípio Fundamental

O Payment Orchestrator é um **microserviço independente** — possui seu próprio ciclo de deploy, banco de dados, domínio de rede e políticas de segurança. O Laurus Cloud (ou qualquer outro consumer) **nunca se comunica diretamente** com gateways de pagamento; toda interação financeira passa exclusivamente pelo Orchestrator.

### Fluxo de Comunicação

```
┌─────────────┐       PaymentIntent        ┌──────────────────────┐
│ Laurus Cloud│ ─────────────────────────▶  │ Payment Orchestrator │
│  (Consumer) │ ◀───────── Result ───────── │    (Microserviço)    │
└─────────────┘                             └──────────┬───────────┘
                                                       │
                                          ┌────────────┼────────────┐
                                          ▼            ▼            ▼
                                     ┌────────┐  ┌──────────┐  ┌─────────┐
                                     │ Stripe │  │ Mercado  │  │ Pagar.me│
                                     │        │  │  Pago    │  │         │
                                     └────────┘  └──────────┘  └─────────┘
```

### Contrato de Entrada — `PaymentIntent`

O consumer envia uma intenção de pagamento contendo apenas dados de alto nível:

| Campo               | Tipo     | Descrição                                      |
|---------------------|----------|------------------------------------------------|
| `tenant_id`         | UUID     | Identificador do tenant (restaurante/hotel)    |
| `amount_cents`      | int64    | Valor em centavos (ex: 5990 = R$ 59,90)        |
| `currency`          | string   | ISO 4217 (BRL, USD, EUR)                       |
| `payment_method`    | object   | Token de pagamento (cartão tokenizado, PIX, boleto) |
| `card_brand`        | string?  | Bandeira do cartão (visa, mastercard, elo...)   |
| `idempotency_key`   | string   | Chave de idempotência para retry seguro         |
| `metadata`          | object?  | Dados adicionais livres (order_id, etc.)        |

### Benefícios do Desacoplamento

- **Isolamento de PCI Scope** — O Laurus Cloud permanece fora do escopo PCI DSS; toda manipulação sensível fica contida no Orchestrator.
- **Evolução independente** — Novos gateways podem ser adicionados sem alterar uma linha de código no consumer.
- **Reutilização** — Qualquer novo produto B2B da holding (Salv, futuros SaaS) pode consumir o mesmo serviço.
- **Testabilidade** — O contrato via `PaymentIntent` permite mocking completo em ambientes de staging.

---

## 2. Smart Routing (Roteamento Inteligente)

### Objetivo

Maximizar o **spread** (diferença entre a taxa cobrada do tenant e o custo real de processamento) roteando cada transação para o gateway com menor custo efetivo naquele instante.

### Engine de Decisão — Fluxo

```
PaymentIntent recebido
       │
       ▼
┌──────────────────────┐
│ 1. Identificar Brand │  (visa, mastercard, elo, amex, hipercard)
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│ 2. Consultar Tabela de Taxas por Gateway │
│    (rate_tables / cache em Redis)        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│ 3. Aplicar Regras de Negócio:        │
│    - Taxa efetiva para brand+parcelas│
│    - Prioridade manual (override)    │
│    - Health score do gateway         │
│    - Limite de volume contratual     │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────┐
│ 4. Selecionar Gateway    │
│    com menor custo final │
└──────────────────────────┘
```

### Tabela de Taxas (`rate_tables`)

Estrutura de configuração por gateway/brand/parcela:

```json
{
  "gateway": "mercado_pago",
  "brand": "visa",
  "installments": 1,
  "mdr_percent": 3.49,
  "fixed_fee_cents": 0,
  "updated_at": "2026-08-01T00:00:00Z"
}
```

As tabelas são atualizadas periodicamente (via backoffice ou API dos gateways) e cacheadas em Redis para consulta em tempo real (latência < 5ms).

### Critérios de Roteamento (por prioridade)

1. **Menor MDR efetivo** para a combinação `brand + installments`.
2. **Health Score** — gateways com alta taxa de erro recente perdem prioridade temporariamente.
3. **Override manual** — permite forçar roteamento para um gateway específico via configuração do tenant (útil para contratos comerciais exclusivos).
4. **Volume cap** — respeita limites de volume mensal contratados com cada gateway.

---

## 3. Alta Disponibilidade e Fallback (Anti-Queda)

### Premissa

Nenhuma transação legítima deve ser perdida por falha de infraestrutura de terceiros. O lojista não pode perder uma venda porque um gateway ficou indisponível.

### Mecânica de Contingência

```
Tentativa no Gateway Primário
       │
       ├── HTTP 2xx → Sucesso, retornar resultado
       │
       ├── HTTP 5xx / Timeout (> 8s) / Connection Refused
       │         │
       │         ▼
       │   Incrementar failure_count no Circuit Breaker
       │         │
       │         ▼
       │   Retry no Gateway Secundário (próximo na fila de prioridade)
       │         │
       │         ├── HTTP 2xx → Sucesso (log de fallback ativado)
       │         │
       │         ├── Falha → Tentar Gateway Terciário (se houver)
       │         │
       │         └── Todos falharam → Retornar erro ao consumer
       │                              com código `PAYMENT_UNAVAILABLE`
       │
       └── HTTP 4xx (erro de validação) → NÃO fazer fallback
                                           (erro do payload, não do gateway)
```

### Circuit Breaker

Implementação baseada em estados:

| Estado     | Comportamento                                                        |
|------------|----------------------------------------------------------------------|
| **Closed** | Tráfego normal para o gateway.                                       |
| **Open**   | Gateway removido da fila por N segundos após X falhas consecutivas.  |
| **Half-Open** | Permite 1 request de teste; se sucesso, retorna a Closed.        |

Parâmetros configuráveis:

- `failure_threshold`: 3 falhas consecutivas (default)
- `recovery_timeout`: 30 segundos
- `timeout_ms`: 8000ms por request ao gateway

### Observabilidade

- Cada fallback gera um evento `payment.fallback.triggered` no barramento de eventos.
- Dashboard de saúde por gateway (uptime, latência p95, taxa de erro 1h/24h).
- Alertas automáticos via webhook quando um gateway entra em estado **Open**.

---

## 4. Segurança e PCI Compliance

### Tokenização

O Payment Orchestrator **nunca armazena** dados sensíveis de cartão (PAN, CVV, data de expiração) em qualquer banco de dados, cache ou log.

#### Fluxo de Tokenização

1. O frontend do tenant coleta os dados do cartão via **SDK client-side** do gateway (Stripe Elements, MP CardForm, etc.).
2. O SDK retorna um **token opaco** (ex: `tok_abc123`).
3. O consumer envia apenas o token no campo `payment_method` do `PaymentIntent`.
4. O Orchestrator usa o token para realizar a cobrança via API do gateway correspondente.
5. Após a transação, o token é descartado; apenas referências de transação (`transaction_id`) são armazenadas.

### Medidas de Segurança

| Camada            | Controle                                                             |
|-------------------|----------------------------------------------------------------------|
| **Transporte**    | TLS 1.3 obrigatório em todas as comunicações (inbound e outbound).   |
| **Autenticação**  | mTLS entre consumer e Orchestrator; API Keys rotacionáveis por tenant.|
| **Autorização**   | RBAC — cada tenant opera exclusivamente sobre seus próprios recursos. |
| **Armazenamento** | Nenhum dado PCI em repouso. Secrets de gateway em Vault (HashiCorp/AWS Secrets Manager). |
| **Logs**          | Redação automática (masking) de qualquer campo sensível antes de persistir. |
| **Auditoria**     | Registro imutável (append-only) de todas as operações com hash encadeado. |

### Escopo PCI DSS

Com a tokenização client-side, o Orchestrator se enquadra no **SAQ-A** (menor nível de compliance), já que dados de cartão nunca trafegam pelos nossos servidores em formato legível.

---

## 5. Estratégia de Monetização (Freemium via Transação)

### Modelo de Negócio

A holding cobra uma **taxa fixa por transação** do tenant, enquanto o Smart Routing garante que o custo real de processamento seja inferior à taxa cobrada. A diferença é o lucro por transação (spread).

### Exemplo Numérico

```
Transação de R$ 100,00 (cartão Visa, 1x)

┌─────────────────────────────────────────────────────┐
│ Taxa cobrada do tenant (fixa):          5,00%       │
│ Custo efetivo (Mercado Pago, Visa 1x):  3,49%       │
│                                                     │
│ Spread por transação:                   1,51%       │
│ Lucro nesta transação:                  R$ 1,51     │
└─────────────────────────────────────────────────────┘
```

### Requisitos Técnicos para Suporte ao Modelo

| Requisito                        | Implementação                                              |
|----------------------------------|------------------------------------------------------------|
| Taxa configurável por tenant     | Campo `tenant_fee_percent` na tabela de configuração.      |
| Split de pagamento               | Usar APIs de split/marketplace dos gateways quando disponível (Stripe Connect, MP Marketplace). |
| Registro de spread               | Cada transação registra `tenant_fee`, `gateway_cost`, `spread` para reconciliação financeira. |
| Billing por período              | Relatório consolidado mensal por tenant com total de fees.  |
| Suporte a planos                 | Tenants em planos superiores podem ter taxas reduzidas (ex: 4,5% em vez de 5%). |

### Escalabilidade do Modelo

- Quanto maior o volume transacionado, maior o poder de negociação com gateways (taxas regressivas por volume).
- Novos gateways com taxas mais competitivas podem ser integrados sem impacto no contrato com o tenant.
- O modelo funciona independentemente do produto consumer (Laurus, Salv, etc.).

---

## 6. Fase 2 — Evolução para Adquirente Direto (Laurus Pay)

### Visão

Na segunda fase evolutiva, a AGIS Holding deixa de depender exclusivamente de gateways terceiros e passa a operar como **adquirente/sub-adquirente direto**, com conexão própria às bandeiras via protocolo ISO 8583 e liquidação regulada pelo BACEN. O produto assume a marca **Laurus Pay**.

### Arquitetura de Processamento Próprio

```
┌─────────────┐       PaymentIntent        ┌──────────────────────┐
│ Laurus Cloud│ ─────────────────────────▶  │ Payment Orchestrator │
│  (Consumer) │ ◀───────── Result ───────── │    (Microserviço)    │
└─────────────┘                             └──────────┬───────────┘
                                                       │
                                         Smart Routing Engine
                                                       │
                              ┌─────────────────────────┼──────────────────────────┐
                              ▼                         ▼                          ▼
                  ┌───────────────────┐       ┌──────────────┐          ┌──────────────┐
                  │   Laurus Pay      │       │    Stripe    │          │ Mercado Pago │
                  │ (Processamento    │       │ (Contingência│          │ (Contingência│
                  │  Próprio)         │       │  Secundária) │          │  Terciária)  │
                  │                   │       └──────────────┘          └──────────────┘
                  │  ISO 8583 ←→ Redes│
                  │  BACEN Settlement │
                  └───────────────────┘
```

### Diretriz de Alta Disponibilidade e Fallback Ativo

> **Regra Inviolável:** Os gateways terceiros (Stripe, Mercado Pago, Pagar.me) **NÃO serão removidos** da arquitetura quando o processamento próprio estiver ativo. Eles são permanentemente mantidos como **rotas de contingência automáticas**.

#### Comportamento do Smart Routing na Fase 2

O engine de roteamento passa a operar com uma nova hierarquia de prioridade:

| Prioridade | Rota                    | MDR Efetivo | Papel                          |
|------------|-------------------------|-------------|--------------------------------|
| 1 (máxima) | Laurus Pay (próprio)    | ~0%*        | Processamento primário         |
| 2          | Stripe                  | Variável    | Fallback automático secundário |
| 3          | Mercado Pago            | Variável    | Fallback automático terciário  |
| 4          | Pagar.me                | Variável    | Fallback automático quaternário|

*\* Custo operacional interno (infraestrutura, compliance), sem MDR de terceiros.*

#### Mecânica de Fallback Silencioso

```
PaymentIntent recebido
       │
       ▼
┌────────────────────────────────────┐
│ Smart Routing: priorizar Laurus Pay│
│ (taxa zero → máximo spread)        │
└──────────┬─────────────────────────┘
           │
           ▼
┌────────────────────────────────────┐
│ Laurus Pay - Processamento Direto  │
│ (ISO 8583 → Redes de Bandeira)     │
└──────────┬─────────────────────────┘
           │
           ├── Aprovado → Retornar sucesso
           │
           ├── Recusa Técnica / Timeout / Instabilidade Interna
           │         │
           │         ▼
           │   ┌──────────────────────────────────────────────┐
           │   │ FALLBACK INSTANTÂNEO E SILENCIOSO             │
           │   │                                              │
           │   │ • Transparente para o usuário final          │
           │   │ • Transparente para o tenant/lojista         │
           │   │ • Sem retry visível, sem redirecionamento    │
           │   │ • Latência adicional < 200ms                 │
           │   └──────────────────┬───────────────────────────┘
           │                      │
           │                      ▼
           │              Stripe (Gateway Secundário)
           │                      │
           │                      ├── Aprovado → Sucesso (flag: fallback_route=stripe)
           │                      │
           │                      └── Falha → Mercado Pago → Pagar.me → ...
           │
           └── Recusa de Negócio (saldo, fraude) → NÃO fazer fallback
                                                    (retornar decline ao consumer)
```

#### Critérios para Ativação de Fallback

| Cenário                           | Ação                                | Fallback? |
|-----------------------------------|-------------------------------------|-----------|
| Timeout interno (> 5s)            | Redirecionar para gateway externo   | Sim       |
| Erro de comunicação com rede      | Redirecionar para gateway externo   | Sim       |
| HTTP 5xx do motor próprio         | Redirecionar para gateway externo   | Sim       |
| Degradação de latência (p95 > 3s) | Circuit breaker → fallback proativo | Sim       |
| Manutenção programada             | Desvio automático pré-agendado      | Sim       |
| Recusa por saldo insuficiente     | Retornar decline                    | Não       |
| Recusa por suspeita de fraude     | Retornar decline                    | Não       |
| Erro de validação (4xx)           | Retornar erro ao consumer           | Não       |

#### Meta de Uptime

A combinação de processamento próprio + fallback ativo para múltiplos gateways terceiros garante o target de **99.999% de disponibilidade** (menos de 5.26 minutos de indisponibilidade por ano) nas transações do cliente.

```
Cálculo de disponibilidade composta:

P(indisponibilidade total) = P(falha_laurus) × P(falha_stripe) × P(falha_mp) × P(falha_pagarme)

Se cada rota tem 99.9% uptime individual:
P(todas falham) = 0.001 × 0.001 × 0.001 × 0.001 = 0.000000000001 (desprezível)

Uptime efetivo > 99.999%
```

#### Impacto na Monetização

| Rota utilizada   | Custo para a holding | Spread (taxa 5% do tenant) |
|------------------|----------------------|----------------------------|
| Laurus Pay       | ~0% (custo fixo ops) | ~5,00% (máximo)            |
| Stripe (fallback)| ~2,9%                | ~2,10%                     |
| Mercado Pago     | ~3,49%               | ~1,51%                     |

O Smart Routing maximiza o uso da rota própria para capturar spread máximo, recorrendo a gateways externos apenas quando necessário para preservar a transação.

#### Observabilidade Específica da Fase 2

- Métrica `laurus_pay.own_route_ratio` — percentual de transações processadas via motor próprio (target: > 95%).
- Evento `payment.fallback.own_to_external` — rastreia cada vez que o motor próprio falha e um gateway externo assume.
- Alerta de degradação se `own_route_ratio` cair abaixo de 90% por mais de 15 minutos.
- Dashboard comparativo de custo: processamento próprio vs. gateways externos (economia mensal).

---

## Stack Tecnológica Sugerida

| Componente          | Tecnologia                         | Justificativa                           |
|---------------------|------------------------------------|-----------------------------------------|
| Runtime             | Node.js (NestJS) ou Go             | Alta performance para I/O bound         |
| Banco de dados      | PostgreSQL                         | ACID, auditabilidade, JSONB para metadata |
| Cache / Rate Tables | Redis                              | Latência sub-ms para decisões de roteamento |
| Secrets             | HashiCorp Vault / AWS Secrets Manager | Gestão segura de API keys             |
| Mensageria          | RabbitMQ / SQS                     | Eventos assíncronos (webhooks, reconciliação) |
| Observabilidade     | OpenTelemetry + Grafana            | Tracing distribuído, métricas, alertas  |
| CI/CD               | GitHub Actions + ArgoCD            | Deploy independente com rollback rápido |

---

## Próximos Passos

- [ ] Definir contrato OpenAPI 3.1 completo para o endpoint `/v1/payment-intents`
- [ ] Criar PoC de integração com Stripe e Mercado Pago
- [ ] Implementar circuit breaker com testes de chaos engineering
- [ ] Configurar ambiente PCI-compliant isolado (VPC dedicada)
- [ ] Desenvolver dashboard de roteamento e saúde dos gateways
- [ ] Negociar taxas comerciais com gateways para volume projetado

---

*Documento mantido por: Equipe de Arquitetura — AGIS Holding*  
*Última atualização: Agosto 2026*
