# Pedii Agent API

APIs do Pedii consumidas pelo agente de vendas WhatsApp rodando no GCP.

## Base URL

```
https://qiqwylcjoyztqebqglok.supabase.co/functions/v1
```

## Autenticação

Todas as rotas usam `Authorization: Bearer <AGENT_API_KEY>`.

A chave é armazenada como secret no Supabase (`AGENT_API_KEY`) e no **GCP Secret Manager**.
Nunca commitar a chave no repositório.

---

## Fluxo típico do agente

```
1. Mensagem recebida do cliente via WhatsApp
        ↓
2. GET /customers?store_id=X&whatsapp=55319...
   → found: false?
        ↓
3. POST /customers  ← cria o cliente
        ↓
4. GET /products?store_id=X&in_stock=true  ← catálogo disponível
        ↓
5. POST /calculate-freight  ← verifica raio + valor do frete
        ↓
6. POST /orders  ← registra o pedido no dashboard do lojista
        ↓
7. [Webhook recebido] order.status_changed  ← lojista atualizou status
        ↓
8. Agente notifica o cliente via WhatsApp
```

---

## Endpoints de saída (agente → Pedii)

### `GET /customers` — Buscar cliente

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `store_id` | uuid | ✅ | ID da loja do agente |
| `whatsapp` | string | ⚠️ | Número com DDI (ex: `5531999990000`) |
| `id` | uuid | ⚠️ | UUID do cliente |

> ⚠️ Informe `whatsapp` **ou** `id`.

**Resposta 200:**
```json
{ "found": true, "customer": { "id": "...", "nome": "João Silva", "whatsapp": "5531999990000" } }
```
Se não encontrado: `{ "found": false, "customer": null }`

---

### `POST /customers` — Criar ou atualizar cliente

Upsert por `whatsapp` + organização. Use para novos clientes ou atualização de endereço.

**Body:**
```json
{
  "store_id": "uuid-da-loja",
  "nome": "João Silva",
  "whatsapp": "5531999990000",
  "logradouro": "Avenida do Contorno",
  "numero": "40",
  "bairro": "Santa Luiza",
  "cidade": "Varginha",
  "estado": "MG",
  "cep": "37062683"
}
```

**Resposta 201:** `{ "customer": { ...dados } }`

---

### `GET /products` — Consultar estoque

| Parâmetro | Tipo | Padrão | Descrição |
|---|---|---|---|
| `store_id` | uuid | — | ✅ Obrigatório |
| `search` | string | — | Busca textual na descrição |
| `group_id` | uuid | — | Filtrar por categoria |
| `in_stock` | boolean | `false` | Somente com quantidade > 0 |

**Resposta 200:**
```json
{
  "store": { "id": "uuid", "nome": "Loja Centro" },
  "total": 1,
  "products": [{ "id": "uuid", "descricao": "Café Especial 250g", "quantidade": 48, "valor_venda": 32.90 }],
  "groups": [{ "id": "uuid", "nome": "Cafés" }]
}
```

---

### `POST /calculate-freight` — Calcular frete e raio de entrega

**Body:**
```json
{ "loja_id": "uuid-da-loja", "cliente_id": "uuid-do-cliente" }
```

**Resposta 200:**
```json
{
  "dentro_raio": true,
  "distancia_km": 3.24,
  "raio_entrega_km": 5.0,
  "valor_frete": 9.72,
  "frete_gratis": false,
  "pedido_minimo_entrega": 30.00,
  "frete_gratis_acima_de": 100.00
}
```

> Se `dentro_raio: false`, informar ao cliente que a loja não entrega no endereço dele.

---

### `POST /orders` — Criar pedido

**Body:**
```json
{
  "store_id": "uuid-da-loja",
  "customer_id": "uuid-do-cliente",
  "notes": "Entregar no período da tarde",
  "shipping_value": 9.72,
  "shipping_distance_km": 3.24,
  "shipping_within_radius": true,
  "items": [
    { "product_id": "uuid", "quantity": 2, "unit_price": 32.90 }
  ]
}
```

