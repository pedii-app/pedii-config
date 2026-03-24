# Pedii Agent API

APIs do Pedii consumidas pelo agente de vendas WhatsApp rodando no GCP.

## Base URL

```
https://qiqwylcjoyztqebqglok.supabase.co/functions/v1
```

## Autenticação

### Agente → Pedii (chamadas de saída)

Todas as rotas usam:
```
Authorization: Bearer <AGENT_API_KEY>
```

### Pedii → Agente (webhooks recebidos)

Os webhooks enviados pelo Pedii ao endpoint do agente usam:
```
x-api-key: <AGENT_API_KEY>
x-internal-secret: <INTERNAL_WEBHOOK_SECRET>
```

> A `AGENT_API_KEY` é a mesma chave nos dois sentidos — a diferença é **apenas o header**.
> A chave é armazenada como secret no Supabase (`AGENT_API_KEY`) e no **GCP Secret Manager**.
> Nunca commitar a chave no repositório.

---

## Campos presentes em todas as respostas

Toda resposta bem-sucedida inclui dois campos de contexto da organização:

| Campo | Tipo | Descrição |
|---|---|---|
| `organization_id` | uuid | ID da organização (tenant raiz do Pedii) |
| `instance_name` | string \| null | Nome da instância WhatsApp conectada na Evolution API. `null` se não configurada. |

O agente deve usar o `instance_name` para enviar mensagens via Evolution API.

---

## Fluxo típico do agente

```
1. Mensagem recebida do cliente via WhatsApp
        ↓
1a. GET /resolve-org?instance_name=<nome_da_instancia>  ← PRIMEIRA CHAMADA
    → Resolve organization_id a partir do nome da instância Evolution
    → O agente usa este organization_id em todas as chamadas subsequentes
        ↓
1b. GET /nearest-stores?cep=<CEP_do_cliente>&organization_id=X  ← SEGUNDA CHAMADA
    → Geocodifica o CEP e retorna as 3 lojas mais próximas
    → O agente usa o store_id da loja escolhida nas chamadas seguintes
        ↓
2. GET /customers?organization_id=X&whatsapp=55319...
   → found: false?
        ↓
3. POST /customers  ← cria o cliente
        ↓
4. GET /products?organization_id=X&in_stock=true  ← catálogo de toda a org
   (ou GET /products?store_id=X&in_stock=true  ← catálogo de uma loja específica)
        ↓
5. POST /calculate-freight  ← verifica raio + valor do frete
        ↓
6. POST /orders  ← registra o pedido no dashboard do lojista
        ↓
7a. [Webhook recebido] order.status_changed  ← lojista atualizou status
    → Agente notifica cliente via WhatsApp (instance_name + customer.whatsapp)

7b. [Webhook recebido] order.items_updated   ← atendente editou itens do pedido
    → Agente comunica mudanças ao cliente em linguagem natural
    → Cliente confirma ou recusa na conversa
        ↓
8.  Cliente confirma ajuste:
    PATCH /orders { action: "customer_approved" }
    → awaiting_customer_approval volta a false
    → Atendente fica livre para aceitar o pedido no dashboard
```

---

## Endpoints de saída (agente → Pedii)

### `GET /resolve-org` — Resolver organização pelo nome da instância

**Primeira chamada de toda conversa.** O agente só sabe o `instance_name` da Evolution API ao receber uma mensagem — esta rota retorna o `organization_id` correspondente, que deve ser passado em todas as chamadas subsequentes.

> Rota **exclusiva do agente** — não aceita JWT de usuário, apenas `AGENT_API_KEY`.

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `instance_name` | string | ✅ | Nome exato da instância no Evolution API |

**Resposta 200 (encontrada):**
```json
{
  "found": true,
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "instance_name": "pedii_f988b66e",
  "phone_number": "5531999990000",
  "instance_status": "connected"
}
```

**Resposta 404 (não encontrada):**
```json
{
  "found": false,
  "organization_id": null,
  "instance_name": "nome_que_foi_buscado"
}
```

