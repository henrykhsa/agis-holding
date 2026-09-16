# Motor de Integração de Delivery

> RFC Técnica — Produto Independente (API as a Service)  
> Status: **Proposta Arquitetural**  
> Autor: Engenharia Agis Group  
> Data: Agosto 2026  
> Classificação: **Novo produto SaaS do ecossistema Agis Group**

---

## Sumário

- [Contexto e Motivação](#contexto-e-motivação)
- [Visão de Produto](#visão-de-produto)
- [Posicionamento no Ecossistema](#posicionamento-no-ecossistema)
- [Arquitetura Proposta](#arquitetura-proposta)
- [Contrato de Integração (Consumer API)](#contrato-de-integração-consumer-api)
- [Fluxo de Dados](#fluxo-de-dados)
- [Comunicação em Tempo Real](#comunicação-em-tempo-real)
- [Impressão Térmica via Navegador](#impressão-térmica-via-navegador)
- [Unificação de Status e Ciclo de Vida do Pedido](#unificação-de-status-e-ciclo-de-vida-do-pedido)
- [Modelagem de Dados](#modelagem-de-dados)
- [Integrações de Plataforma](#integrações-de-plataforma)
- [Considerações de Segurança](#considerações-de-segurança)
- [Riscos e Mitigações](#riscos-e-mitigações)
- [Infraestrutura e Deploy](#infraestrutura-e-deploy)
- [Referências](#referências)
- [Brainstorming de Produto](#brainstorming-de-produto)

---

## Contexto e Motivação

O mercado brasileiro de food delivery movimenta R$ 50B+/ano (2025) e depende estruturalmente de aplicativos desktop nativos (Windows) fornecidos por cada marketplace (iFood Gestor, 99Food Manager, etc.) para que restaurantes recebam pedidos e imprimam cupons em impressoras térmicas locais.

Esse modelo impõe três restrições sistêmicas a qualquer operação de F&B que utilize sistemas de gestão cloud:

1. **Dependência de hardware Windows** — Exige um terminal desktop dedicado por plataforma. Operações que migraram para PDVs web perdem o benefício cloud-native ao manter máquinas legadas exclusivamente para delivery.
2. **Fragmentação operacional** — Cada marketplace opera em silo. O gestor monitora N janelas independentes, sem visão consolidada de pedidos, sem correlação com o financeiro do PDV e sem rastreabilidade no turno de caixa.
3. **Dados órfãos** — Receita de delivery não alimenta automaticamente relatórios de vendas, controle de estoque ou reconciliação fiscal do sistema de gestão principal. São operações fantasma do ponto de vista contábil.

Esse problema não é exclusivo do Laurus Cloud — afeta qualquer PDV web, ERP ou sistema de gestão do setor de food service. É um gap de infraestrutura do mercado, não de um produto.

---

## Visão de Produto

O Motor de Integração de Delivery será construído como um **produto SaaS independente** — um microserviço com API pública, operando como infraestrutura de integração agnóstica ao sistema de PDV consumidor.

### Proposta

> Uma **API as a Service** que centraliza, normaliza e roteia pedidos de múltiplas plataformas de delivery para qualquer sistema de PDV, ERP ou gestão que implemente o contrato de integração.

### Princípios de Produto

| Princípio | Implicação |
|-----------|------------|
| **Platform-agnostic** | O motor não conhece nem depende da estrutura interna do PDV consumidor. Se comunica exclusivamente via HTTP webhooks padronizados. |
| **Multi-tenant nativo** | Cada cliente (restaurante, rede, sistema de PDV parceiro) é um tenant isolado com credenciais, configurações e billing próprios. |
| **Marketplace extensível** | Novos marketplaces de delivery são adicionados via Adapter Pattern sem alterar o core. |
| **API-first** | Toda funcionalidade é exposta via API REST documentada (OpenAPI 3.1). Não existe UI acoplada — consumidores constroem sua própria experiência. |
| **Self-service onboarding** | Parceiros (sistemas de PDV) integram via documentação pública + API keys, sem intervenção manual. |

### Modelo de Negócio (Projeção)

| Fonte de Receita | Mecanismo |
|------------------|-----------|
| Assinatura por restaurante | Plano mensal por volume de pedidos processados |
| Revenue share por pedido | Fee fixa (R$ 0,10–0,30) por pedido roteado com sucesso |
| Tier enterprise para redes | SLA dedicado, suporte prioritário, custom adapters |
| White-label para PDVs parceiros | Markup sobre assinatura quando o motor é vendido embedded em outro produto |

---

## Posicionamento no Ecossistema

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AGIS GROUP                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   LAURUS CLOUD (B2B)       DELIVERY ENGINE (B2B2B)       SALV (B2C)          │
│   ─────────────────        ────────────────────          ──────────          │
│   PMS + POS SaaS           Hub de Integração de          Educação &          │
│   Gestão hoteleira         Delivery (API as a Service)   Gestão Financeira   │
│   Multi-tenant             Multi-tenant                  Freemium            │
│   Receita: assinaturas     Receita: volume + SaaS        Receita: ads/BaaS   │
│                                                                              │
│   ┌──────────────────────────────────────────────────┐                       │
│   │  Laurus Cloud = CLIENTE Nº 1 do Delivery Engine  │                       │
│   │  (consome a mesma API pública que qualquer PDV)  │                       │
│   └──────────────────────────────────────────────────┘                       │
│                                                                              │
│   Demais consumidores (mercado aberto):                                      │
│   ├── PDVs de terceiros (TOTVS, Linx, Stone)                                │
│   ├── ERPs de food service                                                   │
│   ├── Redes de franquias com sistema próprio                                 │
│   └── Agregadores regionais                                                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

O Laurus Cloud será o primeiro consumidor, validando o produto em produção antes da abertura ao mercado. Nenhuma lógica específica do Laurus existirá dentro do motor — a integração é feita exclusivamente pelo contrato público da API.

---

## Arquitetura Proposta

### Premissa: Isolamento Total

O Delivery Engine roda como um **serviço independente** — processo, banco de dados, domínio e infraestrutura separados de qualquer sistema consumidor. Essa decisão é inegociável por três razões:

1. **Blindagem de performance** — O tráfego de webhooks de marketplaces (especialmente iFood, que exige polling ativo a cada 30s por merchant) é imprevisível e pode gerar picos de carga. Esse tráfego não deve competir por recursos com o PDV do consumidor.
2. **Independência de deploy** — O motor evolui, escala e sofre deploys em cadência própria, sem coordenar releases com sistemas consumidores.
3. **Vendabilidade** — Um serviço acoplado ao Laurus não pode ser vendido a terceiros. O isolamento é pré-requisito comercial.

### Visão Geral dos Componentes

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    DELIVERY ENGINE (Microserviço Independente)                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────┐     ┌──────────────────────────┐     ┌─────────────────────────┐  │
│  │  iFood API   │────▶│                          │     │   Consumer Webhook      │  │
│  └──────────────┘     │  Ingestion Layer         │     │   Dispatcher            │  │
│  ┌──────────────┐     │  (Webhook Receivers +    │────▶│                         │  │
│  │  99Food API  │────▶│   Polling Workers)       │     │  POST padronizado para  │  │
│  └──────────────┘     │                          │     │  N sistemas de PDV      │  │
│  ┌──────────────┐     └──────────────────────────┘     └────────────┬────────────┘  │
│  │  Rappi API   │────▶         │                                    │               │
│  └──────────────┘              │                                    │               │
│                                ▼                                    ▼               │
│                    ┌──────────────────────┐         ┌───────────────────────────┐   │
│                    │  Normalization +     │         │  Realtime Gateway         │   │
│                    │  Persistence         │         │  (SSE/WebSocket p/ UI)    │   │
│                    │  (DeliveryOrder)     │         └───────────────────────────┘   │
│                    └──────────────────────┘                                          │
│                                                                                      │
│  ┌──────────────────────────┐     ┌──────────────────────────────────────────────┐  │
│  │  Status Sync (Outbound)  │     │  Management API (REST)                       │  │
│  │  Propaga ações do PDV    │     │  - Onboarding de merchants                   │  │
│  │  de volta ao marketplace │     │  - Configuração de integrações               │  │
│  └──────────────────────────┘     │  - Consulta de pedidos/eventos               │  │
│                                   │  - Ações (accept, reject, dispatch, cancel)  │  │
│                                   └──────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
         │                                              │
         │  Consome APIs dos                            │  Entrega pedidos e eventos
         │  marketplaces                                │  para sistemas consumidores
         ▼                                              ▼
┌─────────────────┐                        ┌──────────────────────────────────┐
│  iFood / 99Food │                        │  CONSUMIDORES                    │
│  Rappi / etc.   │                        │  ├── Laurus Cloud (Cliente Nº1)  │
└─────────────────┘                        │  ├── PDV Terceiro A              │
                                           │  ├── ERP Food Service B          │
                                           │  └── Sistema Próprio C           │
                                           └──────────────────────────────────┘
```

### Camadas

| Camada | Responsabilidade |
|--------|------------------|
| **Ingestion Layer** | Recebe webhooks ou executa polling contra APIs de delivery. Valida payload, deduplica, armazena evento raw. Opera por merchant registrado. |
| **Normalization** | Transforma o payload proprietário de cada plataforma em uma estrutura interna unificada (`DeliveryOrder`). |
| **Consumer Webhook Dispatcher** | Entrega o pedido normalizado via POST HTTP ao endpoint configurado pelo consumidor (PDV). Retry com exponential backoff. Garantia at-least-once. |
| **Realtime Gateway** | Canal SSE/WebSocket opcional para consumidores que precisam de push em tempo real (complementar ao webhook). |
| **Status Sync (Outbound)** | Quando o consumidor reporta uma ação (aceitar, despachar, cancelar) via Management API, propaga de volta à plataforma de origem. |
| **Management API** | API REST pública (OpenAPI 3.1) para CRUD de merchants, configurações, consultas e ações sobre pedidos. |

---

## Contrato de Integração (Consumer API)

O sistema consumidor (PDV, ERP, etc.) integra com o Delivery Engine de duas formas:

### 1. Recebimento de Pedidos (Inbound — Engine → Consumidor)

O consumidor registra um **webhook URL** no onboarding. O Engine faz POST nesse endpoint sempre que um evento relevante ocorre.

```
POST https://pdv-consumidor.com/api/delivery/incoming
Content-Type: application/json
X-Delivery-Engine-Signature: sha256=abc123...
X-Delivery-Engine-Event: order.created

{
  "event": "order.created",
  "timestamp": "2026-08-10T14:32:00Z",
  "merchant_id": "merch_abc123",
  "order": {
    "id": "dord_xyz789",
    "platform": "IFOOD",
    "external_id": "ifood-order-456",
    "display_code": "#1234",
    "status": "PENDING_ACCEPTANCE",
    "accept_deadline": "2026-08-10T14:37:00Z",
    "customer": {
      "name": "João Silva",
      "phone": "+5511999999999"
    },
    "delivery_address": {
      "street": "Rua Exemplo",
      "number": "123",
      "complement": "Apt 4",
      "neighborhood": "Centro",
      "city": "São Paulo",
      "zipcode": "01001-000",
      "latitude": -23.5505,
      "longitude": -46.6333
    },
    "items": [
      {
        "name": "X-Burger Duplo",
        "quantity": 2,
        "unit_price": 29.90,
        "total_price": 59.80,
        "notes": "Sem cebola",
        "modifiers": [
          { "name": "Bacon Extra", "quantity": 1, "price": 5.00 }
        ]
      }
    ],
    "subtotal": 64.80,
    "delivery_fee": 8.99,
    "discount": 5.00,
    "total": 68.79,
    "payment": {
      "method": "ONLINE",
      "paid_online": true,
      "change_for": null
    },
    "scheduled_for": null,
    "platform_metadata": {}
  }
}
```

**Eventos disponíveis:**

| Evento | Descrição |
|--------|-----------|
| `order.created` | Novo pedido recebido (requer ação de aceite) |
| `order.status_changed` | Mudança de status originada pela plataforma |
| `order.cancelled` | Cancelamento originado pela plataforma ou cliente |
| `order.updated` | Atualização de dados do pedido (raro) |

### 2. Ações sobre Pedidos (Outbound — Consumidor → Engine)

O consumidor executa ações via Management API:

```
POST https://api.delivery-engine.com/v1/orders/{order_id}/accept
Authorization: Bearer {api_key}
Content-Type: application/json

{
  "estimated_preparation_minutes": 25
}
```

**Endpoints de ação:**

| Método | Endpoint | Efeito |
|--------|----------|--------|
| POST | `/v1/orders/{id}/accept` | Confirma pedido na plataforma |
| POST | `/v1/orders/{id}/reject` | Rejeita com motivo obrigatório |
| POST | `/v1/orders/{id}/ready-for-pickup` | Marca como pronto para retirada |
| POST | `/v1/orders/{id}/dispatch` | Confirma despacho |
| POST | `/v1/orders/{id}/cancel` | Solicita cancelamento |

O Engine recebe a ação, propaga para a API da plataforma de origem e confirma o resultado ao consumidor na response.

---

## Fluxo de Dados

### Ingestion — Webhook vs. Polling

A estratégia de ingestion depende da API de cada plataforma:

| Plataforma | Modelo Suportado | Observações |
|------------|------------------|-------------|
| iFood | Polling (events endpoint) | iFood não envia webhooks ativamente. A aplicação deve consumir `/v3.0/events:polling` a cada N segundos e fazer ACK. |
| 99Food | Webhook | Envia POST para URL configurada no painel do restaurante. |
| Rappi | Webhook + Polling | Suporta ambos. Preferir webhook com fallback de polling. |

**Decisão arquitetural:** Implementar um **Polling Worker** assíncrono (background job via BullMQ ou similar) para plataformas que não suportam push. Para webhooks inbound (marketplaces → Engine), expor endpoints dedicados por plataforma em `/api/ingestion/webhook/{platform}`.

### Sequência Completa (Webhook Inbound)

```
Plataforma ──POST──▶ /api/ingestion/webhook/ifood (Delivery Engine)
                          │
                          ▼
                    Validação de assinatura (HMAC)
                          │
                          ▼
                    Persistência do evento raw (tabela delivery_events)
                          │
                          ▼
                    Normalização → DeliveryOrder interno
                          │
                          ▼
                    Consumer Webhook Dispatcher:
                    POST https://consumidor.com/api/delivery/incoming
                    (payload normalizado + assinatura)
                          │
                          ▼
                    Emit SSE/WebSocket (canal realtime opcional)
                          │
                          ▼
                    Consumidor (PDV) recebe, exibe alerta, imprime cupom
                          │
                          ▼
                    Consumidor aceita → POST /v1/orders/{id}/accept (Management API)
                          │
                          ▼
                    Engine propaga → PATCH plataforma (confirm)
```

### Sequência Completa (Polling)

```
Polling Worker (cron: cada 30s por merchant ativo)
       │
       ▼
GET /v3.0/events:polling (iFood)
       │
       ▼
Para cada evento novo:
  ├── ACK do evento na API da plataforma
  ├── Persistência do evento raw
  ├── Normalização → DeliveryOrder
  ├── Dispatch webhook ao consumidor
  ├── Emit SSE/WebSocket (canal realtime)
  └── Aguarda ação do consumidor via Management API
```

### Garantias de Entrega ao Consumidor

| Aspecto | Comportamento |
|---------|---------------|
| **Retry** | Exponential backoff (1s, 2s, 4s, 8s, 16s, 32s, 60s). Máximo 7 tentativas. |
| **Timeout** | 10s por tentativa. Se o consumidor não responder, conta como falha. |
| **Idempotência** | Header `X-Delivery-Engine-Delivery-Id` único por entrega. Consumidor pode deduplicar. |
| **Dead Letter** | Após esgotar retries, evento vai para dead letter queue. Visível no dashboard/API. Reprocessamento manual disponível. |
| **Assinatura** | HMAC-SHA256 do body com secret compartilhado. Header `X-Delivery-Engine-Signature`. |

---

## Comunicação em Tempo Real

> **Nota:** O canal realtime é uma feature complementar do Engine, não um substituto do webhook. Consumidores que precisam apenas de entrega garantida podem operar exclusivamente via webhook. O canal SSE/WebSocket atende consumidores que necessitam de latência sub-segundo para exibir alertas instantâneos ao operador.

### Escolha do Protocolo: WebSocket vs. SSE

| Critério | WebSocket | SSE (Server-Sent Events) |
|----------|-----------|--------------------------|
| Direção | Bidirecional | Unidirecional (server → client) |
| Reconexão automática | Manual | Nativa (EventSource API) |
| Compatibilidade com proxies/CDN | Pode ter problemas com Vercel Edge | Funciona nativamente via HTTP |
| Complexidade de infra | Requer servidor stateful ou serviço dedicado (Ably, Pusher, Socket.IO) | Stateless; compatível com serverless |
| Caso de uso | Chat, colaboração real-time | Notificações, feeds, alertas |

**Decisão:** Para o canal realtime do Engine — onde o fluxo primário é server → client (notificar o PDV de um novo pedido) — **SSE é a escolha pragmática** para o MVP. Não requer servidor WebSocket dedicado e tem reconexão nativa no browser.

Para ações do consumidor (aceitar, rejeitar), o client faz POST convencional na Management API do Engine. Não há necessidade de canal bidirecional.

**Alternativa para escala:** Se a volumetria exigir (centenas de merchants com conexões simultâneas), migrar para WebSocket via serviço gerenciado (Ably, Pusher, ou Soketi self-hosted). A interface do client deve abstrair o transporte para permitir swap transparente.

### Implementação SSE (Realtime Gateway do Engine)

```typescript
// GET /v1/stream?merchant_id=merch_abc123
// Header: Authorization: Bearer {api_key}
export async function GET(request: Request) {
  const encoder = new TextEncoder();
  const merchantId = extractMerchantId(request);

  const stream = new ReadableStream({
    start(controller) {
      const unsubscribe = eventBus.subscribe(merchantId, (event) => {
        const data = `data: ${JSON.stringify(event)}\n\n`;
        controller.enqueue(encoder.encode(data));
      });

      request.signal.addEventListener('abort', () => {
        unsubscribe();
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### Alerta Sonoro no Client

```typescript
// Ao receber evento "delivery:new_order" via EventSource
const audio = new Audio('/sounds/delivery-alert.mp3');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'NEW_ORDER') {
    audio.play().catch(() => {
      // Browser bloqueou autoplay — mostrar prompt para interação
    });
    showDeliveryNotification(data.order);
  }
};
```

> **Nota sobre autoplay:** Navegadores modernos bloqueiam `audio.play()` sem interação prévia do usuário. A solução é exigir que o operador clique em "Ativar alertas sonoros" ao abrir o POS pela primeira vez na sessão. Esse clique desbloqueia o `AudioContext` para a duração da sessão.

---

## Impressão Térmica via Navegador

> **Nota de posicionamento:** A impressão térmica é responsabilidade do sistema consumidor (PDV), não do Delivery Engine. No entanto, o Engine oferece um **Print Agent open-source** como componente opcional para consumidores que operam via browser e não possuem solução própria de impressão. O Laurus Cloud, como Cliente Nº 1, será o primeiro a utilizar esse agente.

Este é o desafio técnico mais significativo no lado do consumidor. Impressoras térmicas (Epson TM-T20, Elgin i9, Bematech MP-4200) usam protocolo ESC/POS e se conectam via USB ou Rede (Ethernet/Wi-Fi). Navegadores não possuem acesso nativo a drivers de impressão térmica com formatação ESC/POS.

### Abordagens Avaliadas

#### 1. Web Serial API / WebUSB

| Prós | Contras |
|------|---------|
| Zero dependência de software externo | Suporte limitado (Chrome/Edge only, flag experimental) |
| Comunicação direta browser ↔ hardware | Requer pairing manual do dispositivo pelo operador |
| Controle total sobre bytes ESC/POS | Não funciona com impressoras em rede (apenas USB direto) |
| | Incompatível com Firefox/Safari |

**Veredicto:** Viável como opção secundária para cenários USB-only com Chrome. Não confiável como solução primária em ambiente de produção F&B.

#### 2. Print Dialog Nativo (`window.print()`)

| Prós | Contras |
|------|---------|
| Funciona em qualquer navegador | Não suporta formatação ESC/POS (negrito térmico, corte de papel, gaveta) |
| Sem setup | Requer interação manual do operador a cada impressão |
| | Resultado inconsistente entre impressoras e drivers |

**Veredicto:** Inaceitável para operação de delivery onde o pedido precisa imprimir automaticamente sem intervenção.

#### 3. Micro-Serviço de Spooler Local (Recomendado)

| Prós | Contras |
|------|---------|
| Controle total sobre ESC/POS | Requer instalação de um agente local (leve) |
| Funciona com USB e rede | Mais um componente para manter |
| Impressão automática (sem dialog) | |
| Cross-browser | |
| Suporta múltiplas impressoras | |
| Auto-update possível | |

**Veredicto:** Solução recomendada. O trade-off de instalar um agente local leve (< 20MB, sem dependências pesadas) é aceitável dado que elimina completamente o desktop app dos marketplaces.

### Arquitetura do Print Spooler

```
┌──────────────────────────┐          ┌─────────────────────────────┐
│   Browser (POS Client)   │          │   Print Spooler Agent       │
│                          │          │   (localhost:9100)           │
│  Recebe pedido via SSE   │──HTTP───▶│                             │
│  Monta payload ESC/POS   │  POST    │  - Recebe payload ESC/POS   │
│  Envia para localhost    │          │  - Roteia para impressora   │
│                          │          │  - Corte de papel + gaveta  │
└──────────────────────────┘          │  - Status de confirmação    │
                                      └──────────────┬──────────────┘
                                                     │
                                          ┌──────────▼──────────┐
                                          │  Impressora Térmica  │
                                          │  (USB / Rede)        │
                                          └─────────────────────┘
```

### Especificação do Print Spooler Agent

| Aspecto | Detalhe |
|---------|---------|
| **Runtime** | Electron (sem UI) ou binário Go/Rust compilado. Preferência por Go pela simplicidade de cross-compilation e tamanho mínimo do binário. |
| **Protocolo** | HTTP REST em `localhost:9100`. Endpoint `POST /print` aceita payload com bytes ESC/POS em base64 ou JSON estruturado. |
| **Descoberta** | O browser tenta `fetch('http://localhost:9100/health')`. Se responder 200, o spooler está ativo. Caso contrário, exibe banner: "Instale o Print Agent para impressão automática". |
| **Segurança** | Aceita conexões apenas de `127.0.0.1` e `::1`. Header `Origin` validado contra domínios autorizados (configurável). Token compartilhado gerado na instalação. |
| **Configuração** | Arquivo local `~/.delivery-print-agent/config.json` com lista de impressoras, porta preferida, e mapeamento impressora ↔ tipo de documento. |
| **Atualização** | Auto-update via GitHub Releases ou endpoint próprio. Verifica nova versão a cada 24h. |
| **Instalação** | Installer MSI (Windows), .deb/.AppImage (Linux), .dmg (macOS). Tamanho alvo < 15MB. |

### Fallback: WebUSB para Cenários Simplificados

Para clientes que possuem impressora USB conectada diretamente ao terminal e utilizam Chrome, oferecer a opção de impressão via WebUSB como alternativa ao spooler:

```typescript
// Fluxo simplificado WebUSB
const device = await navigator.usb.requestDevice({
  filters: [{ vendorId: 0x04b8 }] // Epson
});
await device.open();
await device.selectConfiguration(1);
await device.claimInterface(0);
await device.transferOut(1, escPosBuffer);
```

Essa opção fica disponível nas configurações do POS como "Impressão direta via navegador (experimental)".

---

## Unificação de Status e Ciclo de Vida do Pedido

### Estado Unificado

O delivery engine precisa mapear os estados internos de cada plataforma para um ciclo de vida unificado no POS:

```
                    ┌──────────────────────────────────────────────────┐
                    │         CICLO DE VIDA DO PEDIDO DELIVERY          │
                    ├──────────────────────────────────────────────────┤
                    │                                                    │
                    │  RECEIVED ──▶ PENDING_ACCEPTANCE                   │
                    │                    │                               │
                    │           ┌────────┼────────┐                     │
                    │           ▼                  ▼                     │
                    │      ACCEPTED           REJECTED                   │
                    │           │                  │                     │
                    │           ▼                  ▼                     │
                    │      IN_PREPARATION     CANCELLED                  │
                    │           │                                        │
                    │           ▼                                        │
                    │      READY_FOR_PICKUP                              │
                    │           │                                        │
                    │           ▼                                        │
                    │      DISPATCHED                                    │
                    │           │                                        │
                    │           ▼                                        │
                    │      DELIVERED (confirmado pela plataforma)        │
                    │                                                    │
                    └──────────────────────────────────────────────────┘
```

### Mapeamento por Plataforma

| Estado Interno | iFood | 99Food | Rappi |
|----------------|-------|--------|-------|
| PENDING_ACCEPTANCE | `PLC` (Placed) | `PENDING` | `NEW` |
| ACCEPTED | `CFM` (Confirmed) | `ACCEPTED` | `ACCEPTED` |
| IN_PREPARATION | `CFM` | `PREPARING` | `IN_STORE` |
| READY_FOR_PICKUP | `RTP` (Ready to Pickup) | `READY` | `READY_FOR_PICKUP` |
| DISPATCHED | `DSP` (Dispatched) | `ON_THE_WAY` | `IN_ROUTE` |
| DELIVERED | `CON` (Concluded) | `DELIVERED` | `DELIVERED` |
| CANCELLED | `CAN` (Cancelled) | `CANCELLED` | `CANCELLED` |

### Ações do Operador na Interface

| Ação | Efeito Local | Efeito na Plataforma |
|------|--------------|----------------------|
| **Aceitar** | Status → ACCEPTED, trigger impressão | PATCH /orders/{id}/confirm |
| **Rejeitar** | Status → REJECTED, motivo obrigatório | PATCH /orders/{id}/reject + reason |
| **Preparando** | Status → IN_PREPARATION | Atualiza status (se API suportar) |
| **Pronto p/ retirada** | Status → READY_FOR_PICKUP | PATCH /orders/{id}/readyForPickup |
| **Solicitar cancelamento** | Status → CANCELLATION_REQUESTED | POST /orders/{id}/cancellation |

### Timeout de Aceite

As plataformas impõem SLAs de aceitação (geralmente 5 minutos). O sistema deve:

1. Exibir countdown visual no card do pedido
2. Escalar alerta sonoro a cada 60s sem ação
3. Se o timer expirar, a plataforma cancela automaticamente — o sistema deve refletir esse cancelamento via polling/webhook

---

## Modelagem de Dados

> O Delivery Engine possui banco de dados próprio, isolado de qualquer consumidor. O schema abaixo descreve a estrutura interna do microserviço.

### Schema do Delivery Engine (PostgreSQL — banco dedicado)

```prisma
// Schema interno do Delivery Engine (serviço isolado)
// Não faz parte do schema do Laurus Cloud ou de qualquer consumidor

model Tenant {
  id            String   @id @default(cuid())
  name          String   // Nome do estabelecimento ou sistema parceiro
  type          TenantType // DIRECT (restaurante) ou PLATFORM (PDV parceiro)
  webhookUrl    String   // Endpoint para entrega de eventos
  webhookSecret String   // Secret para assinatura HMAC
  apiKey        String   @unique // Chave de acesso à Management API
  apiSecret     String   // Hash do secret
  isActive      Boolean  @default(true)
  config        Json?    // Configurações globais do tenant
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  merchants     Merchant[]
}

model Merchant {
  id            String   @id @default(cuid())
  tenantId      String
  name          String   // Nome do restaurante/loja
  externalRef   String?  // Referência no sistema do consumidor
  tenant        Tenant   @relation(fields: [tenantId], references: [id])
  integrations  PlatformIntegration[]
  orders        DeliveryOrder[]

  @@index([tenantId])
}

model PlatformIntegration {
  id             String   @id @default(cuid())
  merchantId     String
  platform       DeliveryPlatform
  platformMerchantId String // ID do restaurante na plataforma de delivery
  accessToken    String   // Encriptado (AES-256-GCM)
  refreshToken   String?
  tokenExpiresAt DateTime?
  isActive       Boolean  @default(true)
  config         Json?    // Configurações específicas da plataforma
  lastPolledAt   DateTime?
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  merchant       Merchant @relation(fields: [merchantId], references: [id])

  @@unique([merchantId, platform])
}

model DeliveryEvent {
  id            String   @id @default(cuid())
  merchantId    String
  platform      DeliveryPlatform
  externalId    String   // ID do evento na plataforma
  eventType     String   // NEW_ORDER, STATUS_CHANGE, CANCELLATION
  rawPayload    Json     // Payload original (auditoria)
  processedAt   DateTime?
  errorMessage  String?
  createdAt     DateTime @default(now())

  @@unique([platform, externalId])
  @@index([merchantId, createdAt])
}

model DeliveryOrder {
  id                 String   @id @default(cuid())
  merchantId         String
  platform           DeliveryPlatform
  externalOrderId    String   // ID do pedido na plataforma
  status             DeliveryOrderStatus
  displayCode        String   // Código curto (#1234)
  customerName       String
  customerPhone      String?
  deliveryAddress    Json?
  items              Json     // Itens normalizados (snapshot)
  subtotal           Decimal  @db.Decimal(10, 2)
  deliveryFee        Decimal  @db.Decimal(10, 2)
  discount           Decimal  @db.Decimal(10, 2) @default(0)
  total              Decimal  @db.Decimal(10, 2)
  paymentMethod      String   // ONLINE, CASH, CARD_ON_DELIVERY
  paymentPaidOnline  Boolean  @default(false)
  scheduledFor       DateTime?
  acceptDeadline     DateTime
  rejectionReason    String?
  platformMetadata   Json?
  receivedAt         DateTime @default(now())
  acceptedAt         DateTime?
  dispatchedAt       DateTime?
  concludedAt        DateTime?
  cancelledAt        DateTime?

  merchant           Merchant @relation(fields: [merchantId], references: [id])
  webhookDeliveries  WebhookDelivery[]

  @@unique([platform, externalOrderId])
  @@index([merchantId, status])
  @@index([merchantId, receivedAt])
}

model WebhookDelivery {
  id            String   @id @default(cuid())
  orderId       String
  eventType     String
  httpStatus    Int?
  responseBody  String?
  attempts      Int      @default(0)
  nextRetryAt   DateTime?
  deliveredAt   DateTime?
  failedAt      DateTime?
  createdAt     DateTime @default(now())

  order         DeliveryOrder @relation(fields: [orderId], references: [id])

  @@index([orderId])
  @@index([nextRetryAt])
}

enum TenantType {
  DIRECT    // Restaurante usando o Engine diretamente
  PLATFORM  // PDV/ERP parceiro integrando para seus clientes
}

enum DeliveryPlatform {
  IFOOD
  NINETY_NINE_FOOD
  RAPPI
  UBER_EATS
}

enum DeliveryOrderStatus {
  RECEIVED
  PENDING_ACCEPTANCE
  ACCEPTED
  REJECTED
  IN_PREPARATION
  READY_FOR_PICKUP
  DISPATCHED
  DELIVERED
  CANCELLED
  CANCELLATION_REQUESTED
}
```

### Como o Laurus Cloud (Consumidor) se Integra

O Laurus Cloud — como qualquer outro consumidor — não compartilha banco com o Engine. A integração acontece assim:

1. **Onboarding:** Laurus se registra como tenant tipo `PLATFORM` no Engine, recebe `api_key` + `webhook_secret`.
2. **Webhook recebido:** Quando o Engine faz POST no endpoint do Laurus com um `order.created`, o Laurus cria internamente um `POSOrder` com `source: 'DELIVERY'` e vincula os dados recebidos.
3. **Ações:** Quando o operador aceita o pedido no POS do Laurus, o backend do Laurus faz `POST /v1/orders/{id}/accept` na Management API do Engine.
4. **Impressão:** O Laurus utiliza o Print Agent (componente open-source) para imprimir o cupom localmente.

Nenhuma tabela, enum ou relação do Engine existe dentro do schema do Laurus. O acoplamento é zero — apenas HTTP e JSON.

---

## Integrações de Plataforma

### iFood (Prioridade 1)

| Aspecto | Detalhe |
|---------|---------|
| **Autenticação** | OAuth2 client_credentials. Token com TTl de 1h. |
| **Ingestion** | Polling em `/v3.0/events:polling`. Batch de até 50 eventos. ACK obrigatório em `/v1.0/events/acknowledgment`. |
| **Confirmação** | `POST /v1.0/orders/{id}/confirm` com `estimatedPreparationTime` em segundos. |
| **Cancelamento** | `POST /v1.0/orders/{id}/requestCancellation` com `reason` e `cancellationCode`. |
| **Dispatch** | `POST /v1.0/orders/{id}/readyToPickup` quando pedido pronto. |
| **Rate Limit** | 60 req/min por merchant. Polling recomendado a cada 30s. |
| **Sandbox** | Disponível via programa de parceiros iFood. |

### 99Food

| Aspecto | Detalhe |
|---------|---------|
| **Autenticação** | API Key + Secret no header. |
| **Ingestion** | Webhook configurável no painel administrativo. |
| **Formato** | JSON com estrutura proprietária (itens aninhados, modificadores separados). |
| **SLA** | Aceite em até 300 segundos. |

### Rappi

| Aspecto | Detalhe |
|---------|---------|
| **Autenticação** | OAuth2 com refresh token. |
| **Ingestion** | Webhook primário + polling fallback. |
| **Particularidades** | Suporta pedidos multi-store. Campo `store_id` identifica unidade. |

### Padrão Adapter

Cada plataforma é implementada como um adapter que satisfaz uma interface comum:

```typescript
interface DeliveryPlatformAdapter {
  readonly platform: DeliveryPlatform;

  // Autenticação
  authenticate(credentials: PlatformCredentials): Promise<AuthTokens>;
  refreshAuth(integration: DeliveryIntegration): Promise<AuthTokens>;

  // Ingestion
  pollEvents?(integration: DeliveryIntegration): Promise<RawDeliveryEvent[]>;
  acknowledgeEvent?(integration: DeliveryIntegration, eventId: string): Promise<void>;
  parseWebhookPayload(headers: Headers, body: unknown): ParsedWebhookEvent;
  validateWebhookSignature(headers: Headers, body: Buffer, secret: string): boolean;

  // Normalização
  normalizeOrder(rawPayload: unknown): NormalizedDeliveryOrder;

  // Ações
  confirmOrder(integration: DeliveryIntegration, orderId: string, prepTime: number): Promise<void>;
  rejectOrder(integration: DeliveryIntegration, orderId: string, reason: string): Promise<void>;
  markReadyForPickup(integration: DeliveryIntegration, orderId: string): Promise<void>;
  requestCancellation(integration: DeliveryIntegration, orderId: string, reason: string): Promise<void>;
}
```

Novos marketplaces são integrados implementando esse contrato sem alterar o core do Delivery Engine.

---

## Considerações de Segurança

| Vetor | Mitigação |
|-------|-----------|
| **Webhook spoofing (inbound)** | Validação obrigatória de HMAC/assinatura em cada payload recebido dos marketplaces. Rejeição imediata se inválido. |
| **Webhook spoofing (outbound)** | Toda entrega ao consumidor assinada com HMAC-SHA256 via secret compartilhado. Consumidor deve validar `X-Delivery-Engine-Signature`. |
| **Token storage** | Tokens OAuth2 dos marketplaces encriptados em banco (AES-256-GCM). Nunca em plain text. Decriptação apenas no momento de uso. |
| **API Key management** | API keys dos consumidores armazenadas como hash (bcrypt/argon2). Prefixo público (`dek_live_`) para identificação sem exposição. |
| **Rate limiting** | Endpoints públicos com rate limit por API key e por IP. Proteção contra abuso e DDoS. |
| **Tenant isolation** | Toda query filtrada por `merchantId` derivado da API key autenticada. Impossível acessar dados de outro tenant. |
| **Print Agent** | Bind exclusivo em localhost. Token de autenticação local. Validação de Origin header contra whitelist configurável. |
| **Dados sensíveis (LGPD)** | Dados do cliente final (endereço, telefone) retidos apenas pelo tempo necessário. Política de retenção configurável por tenant. Endpoint de data deletion disponível. |
| **Idempotência** | Todas as operações de webhook são idempotentes. `externalId` como chave de deduplicação. Reprocessamento seguro. |

---

## Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| iFood deprecar endpoint de polling | Média | Alto | Monitorar changelog da API. Adapter pattern permite swap rápido. Versionar adapters. |
| Latência de entrega ao consumidor | Média | Alto | SLA interno de < 2s entre recebimento e dispatch. Monitoramento ativo de p95/p99. Fallback SSE para latência sub-segundo. |
| Consumidor offline (webhook falha) | Alta | Médio | Retry com exponential backoff (7 tentativas). Dead letter queue. Dashboard de falhas. Alerta ao consumidor. |
| Operador não aceitar pedido a tempo | Alta | Alto | Responsabilidade do consumidor, mas o Engine emite evento `order.accept_deadline_approaching` 60s antes do timeout para facilitar escalation. |
| Impressora offline no momento do pedido | Média | Médio | Queue local no Print Agent. Retry automático. Status de impressão reportado ao consumidor. |
| Divergência de cardápio (item sem correspondência) | Alta | Médio | Engine entrega itens com nome e preço originais da plataforma. Mapeamento de SKU é responsabilidade do consumidor. |
| Concorrência (outros hubs de delivery) | Média | Médio | Diferenciação por qualidade de API, latência, preço e developer experience. API-first com docs excepcionais. |
| Dependência de parceria com marketplaces | Alta | Alto | Diversificar plataformas desde o início. Não depender de um único marketplace para viabilidade. |

---

## Infraestrutura e Deploy

### Stack do Microserviço

| Camada | Tecnologia | Justificativa |
|--------|------------|---------------|
| **Runtime** | Node.js (NestJS ou Fastify) | Ecossistema TypeScript do grupo. NestJS para estrutura enterprise-grade com DI, modules e guards. |
| **Banco de Dados** | PostgreSQL (dedicado) | Consistência com stack do grupo. Isolamento total de dados. |
| **Queue / Jobs** | BullMQ + Redis | Polling workers, webhook dispatch com retry, dead letter queue. |
| **Cache** | Redis | Rate limiting, deduplicação, token cache, pub/sub para SSE. |
| **Hosting** | AWS (ECS Fargate) ou Railway/Render | Container-based. Sem acoplamento à Vercel do Laurus. Autoscaling por carga de webhooks. |
| **Observabilidade** | Sentry + Grafana/Datadog | Error tracking, métricas de latência de webhook delivery, health de polling workers. |

### Justificativa de Separação da Vercel

O Laurus Cloud roda na Vercel (otimizada para Next.js, edge functions, SSR). O Delivery Engine tem necessidades incompatíveis:

- **Long-running processes** — Polling workers rodam continuamente (30s loops). Vercel não suporta processos persistentes.
- **WebSocket/SSE persistente** — Vercel impõe limites de 30s–300s em streaming. O Engine precisa de conexões abertas por minutos/horas.
- **Background jobs** — BullMQ requer worker processes dedicados.
- **Custo de escala** — Tráfego de webhooks inbound (marketplaces) pode gerar milhares de invocations/minuto. Em serverless, o custo escala linearmente. Em container, é previsível.

### Topology de Deploy

```
┌─────────────────────────────────────────────────────────┐
│              DELIVERY ENGINE (AWS / Railway)              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────┐  ┌────────────────────────────┐   │
│  │  API Server      │  │  Worker Pool               │   │
│  │  (NestJS/Fastify)│  │  (Polling + Dispatch)      │   │
│  │  - Management API│  │  - iFood poller            │   │
│  │  - Webhook recv  │  │  - Webhook dispatcher      │   │
│  │  - SSE gateway   │  │  - Dead letter processor   │   │
│  └────────┬─────────┘  └─────────────┬──────────────┘   │
│           │                           │                  │
│           ▼                           ▼                  │
│  ┌──────────────────────────────────────────────────┐   │
│  │  PostgreSQL (RDS)          Redis (ElastiCache)   │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Pré-requisitos para Implementação

| Dependência | Status | Bloqueante? |
|-------------|--------|-------------|
| Cadastro como parceiro/integrador iFood | Pendente | Sim |
| Credenciais API 99Food | Pendente | Sim |
| Definição de infraestrutura (AWS / Railway) | Pendente | Sim |
| Setup de PostgreSQL + Redis dedicados | Pendente | Sim |
| Definição de domínio/branding do produto | Pendente | Não (pode usar subdomain provisório) |
| Print Agent (componente open-source) | Não iniciado | Não (consumidor pode operar sem ele) |

---

## Referências

- [iFood API - Documentação Oficial](https://developer.ifood.com.br/)
- [Web Serial API - W3C Specification](https://wicg.github.io/serial/)
- [WebUSB API - W3C Specification](https://wicg.github.io/webusb/)
- [Server-Sent Events - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [ESC/POS Command Reference - Epson](https://reference.epson-biz.com/modules/ref_escpos/)
- [Next.js Streaming - Vercel Docs](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)

---

## Apêndice: Decisões em Aberto

| Decisão | Opções | Critério de Escolha |
|---------|--------|---------------------|
| Runtime do Print Agent | Go / Rust / Electron (headless) | Tamanho do binário, facilidade de cross-compile, manutenibilidade pelo time (TS-first) |
| Framework do microserviço | NestJS / Fastify standalone / Hono | Estrutura para DI e modules vs. performance raw vs. edge-readiness |
| Serviço de realtime em escala | Ably / Pusher / Soketi / Socket.IO próprio | Custo, latência, vendor lock-in, self-hosting |
| Polling interval (iFood) | 15s / 30s / 60s | Trade-off entre latência de recebimento e consumo de rate limit |
| Mapeamento de cardápio | Manual (config) / Automático (fuzzy match por nome) | Acurácia vs. esforço de configuração para o operador |
| Persistência de eventos raw | Mesma database / Object Storage (S3) | Volume esperado, custo de storage, necessidade de query |
| Hosting | AWS ECS Fargate / Railway / Render / Fly.io | Custo, complexidade de ops, autoscaling, proximidade com região BR |
| Modelo de pricing | Per-order fee / Flat subscription / Hybrid | Elasticidade, previsibilidade para o cliente, margem |

---

## Brainstorming de Produto

### Candidatos a Nome

| Nome | Conceito | Análise |
|------|----------|---------|
| **Rotaflow** | Rota + fluxo. Evoca o caminho do pedido desde a plataforma até o PDV. Sonoridade fluida, fácil de pronunciar em português. | Forte. Transmite movimento e direção. |
| **Ordrhub** | Order + hub. Centro de convergência de pedidos. Grafia compacta, estética tech. | Funcional. Comunica exatamente o que faz. Pode ser genérico demais. |
| **Pulseo** | Pulso. O motor que bate em tempo real, vivo, responsivo. Sufixo -eo dá modernidade. | Diferenciado. Transmite urgência e velocidade. Memorável. |
| **Delvox** | Delivery + vox (voz). A voz unificada de todos os canais de delivery. | Curto, forte, sonoro. Registro de domínio possivelmente disponível. |
| **Nexord** | Next + order. O próximo pedido está sempre chegando. | Compacto. Pode soar artificial. |
| **Tramit** | Trâmite / transit. O fluxo processual do pedido. | Enraizado no português. Pode parecer burocrático. |
| **Fluxer** | Fluxo + sufixo agente (-er). Quem faz o fluxo acontecer. | Internacional, limpo. Fácil de pronunciar em qualquer idioma. |
| **Vendabit** | Venda + orbit. Os pedidos orbitam um centro gravitacional único. | Criativo. Conecta venda com convergência. |

### Identidade e Posicionamento

**Posicionamento proposto:**

> "A infraestrutura invisível que conecta marketplaces de delivery a qualquer sistema de gestão — sem apps desktop, sem fragmentação, sem dados perdidos."

**Público-alvo primário:**

- Empresas de software de PDV/ERP para food service que querem oferecer integração de delivery aos seus clientes sem construir e manter os adapters de cada marketplace.

**Público-alvo secundário:**

- Restaurantes e redes que operam PDVs próprios (sistemas internos) e precisam de um hub de delivery sem depender dos apps desktop.

**Identidade visual (diretrizes):**

| Aspecto | Direção |
|---------|---------|
| Tom | Técnico, confiável, infraestrutura. Não é consumer-facing. |
| Cores | Escuro (dark mode first). Accent em verde ou azul elétrico — remete a status ativo, real-time. |
| Tipografia | Monospace em código/docs. Sans-serif geométrica (Inter, Geist) na marca. |
| Voz de marca | "Infraestrutura para quem constrói." Fala com CTOs e devs, não com garçons. |
| Analogia | Stripe para pagamentos → [Nome] para delivery. A camada de abstração que ninguém quer construir. |

**Tagline candidates:**

- "Delivery unificado. Uma API."
- "Todos os pedidos. Um endpoint."
- "A API que elimina o app desktop."
- "Delivery integration as infrastructure."

---

<p align="center">
  <sub>
    &copy; 2024–2026 Agis Group. Todos os direitos reservados.<br/>
    <strong>Delivery Integration Engine</strong> — Produto Independente — RFC Arquitetural
  </sub>
</p>
