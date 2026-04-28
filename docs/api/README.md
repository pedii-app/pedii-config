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
   (Evolution fornece: instance_name + whatsapp do cliente)
        ↓
2. GET /resolve-org?instance_name=<nome_da_instancia>
   → Resolve organization_id a partir do nome da instância Evolution
   → Usado em todas as chamadas subsequentes
        ↓
2.5. GET /conversations?thread_id=<instance_name>:<whatsapp_do_cliente>
   → Verifica se a conversa está em handoff (lojista assumiu o atendimento)
   → Se handoff_active: true → agente NÃO processa a mensagem, apenas aguarda
     (o lojista está atendendo diretamente; o agente entra em modo silencioso)
   → Se handoff_active: false → agente segue o fluxo normalmente
        ↓
3. GET /customers?organization_id=X&whatsapp=<numero_do_cliente>
   → O agente já sabe o whatsapp — consulta imediatamente sem pedir nada ao cliente

   ┌──────────────────────────────────────────────────────┐
   │ CLIENTE ENCONTRADO                                   │
   │ a) Se tem 1 endereço: confirma se ainda está correto │
   │    → Se mudou: PATCH /customers add_address (novo)   │
   │      ou POST /customers (atualiza endereço padrão)   │
   │ b) Se tem múltiplos endereços: apresenta lista e     │
   │    pergunta em qual deseja receber o pedido          │
   │ c) CEP do endereço escolhido disponível para step 4  │
   └──────────────────────────────────────────────────────┘
   ┌──────────────────────────────────────────────────────┐
   │ CLIENTE NÃO ENCONTRADO                               │
   │ → Pede apenas o CEP ao cliente                       │
   │ → Cadastro adiado — sem fricção no início            │
   └──────────────────────────────────────────────────────┘
        ↓ (em ambos os casos, CEP está disponível)
4. GET /nearest-stores?organization_id=X&cep=<CEP_do_cliente>
   → Retorna até 3 lojas dentro do raio de entrega, ordenadas por distância
   → Agente apresenta as opções e cliente escolhe a loja desejada
        ↓
5. GET /products?store_id=<loja_escolhida>&in_stock=true
   → Catálogo da loja escolhida; cliente monta o carrinho
        ↓
6. POST /calculate-freight { loja_id, cliente_id, address_id? }
   → Confirma raio, calcula frete e prazo
   → Retorna address_id resolvido (usar no step 9)
        ↓
7. [Somente se cliente NÃO estava cadastrado]
   → Agente pede nome e endereço completo para finalizar
   → POST /customers  ← cria o cadastro com endereço padrão
        ↓
8. GET /payment-methods?organization_id=<org_id>
   → Retorna as formas de pagamento aceitas pela organização
   → Agente pergunta ao cliente qual forma prefere e aguarda resposta
        ↓
9. POST /orders { ..., address_id, payment_method }
   ← address_id vem da resposta do calculate-freight (step 6)
   ← payment_method vem da escolha do cliente (step 8)
   ← garante que o endereço de entrega registrado é o mesmo usado no cálculo
        ↓
9a. [Webhook recebido] order.status_changed  ← lojista atualizou status
    → Agente notifica cliente via WhatsApp (instance_name + customer.whatsapp)

9b. [Webhook recebido] order.items_updated   ← atendente editou itens do pedido
    → Agente comunica mudanças ao cliente em linguagem natural
    → Cliente confirma ou recusa na conversa
        ↓
10. Cliente confirma ajuste:
    PATCH /orders { action: "customer_approved" }
    → awaiting_customer_approval volta a false
    → Atendente fica livre para aceitar o pedido no dashboard