> Se `found: false`, a instância não está cadastrada no Pedii. O agente não deve prosseguir.

---

### `GET /nearest-stores` — Lojas mais próximas do cliente

**Segunda chamada da jornada.** O agente recebe o CEP do cliente e descobre qual(is) loja(s) da organização podem atendê-lo, já calculando a distância e se o cliente está dentro do raio de entrega.

> Rota **exclusiva do agente** — somente `AGENT_API_KEY`. Aceita `GET` (query string) ou `POST` (body JSON).

**Fluxo interno de geocoding (3 etapas):**
1. **ViaCEP** (`viacep.com.br/ws/{cep}/json/`) → resolve o CEP para logradouro, bairro, cidade e estado
2. **Nominatim** tenta geocodificar com fallbacks progressivos:
   - `"Logradouro, Bairro, Cidade, UF, Brasil"` ← mais preciso
   - `"Bairro, Cidade, UF, Brasil"`
   - `"Cidade, UF, Brasil"` ← mínimo viável
3. Primeira tentativa bem-sucedida é usada para o cálculo Haversine

> Para distância entre lojas e cliente, coordenadas no nível da cidade são suficientes.

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `cep` | string | ✅ | CEP do cliente (8 dígitos, com ou sem hífen) |
| `organization_id` | uuid | ✅ | ID da organização (obtido via `/resolve-org`) |

