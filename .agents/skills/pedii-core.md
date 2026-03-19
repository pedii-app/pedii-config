---
name: pedii-core-architect
description: Mantém a integridade da regra de negócio do Pedii: WhatsApp <> Estoque <> SaaS.
triggers: ["novo fluxo", "regra de negócio", "integração estoque", "segurança", "API"]
---

# Pedii Core Architect
Sempre que eu solicitar uma nova funcionalidade ou fluxo, valide contra estes pilares:

### 1. Conexão em Tempo Real
A consulta de estoque deve ser feita na API da loja física em tempo real para evitar venda de produtos sem estoque. A latência não deve exceder 2 segundos.

### 2. Automatização de Pedidos
Ao final de uma venda no WhatsApp, o pedido JSON deve ser enviado e persistido no dashboard SaaS do lojista imediatamente.

### 3. Segurança e Isolamento
Garanta que agentes inteligentes só possam acessar e interagir com o estoque e pedidos da loja com a qual estão autenticados.

### 4. Validação de Dados
Importe a skill `schema-validator-pro` para garantir que todo pedido que entra no SaaS siga o formato JSON correto.