```

---

## Endpoints de saída (agente → Pedii)

### `GET /conversations` — Verificar status de handoff (node `check_handoff`)

**Chamado no início de cada turno de conversa**, antes de processar qualquer mensagem do cliente. O agente verifica se um lojista assumiu o atendimento direto da conversa.

> Rota **exclusiva do agente** — somente `AGENT_API_KEY`. Não aceita JWT de usuário.

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `thread_id` | string | ✅ | `"{instance_name}:{whatsapp_do_cliente}"` |

**Resposta 200:**
```json
{
  "thread_id": "pedii_f988b66e:5511982122686",
  "handoff_active": true,
  "activated_by_name": "Wilson Junior",
  "started_at": "2026-04-04T14:30:00Z"
}
```

| Campo | Descrição |
|---|---|
| `handoff_active` | `true` se um lojista assumiu o atendimento; `false` caso contrário |
| `activated_by_name` | Nome do lojista que assumiu. `null` se `handoff_active: false` |
| `started_at` | Timestamp de início do handoff. `null` se `handoff_active: false` |

**Comportamento esperado do agente:**

```
SE handoff_active: true
  → NÃO processar a mensagem do cliente
  → NÃO enviar respostas via WhatsApp
  → Agente entra em modo silencioso até o lojista devolver o controle
  → (O webhook conversation.handoff com action="end" notifica quando retomar)

SE handoff_active: false
  → Processar normalmente — seguir fluxo do grafo
```

> **Implementação sugerida no grafo LangGraph:** criar um node `check_handoff` como primeiro node do grafo (antes de qualquer lógica de negócio). O node consulta esta rota e, se `handoff_active: true`, retorna um estado especial que faz o grafo encerrar o turno sem responder.

---

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

#### ⚠️ Formato obrigatório do `thread_id` no LangGraph

O `thread_id` usado pelo agente no LangGraph **deve ser composto** por `instance_name` + `:` + número WhatsApp do cliente:

```
thread_id = "{instance_name}:{whatsapp_do_cliente}"

# Exemplos:
"pedii_farmaciaA:5511999887766"
"pedii_farmaciaB:5511999887766"   ← mesmo cliente, org diferente → thread separado
```

**Por que isso é obrigatório:** o LangGraph isola o histórico de conversas pelo `thread_id`. Se dois agentes de orgs diferentes usarem apenas o número do cliente como `thread_id`, eles compartilhariam o mesmo checkpoint no banco — as mensagens ficariam misturadas e haveria contaminação de contexto entre orgs (quebra de isolamento multi-tenant).

Usando o formato composto, cada par `(instância, cliente)` tem seu próprio histórico isolado no LangGraph.

> O `thread_id` composto deve ser passado tanto no parâmetro `thread_id` desta rota quanto como `thread_id` do LangGraph checkpointer.

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
      "dentro_raio": true,
      "is_currently_open": true,
      "store_status": "open",
      "temporarily_closed_reason": null
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
      "dentro_raio": false,
      "is_currently_open": false,
      "store_status": "closed_schedule",
      "temporarily_closed_reason": null
    }
  ]
}
```

| Campo | Descrição |
|---|---|
| `nearest_stores` | Até 3 lojas **dentro do raio de atendimento**, ordenadas por `distancia_km` crescente |
| `dentro_raio` | `true` se `distancia_km ≤ raio_entrega_km`. `null` se a loja não tem raio configurado (incluída sem filtro) |
| `is_currently_open` | `true` se a loja está aberta **agora** (horário BRT), considerando fechamento temporário e grade horária |
| `store_status` | `"open"` \| `"closed_temporary"` \| `"closed_schedule"` \| `"no_hours_configured"` |
| `temporarily_closed_reason` | Motivo do fechamento temporário informado pelo lojista. `null` se não há fechamento ativo ou sem motivo |
| `lojas_sem_coordenadas` | Lojas que não puderam ser geocodificadas e foram excluídas do ranking |
| `message` | Presente apenas quando `nearest_stores` está vazio — explica que o cliente está fora do raio de todas as lojas |

**Valores de `store_status`:**

| Valor | Significado |
|---|---|
| `open` | Loja aberta no momento |
| `closed_schedule` | Fora do horário configurado para hoje, ou o dia está marcado como fechado |
| `closed_temporary` | Fechamento de emergência ativo (independente do horário) |
| `no_hours_configured` | Nenhum horário cadastrado — tratada como aberta |