**Resposta 200:**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "cep": "37062683",
  "cliente_coords": { "lat": -21.5512, "lng": -45.4333 },
  "total_lojas": 3,
  "lojas_sem_coordenadas": 0,
  "nearest_stores": [
    {
      "id": "4d22e5f2-8138-4a67-9446-88d46ed4ee0c",
      "nome": "Varginha",
      "razao_social": "TM Farma - Varginha",
      "apelido": "Varginha",
      "endereco": {
        "logradouro": "Rua Wenceslau Braz",
        "numero": "100",
        "bairro": "Centro",
        "cidade": "Varginha",
        "estado": "MG",
        "cep": "37002000"
      },
      "lat": -21.5510,
      "lng": -45.4300,
      "raio_entrega_km": 5.0,
      "distancia_km": 0.42,
      "dentro_raio": true
    },
    {
      "id": "ee4d2a74-25fd-4432-91a8-96121c6bf64e",
      "nome": "Paraguaçu",
      "razao_social": "TM Farma - Paraguaçu",
      "apelido": "Paraguaçu",
      "endereco": { "cidade": "Paraguaçu", "estado": "MG", "cep": "37280000" },
      "lat": -21.5400,
      "lng": -45.7200,
      "raio_entrega_km": 10.0,
      "distancia_km": 28.15,
      "dentro_raio": false
    }
  ]
}
```

| Campo | Descrição |
|---|---|
| `nearest_stores` | Até 3 lojas **dentro do raio de atendimento**, ordenadas por `distancia_km` crescente |
| `dentro_raio` | `true` se `distancia_km ≤ raio_entrega_km`. `null` se a loja não tem raio configurado (incluída sem filtro) |
| `lojas_sem_coordenadas` | Lojas que não puderam ser geocodificadas e foram excluídas do ranking |
| `message` | Presente apenas quando `nearest_stores` está vazio — explica que o cliente está fora do raio de todas as lojas |

> **Regra de filtragem:** somente lojas onde o cliente está **dentro** do raio de atendimento são retornadas. Lojas sem `raio_entrega_km` configurado (`null`) são incluídas sem restrição de distância. Se nenhuma loja atender o CEP, `nearest_stores` retorna vazio com um campo `message` — **o agente deve informar ao cliente que não há cobertura de entrega no endereço dele**.

**Erros:**
- `400` — `cep` ou `organization_id` ausentes; CEP com menos de 8 dígitos
- `404` — nenhuma loja cadastrada para a org
- `422 (CEP não encontrado)` — ViaCEP retornou erro: CEP inexistente ou desativado nos Correios
- `422 (geocoding falhou)` — CEP existe no ViaCEP mas nenhuma das tentativas retornou coordenadas no Nominatim; o body inclui `endereco_resolvido` para debug

> O geocoding das lojas é lazy: se uma loja ainda não tem `lat/lng`, é geocodificada nesta chamada e as coordenadas são persistidas para uso futuro.

---

### `GET /customers` — Buscar cliente

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `organization_id` | uuid | ✅ | ID da organização (obtido via `/resolve-org`) |
| `store_id` | uuid | — | Alternativa ao `organization_id` (retrocompatibilidade) |
| `whatsapp` | string | ⚠️ | Número com DDI (ex: `5531999990000`) |
| `id` | uuid | ⚠️ | UUID do cliente |

> ⚠️ Informe `whatsapp` **ou** `id`.
> O cliente pertence à **organização**, não à loja — use sempre `organization_id` diretamente. `store_id` é aceito como fallback: quando informado, a função resolve o `organization_id` internamente.

**Resposta 200:**
```json
{
  "found": true,
  "customer": {
    "id": "uuid",
    "nome": "João Silva",
    "whatsapp": "5531999990000",
    "cep": "37062683",
    "logradouro": "Av. do Contorno",
    "numero": "40",
    "complemento": null,
    "bairro": "Santa Luiza",
    "cidade": "Varginha",
    "estado": "MG",
    "lat": -21.5512,
    "lng": -45.4333,
    "created_at": "2026-03-20T12:00:00Z"
  }
}
```
Se não encontrado: `{ "found": false, "customer": null }`

---

### `POST /customers` — Criar ou atualizar cliente

Upsert por `whatsapp` + `organization_id`. Use para novos clientes ou atualização de endereço.

**Body:**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
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

> `store_id` ainda é aceito no lugar de `organization_id` (retrocompatibilidade), mas `organization_id` é o campo canônico. Ao menos um dos dois deve ser informado.

**Resposta 201:** `{ "customer": { ...dados } }`

> O geocoding de lat/lng acontece automaticamente via trigger.

---

### `GET /products` — Consultar estoque

Busca produtos da organização. Pode filtrar por loja específica ou retornar o catálogo inteiro da org.

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `organization_id` | uuid | ⚠️ | ID da org — obrigatório para o agente quando `store_id` não for informado |
| `store_id` | uuid | ⚠️ | Filtrar por loja específica (opcional) |
| `search` | string | — | Busca textual na descrição (case-insensitive) |
| `group_id` | uuid | — | Filtrar por categoria |
| `in_stock` | boolean | — | Se `true`, retorna apenas quantidade > 0 (padrão: `false`) |

> ⚠️ Para o agente: informe `organization_id` **ou** `store_id`. Se ambos forem omitidos, retorna erro 400.

**Resposta 200 (toda a org):**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "total": 3,
  "products": [
    {
      "id": "uuid",
      "store_id": "uuid-da-loja",
      "store_nome": "Varginha",
      "legacy_id": null,
      "descricao": "Café Especial 250g",
      "unidade_medida": "CX",
      "quantidade": 48,
      "valor_venda": 32.90,
      "updated_at": "2026-03-20T10:00:00Z",
      "group": { "id": "uuid", "nome": "Cafés" }
    }
  ],
  "groups": [
    { "id": "uuid", "nome": "Cafés", "store_id": "uuid-da-loja" }
  ]
}
```

**Resposta 200 (com `store_id`):** inclui adicionalmente o campo `store: { id, nome }`.

> Cada produto retorna `store_id` e `store_nome` para identificar a qual loja pertence.

---

### `POST /calculate-freight` — Calcular frete e raio de entrega

**Body:**
```json
{ "loja_id": "uuid-da-loja", "cliente_id": "uuid-do-cliente" }
```

