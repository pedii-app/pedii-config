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

```bash
Authorization: Bearer <AGENT_API_KEY>
```

---

## Fluxo típico do agente

```
1. Mensagem recebe do cliente via WhatsApp
        ↓
2. GET /customers?store_id=X&whatsapp=55319...
   → found: false?
        ↓
3. POST /customers  ← cria o cliente
        ↓
4. GET /products?store_id=X&in_stock=true  ← catálogo disponível
        ↓
5. POST /calcular-frete  ← verifica raio + valor
        ↓
6. [BACKLOG] POST /orders  ← finaliza pedido
```

---

## Endpoints

### `GET /customers` — Buscar cliente

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `store_id` | uuid | ✅ | ID da loja do agente |
| `whatsapp` | string | ⚠️ | Número com DDI (ex: `5531999990000`) |
| `id` | uuid | ⚠️ | UUID do cliente |

> ⚠️ Informe `whatsapp` **ou** `id`.

**Resposta 200:**
```json
{
  "found": true,
  "customer": {
    "id": "ee4d2a74-...",
    "nome": "João Silva",
    "whatsapp": "5531999990000",
    "cidade": "Varginha",
    "estado": "MG",
    "lat": -21.5513,
    "lng": -45.4333
  }
}
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

> O geocoding de `lat`/`lng` acontece automaticamente via trigger após a inserção.

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
  "total": 2,
  "products": [
    {
      "id": "uuid",
      "descricao": "Café Especial 250g",
      "unidade_medida": "UN",
      "quantidade": 48,
      "valor_venda": 32.90,
      "group": { "id": "uuid", "nome": "Cafés" }
    }
  ],
  "groups": [
    { "id": "uuid", "nome": "Cafés" }
  ]
}
```

---

### `POST /calcular-frete` — Calcular frete e raio de entrega

**Body:**
```json
{
  "loja_id": "uuid-da-loja",
  "cliente_id": "uuid-do-cliente"
}
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

**Lógica de cálculo:**
- `distancia_km × valor_por_km` = valor base
- Se valor base < `valor_minimo` → aplica o mínimo
- Se valor do pedido ≥ `frete_gratis_acima_de` → frete = R$ 0,00

> Se `dentro_raio: false`, o agente deve informar que a loja não faz entrega no endereço do cliente.

---

### `POST /orders` — Criar pedido *(backlog)*

Em desenvolvimento. Registrará o pedido no dashboard do lojista ao final da venda.

---

## Códigos de erro

| Status | Significado |
|---|---|
| `400` | Parâmetros obrigatórios ausentes ou inválidos |
| `401` | Token ausente ou inválido |
| `403` | Loja e cliente de organizações diferentes |
| `404` | Recurso não encontrado |
| `422` | Endereço insuficiente para geocodificar |
| `500` | Erro interno |

---

## Referência OpenAPI

O arquivo `openapi.yaml` neste diretório segue o padrão OpenAPI 3.1 e pode ser importado
diretamente em ferramentas como Postman, Insomnia ou frameworks de agentes compatíveis
com OpenAPI (ex: Google Vertex AI Agent Builder, LangChain OpenAPI Toolkit).