> O agente deve informar ao cliente se a loja está fechada (`is_currently_open: false`) antes de prosseguir com o pedido, sugerindo tentar novamente no horário de funcionamento.

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
>
> **Normalização do WhatsApp:** o campo `whatsapp` é sempre armazenado e buscado no formato `DDI+DDD+Número` (ex: `5511982122686`). A função aceita qualquer formato como entrada (com ou sem DDI, com formatação) e normaliza automaticamente: remove caracteres não-numéricos e, se o número tiver ≤ 11 dígitos, adiciona o DDI `55` (Brasil). O agente **não precisa pré-formatar** o número antes de enviar.

**Resposta 200:**
```json
{
  "found": true,
  "customer": {
    "id": "uuid",
    "nome": "João Silva",
    "whatsapp": "5531999990000",
    "created_at": "2026-03-20T12:00:00Z",
    "customer_addresses": [
      {
        "id": "uuid-do-endereco",
        "label": "Casa",
        "logradouro": "Av. do Contorno",
        "numero": "40",
        "complemento": null,
        "bairro": "Santa Luiza",
        "cidade": "Varginha",
        "estado": "MG",
        "cep": "37062683",
        "lat": -21.5512,
        "lng": -45.4333,
        "is_default": true
      }
    ]
  }
}
```
Se não encontrado: `{ "found": false, "customer": null }`

> O array `customer_addresses` vem ordenado com o endereço padrão (`is_default: true`) primeiro.
> **Para o agente:** ao receber um cliente já cadastrado com múltiplos endereços, apresente as opções ao cliente e pergunte em qual deseja receber o pedido. Use o `id` do endereço escolhido como `address_id` nas chamadas de `calculate-freight` e `POST /orders`.

---

### `POST /customers` — Criar ou atualizar cliente

Upsert por `whatsapp` + `organization_id`. Cria o cliente e, opcionalmente, o endereço padrão em `customer_addresses`.

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
  "cep": "37062683",
  "address_label": "Casa"
}
```

> `store_id` ainda é aceito no lugar de `organization_id` (retrocompatibilidade), mas `organization_id` é o campo canônico. Ao menos um dos dois deve ser informado.
>
> **Normalização do WhatsApp:** mesmo comportamento do GET — qualquer formato é aceito e normalizado para `DDI+DDD+Número` antes do upsert. O campo `whatsapp` retornado na resposta já estará no formato normalizado.
>
> **Endereço:** se campos de endereço (`logradouro`, `cep` ou `cidade`) forem enviados, a função cria ou atualiza automaticamente o endereço padrão do cliente em `customer_addresses`. O campo `address_label` (opcional) define a identificação do endereço (ex: `"Casa"`, `"Trabalho"`).

**Resposta 201:** `{ "customer": { id, nome, whatsapp, created_at, customer_addresses: [...] } }`

---

### `PATCH /customers` — Gerenciar endereços do cliente

Permite ao agente adicionar, editar, definir como padrão ou excluir endereços de um cliente.

**Body base:**
```json
{ "customer_id": "uuid", "action": "add_address" }
```

#### `add_address` — Adicionar novo endereço

```json
{
  "customer_id": "uuid",
  "action": "add_address",
  "address": {
    "label": "Trabalho",
    "logradouro": "Rua XV de Novembro",
    "numero": "200",
    "bairro": "Centro",
    "cidade": "Varginha",
    "estado": "MG",
    "cep": "37002100",
    "is_default": false
  }
}
```

**Resposta 200:** `{ "address": { id, label, logradouro, ..., is_default } }`

#### `update_address` — Editar endereço existente

```json
{
  "customer_id": "uuid",
  "action": "update_address",
  "address_id": "uuid-do-endereco",
  "address": { "numero": "205", "complemento": "Sala 3" }
}
```

#### `set_default_address` — Definir endereço padrão

```json
{
  "customer_id": "uuid",
  "action": "set_default_address",
  "address_id": "uuid-do-endereco"
}
```

#### `delete_address` — Excluir endereço

```json
{
  "customer_id": "uuid",
  "action": "delete_address",
  "address_id": "uuid-do-endereco"
}
```

**Resposta 200:** `{ "success": true }`

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

### `GET /payment-methods` — Listar formas de pagamento aceitas

**Query params:**

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `organization_id` | UUID | **Obrigatório.** Organização da qual listar os métodos aceitos. |

**Resposta 200:**
```json
{
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "accepted_payment_methods": [
    { "value": "credit_card", "label": "Cartão de Crédito" },
    { "value": "debit_card",  "label": "Cartão de Débito" },
    { "value": "pix",         "label": "Pix" },
    { "value": "cash",        "label": "Dinheiro" }
  ]
}
```

**Valores possíveis de `value`:**

| value | label |
|---|---|
| `credit_card` | Cartão de Crédito |
| `debit_card` | Cartão de Débito |
| `pix` | Pix |
| `cash` | Dinheiro |

> O agente deve apresentar ao cliente apenas as opções retornadas nesta rota (a organização pode desabilitar formas que não aceita). O `value` escolhido pelo cliente deve ser enviado como `payment_method` no `POST /orders`.

---

### `POST /calculate-freight` — Calcular frete e prazo de entrega

**Body:**
```json
{
  "loja_id": "uuid-da-loja",
  "cliente_id": "uuid-do-cliente",
  "address_id": "uuid-do-endereco"
}
```

> `address_id` é **opcional**. Se omitido, a função usa o endereço marcado como `is_default` do cliente em `customer_addresses`. Passe `address_id` explicitamente quando o cliente escolher um endereço diferente do padrão.

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
  "estimated_delivery_minutes": 35,
  "pedido_minimo_entrega": 30.00,
  "frete_gratis_acima_de": 100.00,
  "_geocodificado": { "loja": false, "cliente": false }
}
```