**Resposta 201:** `{ "order": { "id": "uuid", "status": "open", "total": 75.52, ... } }`

---

### `GET /orders` — Listar ou buscar pedido

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `store_id` | uuid | ✅ Obrigatório para listagem |
| `id` | uuid | Buscar pedido único (retorna itens completos) |
| `status` | string | Filtrar por status |
| `customer_id` | uuid | Filtrar por cliente |
| `date_from` | date | Data inicial (`2026-03-01`) |
| `date_to` | date | Data final (`2026-03-31`) |
| `page` | int | Paginação (padrão: 1) |
| `limit` | int | Itens por página (padrão: 50, máx: 100) |

**Resposta 200 (listagem):**
```json
{ "orders": [...], "total": 10, "page": 1, "limit": 50 }
```

---

### `PATCH /orders` — Atualizar status ou adicionar itens

**Atualizar status:**
```json
{ "id": "uuid-do-pedido", "action": "update_status", "status": "accepted" }
```

**Adicionar itens:**
```json
{
  "id": "uuid-do-pedido",
  "action": "add_items",
  "items": [{ "product_id": "uuid", "quantity": 1, "unit_price": 20.00 }]
}
```

**Máquina de estados:**
```
open → accepted | cancelled
accepted → shipped | cancelled
shipped → received | cancelled
received → (terminal)
cancelled → (terminal)
```

**Resposta 422** se a transição não for válida:
```json
{ "error": "Transição inválida: received → cancelled", "allowed": [] }
```

---

## Webhooks recebidos pelo agente (Pedii → agente)

O Pedii envia eventos HTTP POST ao agente quando o **lojista** realiza ações no dashboard.
O agente deve expor um endpoint para receber esses eventos e disparar mensagens no WhatsApp.

### Autenticação recebida

```
Authorization: Bearer <AGENT_API_KEY>
x-internal-secret: <INTERNAL_WEBHOOK_SECRET>
```

### Evento: `order.status_changed`

Disparado quando o lojista muda o status de um pedido pelo dashboard.
> Mudanças feitas pelo próprio agente via `PATCH /orders` **não** disparam este evento.

**Payload:**
```json
{
  "event": "order.status_changed",
  "order_id": "83781578-c102-4bd3-8f22-9ea6167887aa",
  "old_status": "open",
  "new_status": "accepted",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "order": {
    "id": "83781578-c102-4bd3-8f22-9ea6167887aa",
    "status": "accepted",
    "total": 310.00,
    "shipping_value": 10.00,
    "stores": { "apelido": "Paraguaçu", "razao_social": "TM Farma - Paraguaçu" },
    "customers": { "nome": "Eloá Mariane da Luz", "whatsapp": "(35) 98272-1736" },
    "order_items": [
      { "quantity": 15, "unit_price": 20.00, "subtotal": 300.00, "products": { "descricao": "Produto Teste Paraguaçu" } }
    ]
  }
}
```

**Mensagens sugeridas por transição:**

| Transição | Mensagem ao cliente |
|---|---|
| `open → accepted` | "✅ Seu pedido foi aceito e está sendo preparado!" |
| `accepted → shipped` | "🚚 Seu pedido saiu para entrega!" |
| `shipped → received` | "📦 Entrega confirmada. Obrigado pela compra!" |
| `* → cancelled` | "❌ Seu pedido foi cancelado. Entre em contato para mais informações." |

---

## Códigos de erro

| Status | Significado |
|---|---|
| `400` | Parâmetros obrigatórios ausentes ou inválidos |
| `401` | Token ausente ou inválido |
| `403` | Loja e cliente de organizações diferentes |
| `404` | Recurso não encontrado |
| `422` | Transição inválida ou endereço insuficiente para geocodificar |
| `500` | Erro interno |

---

## Referência OpenAPI

O arquivo `openapi.yaml` neste diretório segue o padrão OpenAPI 3.1 e pode ser importado
diretamente em ferramentas como Postman, Insomnia ou frameworks de agentes compatíveis
com OpenAPI (ex: Google Vertex AI Agent Builder, LangChain OpenAPI Toolkit).