**Resposta 200:**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "instance_name": "pedii_f988b66e",
  "customer": {
    "id": "uuid",
    "nome": "João Silva",
    "whatsapp": "5531999990000"
  },
  "dentro_raio": true,
  "distancia_km": 3.24,
  "raio_entrega_km": 5.0,
  "valor_frete": 9.72,
  "frete_gratis": false,
  "pedido_minimo_entrega": 30.00,
  "frete_gratis_acima_de": 100.00,
  "_geocodificado": { "loja": false, "cliente": false }
}
```

> Se `dentro_raio: false`, informar ao cliente que a loja não entrega no endereço dele.
> O campo `customer.whatsapp` pode ser usado para confirmar o número do cliente no conversacional.

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

**Resposta 201:**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "instance_name": "pedii_f988b66e",
  "order": {
    "id": "uuid",
    "status": "open",
    "total": 75.52,
    "customers": { "id": "uuid", "nome": "João Silva", "whatsapp": "5531999990000" },
    "stores": { "id": "uuid", "apelido": "Varginha" },
    "order_items": [ ... ]
  }
}
```

**Resposta 422 — estoque insuficiente:**
```json
{
  "error": "Estoque insuficiente para um ou mais produtos",
  "itens_com_erro": [
    {
      "product_id": "uuid",
      "descricao": "Café Especial 250g",
      "disponivel": 3,
      "solicitado": 10
    }
  ]
}
```

> A validação de estoque é aplicada tanto na API quanto via trigger no banco. Não é possível criar pedidos com quantidade superior ao estoque disponível.

---

### `GET /orders` — Listar ou buscar pedido

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `store_id` | uuid | ✅ Obrigatório para listagem |
| `id` | uuid | Buscar pedido único (retorna itens completos) |
| `status` | string | Filtrar por status (`open`, `accepted`, `shipped`, `received`, `cancelled`) |
| `customer_id` | uuid | Filtrar por cliente |
| `date_from` | date | Data inicial (`2026-03-01`) |
| `date_to` | date | Data final (`2026-03-31`) |
| `page` | int | Paginação (padrão: 1) |
| `limit` | int | Itens por página (padrão: 50, máx: 100) |

**Resposta 200 (listagem):**
```json
{
  "organization_id": "uuid",
  "instance_name": "pedii_f988b66e",
  "orders": [ ... ],
  "total": 10,
  "page": 1,
  "limit": 50
}
```

**Resposta 200 (pedido único com `id`):**
```json
{
  "organization_id": "uuid",
  "instance_name": "pedii_f988b66e",
  "order": {
    "id": "uuid",
    "status": "accepted",
    "total": 310.00,
    "customers": { "id": "uuid", "nome": "Eloá Mariane", "whatsapp": "35982721736" },
    "stores": { ... },
    "order_items": [ ... ]
  }
}
```

---

### `PATCH /orders` — Atualizar status ou editar itens

Quatro ações disponíveis via `action`:

#### `update_status` — Avançar status do pedido

```json
{ "id": "uuid-do-pedido", "action": "update_status", "status": "accepted" }
```

**Máquina de estados:**
```
open      → accepted | cancelled
accepted  → shipped  | cancelled
shipped   → received | cancelled
received  → (terminal)
cancelled → (terminal)
```

**Resposta 422** se a transição não for válida:
```json
{ "error": "Transição inválida: received → cancelled", "allowed": [] }
```

**Resposta 422** se o pedido aguarda aprovação do cliente:
```json
{
  "error": "Este pedido foi modificado e aguarda aprovação do cliente final antes de ser aceito.",
  "code": "AWAITING_CUSTOMER_APPROVAL"
}
```

> ⚠️ A transição `open → accepted` fica bloqueada enquanto `awaiting_customer_approval = true`. O atendente só consegue aceitar o pedido após o agente chamar `customer_approved`.

---

#### `add_items` — Adicionar itens ao pedido

Disponível para pedidos com status `open` ou `accepted`.