| Campo | Descrição |
|---|---|
| `address_id` | UUID do endereço usado no cálculo (pode ser o padrão resolvido automaticamente ou o explicitamente informado). Use este valor no `address_id` do `POST /orders`. |
| `estimated_delivery_minutes` | Tempo estimado de entrega em minutos, calculado por interpolação linear entre `tempo_entrega_min` e `tempo_entrega_max` da loja, proporcionalmente à distância em relação ao raio. Arredondado para múltiplo de 5. `null` se `dentro_raio = false`. |

**Fórmula:**
```
tempo = tempo_min + (distancia_km / raio_km) × (tempo_max - tempo_min)
→ arredondado para múltiplo de 5 minutos
```

Exemplo: `tempo_min=20`, `tempo_max=60`, `raio=10km`, `distancia=3km` → `20 + (3/10)×40 = 32 → 30 min`

> Se `dentro_raio: false`, informar ao cliente que a loja não entrega no endereço dele.
> Use `estimated_delivery_minutes` para informar o prazo ao cliente durante o checkout: _"Entrega estimada em ~35 minutos"_.

---

### `POST /orders` — Criar pedido

**Body:**
```json
{
  "store_id": "uuid-da-loja",
  "customer_id": "uuid-do-cliente",
  "address_id": "uuid-do-endereco",
  "notes": "Entregar no período da tarde",
  "shipping_value": 9.72,
  "shipping_distance_km": 3.24,
  "shipping_within_radius": true,
  "payment_method": "pix",
  "items": [
    { "product_id": "uuid", "quantity": 2, "unit_price": 32.90 }
  ]
}
```

> Os campos `shipping_*` devem ser preenchidos com os valores retornados por `POST /calculate-freight`. O campo `estimated_delivery_minutes` é calculado automaticamente pelo servidor com base em `shipping_distance_km` e na configuração de tempo da loja — **não precisa ser enviado no body**.
>
> `address_id` é **opcional**. Se informado, é persistido como `delivery_address_id` no pedido. Se omitido, o servidor resolve automaticamente o endereço padrão do cliente. **Recomendado:** passe o `address_id` retornado por `POST /calculate-freight` para garantir consistência entre o endereço usado no cálculo e o endereço de entrega registrado no pedido.
>
> `payment_method` é **opcional** porém recomendado. Deve ser um dos valores retornados por `GET /payment-methods` para esta organização (`credit_card`, `debit_card`, `pix` ou `cash`). Se informado e não estiver entre as formas aceitas, retorna `422 INVALID_PAYMENT_METHOD`.

**Resposta 422 — forma de pagamento não aceita:**
```json
{
  "error": "Forma de pagamento não aceita por esta organização",
  "code": "INVALID_PAYMENT_METHOD",
  "accepted_payment_methods": ["pix", "cash"]
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

**Resposta 422 — loja fechada temporariamente:**
```json
{
  "error": "A loja está temporariamente fechada e não está aceitando pedidos.",
  "code": "STORE_TEMPORARILY_CLOSED",
  "reason": "Falta de energia"
}
```

**Resposta 422 — loja fechada hoje (sem horário de funcionamento):**
```json
{
  "error": "A loja não tem atendimento hoje.",
  "code": "STORE_CLOSED_TODAY"
}
```

**Resposta 422 — fora do horário de funcionamento:**
```json
{
  "error": "A loja está fora do horário de atendimento. Atendimento entre 08:00 e 18:00.",
  "code": "STORE_OUTSIDE_HOURS",
  "open_time": "08:00",
  "close_time": "18:00"
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
| `id` | uuid | Buscar pedido único (retorna itens completos) |
| `organization_id` | uuid | ⚠️ Filtrar pedidos da org (alternativa a `store_id`) |
| `store_id` | uuid | ⚠️ Filtrar por loja (resolve `organization_id` automaticamente) |
| `customer_id` | uuid | ⚠️ Filtrar por cliente (resolve `organization_id` automaticamente) |
| `status` | string | Filtrar por status (`open`, `accepted`, `shipped`, `received`, `cancelled`) |
| `date_from` | date | Data inicial (`2026-03-01`) |
| `date_to` | date | Data final (`2026-03-31`) |
| `page` | int | Paginação (padrão: 1) |
| `limit` | int | Itens por página (padrão: 50, máx: 100) |

> ⚠️ Para listagem, informe **ao menos um** de: `id`, `organization_id`, `store_id` ou `customer_id`.
> Quando apenas `customer_id` é fornecido, a função resolve o `organization_id` diretamente do cadastro do cliente — o agente não precisa enviá-lo explicitamente.
>
> **Exemplo de uso pelo agente** (buscar pedidos em aberto de um cliente):
> ```
> GET /orders?customer_id=<uuid>&status=open
> ```

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
    "customers": { "nome": "Eloá Mariane da Luz", "whatsapp": "5535982721736" },
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

## Webhooks recebidos pelo agente (Pedii → agente) — Handoff

### Visão geral: o que acontece com cada mensagem durante o handoff

Durante o handoff, dois tipos de mensagem trafegam pela conversa e exigem tratamentos distintos no LangGraph:

| Origem | Como chega ao agente | Como armazenar no LangGraph | Aparece no dashboard |
|---|---|---|---|
| **Cliente** envia mensagem | Fluxo normal (Evolution → agente) | `HumanMessage` — comportamento padrão, sem mudança | ✅ Balão branco (esquerda) |
| **Atendente** envia mensagem | Webhook `conversation.message` (Pedii → agente) | `AIMessage` com `additional_kwargs: { pedii_handoff_by: "<nome>" }` | ✅ Balão âmbar (direita) |

O LangGraph é a **única fonte da verdade** para o histórico de mensagens. O campo `pedii_handoff_by` no `additional_kwargs` é o que diferencia uma mensagem do atendente de uma resposta do agente IA no dashboard.

**Regra de ouro:**
- Mensagens do **cliente** durante handoff → `HumanMessage` como sempre. O agente só **não responde** (node `check_handoff` encerra o turno sem gerar resposta), mas a mensagem é gravada normalmente no checkpoint.
- Mensagens do **atendente** durante handoff → `AIMessage` com `additional_kwargs: { pedii_handoff_by: "<nome>" }`. O LangGraph acumula todas as mensagens com `add_messages` — elas jamais somem ao retomar com o mesmo `thread_id`.

---

### Evento: `conversation.handoff`

Disparado quando o lojista **assume ou devolve** uma conversa pelo dashboard SaaS.

Este webhook complementa o node `check_handoff`: enquanto a verificação ativa garante que o agente não interfira (polling por turno), o webhook garante **notificação imediata** — o agente pode reagir em tempo real sem esperar o próximo turno.

**Payload — lojista assume (`action: "start"`):**
```json
{
  "event": "conversation.handoff",
  "thread_id": "pedii_f988b66e:5511982122686",
  "action": "start",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "activated_by_name": "Wilson Junior",
  "timestamp": "2026-04-04T14:30:00Z"
}
```

**Payload — lojista devolve ao agente (`action: "end"`):**
```json
{
  "event": "conversation.handoff",
  "thread_id": "pedii_f988b66e:5511982122686",
  "action": "end",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "activated_by_name": "Wilson Junior",
  "timestamp": "2026-04-04T14:55:00Z"
}
```

| Campo | Descrição |
|---|---|
| `event` | Sempre `"conversation.handoff"` |
| `thread_id` | Identificador composto `"{instance_name}:{whatsapp}"` da conversa |
| `action` | `"start"` — lojista assumiu \| `"end"` — lojista devolveu ao agente |
| `activated_by_name` | Nome do lojista que realizou a ação |
| `timestamp` | ISO 8601 do momento da ação |

**Comportamento esperado ao receber o webhook:**

| `action` | O que o agente deve fazer |
|---|---|
| `"start"` | Registrar no estado da conversa que está em handoff. Não enviar mensagens ao cliente até receber `"end"`. |
| `"end"` | Remover flag de handoff do estado. Na próxima mensagem do cliente, processar normalmente. Pode (opcional) enviar uma mensagem de retorno como *"Olá! Estou de volta para ajudá-lo 😊"*. |

> O webhook pode falhar (timeout, indisponibilidade). A verificação ativa via `GET /conversations` no início de cada turno garante que o agente esteja sempre sincronizado mesmo se o webhook não for entregue.

---

### Evento: `conversation.message`

Disparado quando o **atendente humano** envia uma mensagem ao cliente pelo dashboard Conversas durante o handoff.

> **Garantia de handoff ativo:** a edge function valida que existe um `handoff_session` ativo antes de despachar este webhook — o agente nunca receberá `conversation.message` de conversas sem handoff ativo.

**Payload:**
```json
{
  "event": "conversation.message",
  "thread_id": "pedii_f988b66e:5511982122686",
  "content": "Olá! Deixa eu verificar isso para você.",
  "organization_id": "f988b66e-21e6-48d5-a90d-08f746600763",
  "activated_by_name": "Wilson Junior",
  "timestamp": "2026-04-04T14:35:00Z"
}
```

| Campo | Descrição |
|---|---|
| `event` | Sempre `"conversation.message"` |
| `thread_id` | Identificador da conversa — contém `instance_name` e `whatsapp` do cliente |
| `content` | Texto da mensagem a ser enviada ao cliente |
| `activated_by_name` | Nome do atendente que enviou |
| `timestamp` | ISO 8601 do envio |

**O que o agente deve fazer:**

```
1. Enviar o `content` ao cliente via Evolution API
   instance_name = thread_id.split(':')[0]
   whatsapp      = thread_id.split(':')[1]

2. Adicionar ao estado como AIMessage com additional_kwargs
   AIMessage(
     content = content,
     additional_kwargs = { "pedii_handoff_by": activated_by_name }
   )
```

#### Por que `AIMessage` com `additional_kwargs` é a abordagem correta

O LangGraph é a **única fonte da verdade** para o histórico de mensagens no Pedii. O dashboard lê exclusivamente do LangGraph via `get_conversation_messages`.

O campo `additional_kwargs.pedii_handoff_by` sinaliza ao dashboard que a mensagem foi enviada por um atendente humano (não pelo agente IA), permitindo diferenciação visual — o dashboard exibe em balão âmbar com o nome do atendente. Sem esse campo, a mensagem aparece como resposta normal do agente (balão verde).

**Por que não sumir ao retomar:**
O LangGraph usa `add_messages` como reducer para o canal `messages[]`, que é acumulativo. Cada novo checkpoint contém **todas as mensagens anteriores** — incluindo os `AIMessage` do handoff. Ao retomar com o mesmo `thread_id`, o agente carrega o checkpoint mais recente com o histórico completo. As mensagens do atendente nunca são perdidas.

#### Mensagens do cliente durante o handoff

Essas **não precisam de nenhum tratamento especial**. Quando o cliente envia uma mensagem enquanto o handoff está ativo, o fluxo é:

1. Mensagem chega ao agente pelo canal normal (Evolution API)
2. É adicionada ao estado como `HumanMessage` — comportamento padrão do LangGraph
3. O node `check_handoff` detecta `handoff_active: true` e encerra o turno **sem gerar resposta**
4. A `HumanMessage` fica gravada no checkpoint — o agente terá contexto completo ao retomar

O dashboard exibe essas mensagens normalmente (balão branco, lado esquerdo) via LangGraph, sem nenhuma lógica adicional.

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
| `422` | Transição de status inválida; estoque insuficiente; endereço insuficiente para geocodificar; pedido aguardando aprovação do cliente (`AWAITING_CUSTOMER_APPROVAL`); loja fechada temporariamente (`STORE_TEMPORARILY_CLOSED`); loja sem atendimento hoje (`STORE_CLOSED_TODAY`); fora do horário de funcionamento (`STORE_OUTSIDE_HOURS`) |
| `500` | Erro interno |

---

---

## Campanhas Promocionais — `POST /campaigns` e `GET /campaigns`

O módulo de campanhas permite ao lojista disparar mensagens promocionais ativas para a base de clientes via WhatsApp. O **agente é o único canal de envio** — mantendo todo o histórico no LangGraph.

---

### Visão geral do fluxo

```
Dashboard → POST /campaigns action=dispatch
          → edge function itera campaign_recipients
          → webhook "campaign.send" para cada destinatário
                    ↓
               Agente (GCP)
               ├─ envia imagem+texto via Evolution API
               ├─ persiste AIMessage no LangGraph com additional_kwargs.pedii_campaign_*
               └─ callback POST /campaigns action=recipient_status (sent | failed)

Evolution API → messages.update (fromMe=true, status=READ) → Agente
               → POST /campaigns action=recipient_status_by_message { waid, status }
                 ├── matched: false  → mensagem não é de campanha, ignorar
                 └── matched: true  → Pedii atualiza read_at automaticamente
```

---

### Webhook: `campaign.send`

Disparado pelo Pedii ao agente para cada destinatário. Autenticação: header `x-api-key: <AGENT_API_KEY>`.

**Payload:**
```json
{
  "event": "campaign.send",
  "organization_id": "<uuid>",
  "campaign_id": "<uuid>",
  "campaign_recipient_id": "<uuid>",
  "thread_id": "pedii_f988b66e:5511982122686",
  "product_id": "<uuid>",
  "product_name": "Dorflex 20cp",
  "message_text": "Promoção Dorflex hoje: R$ 8,90! Responda SIM para comprar.",
  "image_url": "https://.../signed-url",
  "instance_name": "pedii_f988b66e",
  "timestamp": "2026-04-14T10:00:00Z"
}
```

> `image_url` é uma signed URL válida por 1 hora. O agente deve baixar e repassar à Evolution API como mídia antes de enviar.

---

### O que o agente deve fazer ao receber `campaign.send`

1. **Enviar a mensagem** via Evolution API usando `instance_name` + destinatário = parte após `:` no `thread_id`.
   - Incluir a imagem (`image_url`) se presente.
   - Salvar o `waid` (WhatsApp message ID) retornado pela Evolution API.

2. **Persistir no LangGraph** como `AIMessage` no thread `thread_id`:
   ```json
   {
     "type": "AIMessage",
     "content": "<message_text>",
     "additional_kwargs": {
       "pedii_campaign_id": "<uuid>",
       "pedii_campaign_product_id": "<uuid>",
       "pedii_campaign_product_name": "Dorflex 20cp"
     }
   }
   ```
   Isso garante que o contexto do produto promovido fique no histórico do thread. Quando o cliente responder "quero comprar", o agente vê no estado do LangGraph qual `product_id` foi promovido e faz o match **determinístico** — sem fuzzy search por nome.

3. **Callback imediato** ao Pedii:
   ```
   POST /campaigns
   Authorization: Bearer <AGENT_API_KEY>
   {
     "action": "recipient_status",
     "campaign_recipient_id": "<uuid>",
     "status": "sent",
     "whatsapp_message_id": "<waid>"
   }
   ```
   Em caso de falha na Evolution API:
   ```json
   { "action": "recipient_status", "campaign_recipient_id": "<uuid>", "status": "failed", "error_message": "..." }
   ```

---

### Rastreamento de entrega e leitura (`messages.update`)

Quando a Evolution API disparar `messages.update` com `fromMe: true` e `status: "READ"`:

O agente **não precisa** distinguir se a mensagem é de campanha ou não. Basta chamar:

```json
POST /campaigns
{
  "action": "recipient_status_by_message",
  "whatsapp_message_id": "3EB0A1B2C3D4E5F6",
  "status": "read"
}
```

O Pedii faz o lookup internamente e retorna:
- `{ "matched": false }` — mensagem não pertence a campanha (notificação de pedido, handoff, etc.) → agente ignora
- `{ "matched": true, "campaign_recipient_id": "...", "updated": true }` — `read_at` registrado, dashboard atualiza em tempo real
- `{ "matched": true, "campaign_recipient_id": "...", "updated": false }` — já estava `read`, idempotente

> **Nunca retorna 404.** "Não encontrado" é o fluxo normal para a maioria das mensagens lidas — não é erro.

---

### Callback `POST /campaigns action=recipient_status_by_message`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `action` | string | sim | `"recipient_status_by_message"` |
| `whatsapp_message_id` | string | sim | ID da mensagem retornado pela Evolution API |
| `status` | string | sim | `read` \| `delivered` \| `sent` \| `failed` |

**Resposta 200:**
```json
{ "matched": false }
// ou
{ "matched": true, "campaign_recipient_id": "<uuid>", "updated": true }
```

---

### Callback `POST /campaigns action=recipient_status`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `action` | string | sim | `"recipient_status"` |
| `campaign_recipient_id` | uuid | sim | ID do destinatário |
| `status` | string | sim | `sent` \| `delivered` \| `read` \| `failed` |
| `whatsapp_message_id` | string | não | waid retornado pela Evolution API (enviar apenas no status `sent`) |
| `error_message` | string | não | Mensagem de erro (apenas se `status = failed`) |

---

### Consulta de metadados `GET /campaigns?campaign_id=X`

Retorna dados completos da campanha com signed URL da imagem. Útil se o agente precisar reprocessar.

```json
{
  "id": "<uuid>",
  "name": "Promoção Dorflex",
  "message_text": "...",
  "image_url": "https://.../signed-url",
  "product_id": "<uuid>",
  "products": { "id": "<uuid>", "descricao": "Dorflex 20cp" }
}
```

---

### Regra obrigatória de histórico

O agente **não pode** emitir `RemoveMessage` no canal `messages` do LangGraph. O trim/sanitize deve ser feito apenas em memória antes do call ao LLM, **sem escrever de volta ao estado**.

Caso contrário, as `AIMessages` com `pedii_campaign_product_id` serão permanentemente apagadas do checkpoint — e o match determinístico de produto falhará quando o cliente responder.

> Contexto: incidente identificado em 2026-04-14 onde os nodes `trim-history.before_model` e `sanitize-history.before_model` estavam emitindo `RemoveMessage`, apagando mensagens de campanha e de atendimento do histórico.

---

## Referência OpenAPI

O arquivo `openapi.yaml` neste diretório segue o padrão OpenAPI 3.1 e pode ser importado
diretamente em ferramentas como Postman, Insomnia ou frameworks de agentes compatíveis
com OpenAPI (ex: Google Vertex AI Agent Builder, LangChain OpenAPI Toolkit).