```json
{
  "id": "uuid-do-pedido",
  "action": "add_items",
  "items": [
    { "product_id": "uuid", "quantity": 1, "unit_price": 20.00 }
  ]
}
```

---

#### `edit_items` — Editar itens do pedido (antes de aceitar)

Permite ao atendente incluir, remover ou ajustar quantidades de itens em pedidos `open` ou `accepted` **numa única chamada**. Disparado geralmente antes de aceitar um pedido onde algum item está sem estoque.

Após a edição:
1. O Pedii seta `awaiting_customer_approval = true` no pedido — bloqueando o aceite até o cliente confirmar.
2. Dispara o webhook `order.items_updated` ao agente para que ele comunique as mudanças ao cliente.

```json
{
  "id": "uuid-do-pedido",
  "action": "edit_items",
  "items_to_add": [
    { "product_id": "uuid", "quantity": 2, "unit_price": 15.00 }
  ],
  "items_to_remove": ["uuid-do-item-1", "uuid-do-item-2"],
  "items_to_update": [
    { "item_id": "uuid-do-item-3", "quantity": 5 }
  ]
}
```

| Campo | Descrição |
|---|---|
| `items_to_add` | Novos itens a inserir no pedido (valida estoque) |
| `items_to_remove` | IDs de `order_items` a remover (estoque é restaurado) |
| `items_to_update` | Ajuste de quantidade de itens existentes (valida delta de estoque) |

> Todos os três campos são opcionais, mas ao menos um deve ter conteúdo.

**Resposta 200:**
```json
{
  "organization_id": "uuid",
  "instance_name": "pedii_f988b66e",
  "order": {
    "id": "uuid",
    "status": "open",
    "awaiting_customer_approval": true,
    "order_items": [ /* lista completa e atualizada */ ]
  }
}
```

---

#### `customer_approved` — Registrar aprovação do cliente

**Exclusivo do agente** (retorna 403 se chamado com JWT de usuário).

Chamado pelo agente após o cliente confirmar os ajustes feitos pelo atendente na conversa do WhatsApp. Zera o flag `awaiting_customer_approval`, liberando o atendente para aceitar o pedido no dashboard.

```json
{ "id": "uuid-do-pedido", "action": "customer_approved" }
```

> Só funciona em pedidos com `status = open`. Retorna 422 se o pedido estiver em outro status.

**Resposta 200:**
```json
{
  "organization_id": "uuid",
  "instance_name": "pedii_f988b66e",
  "order": {
    "id": "uuid",
    "status": "open",
    "awaiting_customer_approval": false,
    "order_items": [ /* itens do pedido */ ]
  }
}
```

**Efeito no dashboard:** via Supabase Realtime, o badge "Ag. cliente" some automaticamente da listagem e o botão "Aceitar pedido" é desbloqueado para o atendente.

---

## Webhooks recebidos pelo agente (Pedii → agente)

O Pedii envia eventos HTTP POST ao agente quando o **lojista** realiza ações no dashboard.
O agente deve expor um endpoint para receber esses eventos e disparar mensagens no WhatsApp.

### Autenticação recebida

```
x-api-key: <AGENT_API_KEY>
x-internal-secret: <INTERNAL_WEBHOOK_SECRET>
```

---

### Evento: `order.status_changed`

Disparado quando o lojista muda o status de um pedido pelo dashboard.

> ⚠️ Mudanças feitas pelo próprio agente via `PATCH /orders` **não** disparam este evento (evita loop).

**Payload:**
```json
{
  "event": "order.status_changed",
  "order_id": "83781578-c102-4bd3-8f22-9ea6167887aa",
  "old_status": "open",
  "new_status": "accepted",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "instance_name": "pedii_f988b66e",
  "order": {
    "id": "83781578-c102-4bd3-8f22-9ea6167887aa",
    "status": "accepted",
    "total": 310.00,
    "shipping_value": 10.00,
    "stores": { "apelido": "Paraguaçu", "razao_social": "TM Farma - Paraguaçu" },
    "customers": { "nome": "Eloá Mariane da Luz", "whatsapp": "35982721736" },
    "order_items": [
      {
        "quantity": 15,
        "unit_price": 20.00,
        "subtotal": 300.00,
        "products": { "descricao": "Produto Teste Paraguaçu" }
      }
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

### Evento: `order.items_updated`

Disparado quando o **atendente no dashboard** edita os itens de um pedido `open` antes de aceitá-lo (via `PATCH /orders` com `action: edit_items`).

O agente deve usar este evento para comunicar ao cliente final as alterações realizadas (ex: remoção de item sem estoque, substituição de produto, ajuste de quantidade).

**Payload:**
```json
{
  "event": "order.items_updated",
  "order_id": "83781578-c102-4bd3-8f22-9ea6167887aa",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "instance_name": "pedii_f988b66e",
  "changes": {
    "items_added": [
      { "product_id": "uuid", "quantity": 2, "unit_price": 15.00 }
    ],
    "items_removed": ["uuid-do-item-removido"],
    "items_updated": [
      { "item_id": "uuid-do-item", "quantity": 3 }
    ]
  },
  "order": {
    "id": "83781578-c102-4bd3-8f22-9ea6167887aa",
    "status": "open",
    "total": 95.52,
    "stores": { "apelido": "Varginha" },
    "customers": { "nome": "João Silva", "whatsapp": "5531999990000" },
    "order_items": [ /* lista completa e atualizada */ ]
  }
}
```

> O campo `order.order_items` reflete o estado final do pedido após todas as alterações.
> Use `changes` para construir uma mensagem legível ao cliente descrevendo o que foi modificado.

---

## Campo `awaiting_customer_approval`

O campo `awaiting_customer_approval` (boolean) está presente em todos os pedidos e controla o fluxo de aprovação quando o atendente edita itens antes do aceite.

| Valor | Significado |
|---|---|
| `false` (padrão) | Pedido normal — atendente pode aceitar livremente |
| `true` | Pedido editado pelo atendente — bloqueado aguardando confirmação do cliente |

**Ciclo de vida do flag:**
1. **`edit_items`** → seta `true` automaticamente + dispara webhook `order.items_updated`
2. **Cliente confirma** na conversa → agente chama `customer_approved` → flag volta a `false`
3. **Dashboard** atualiza em tempo real via Realtime: badge "Ag. cliente" some, botão "Aceitar" é liberado

---

## Comportamento de estoque (automático)

Todos os updates de estoque são gerenciados automaticamente por triggers no banco:

| Evento | Impacto no estoque |
|---|---|
| Item adicionado ao pedido (INSERT) | Estoque reduzido |
| Item removido do pedido (DELETE) | Estoque restaurado |
| Quantidade do item alterada (UPDATE) | Ajuste proporcional (delta) |
| Pedido cancelado | Estoque de todos os itens restaurado |

A validação de estoque é aplicada no INSERT (trigger `BEFORE INSERT`) e na API antes de qualquer operação — impossibilitando criar ou editar itens com quantidade superior ao disponível.

---

## Códigos de erro

| Status | Significado |
|---|---|
| `400` | Parâmetros obrigatórios ausentes ou inválidos |
| `401` | Token ausente ou inválido |
| `403` | Acesso negado (ex: loja de outra organização) |
| `404` | Recurso não encontrado |
| `422` | Transição de status inválida; estoque insuficiente; endereço insuficiente para geocodificar; pedido aguardando aprovação do cliente (`AWAITING_CUSTOMER_APPROVAL`) |
| `500` | Erro interno |

---

## Referência OpenAPI

O arquivo `openapi.yaml` neste diretório segue o padrão OpenAPI 3.1 e pode ser importado
diretamente em ferramentas como Postman, Insomnia ou frameworks de agentes compatíveis
com OpenAPI (ex: Google Vertex AI Agent Builder, LangChain OpenAPI Toolkit).
