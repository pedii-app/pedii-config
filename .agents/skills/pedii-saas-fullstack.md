---
name: pedii-saas-fullstack
description: Especialista em arquitetura multi-tenant (Supabase + Cloudflare) e UI/UX (React + Vite + Tailwind) para a área logada do Pedii.
triggers: ["área logada", "dashboard", "supabase", "rls", "multi-tenant", "ui", "componente", "tabela"]
---

# Pedii SaaS Fullstack (Supabase + Cloudflare + React)
Ao gerar código para a área logada do lojista no Pedii, você deve integrar perfeitamente a segurança do back-end com a identidade visual do front-end, seguindo estas diretrizes:

### 1. Identidade Visual da Área Logada (Light Theme)
* **Fundo (Background):** Use fundos claros e limpos. Priorize branco (`bg-white`) para cartões de conteúdo e um cinza muito sutil (`bg-slate-50` ou `bg-gray-50`) para o fundo geral da aplicação, criando contraste.
* **Texto:** Cores escuras para máxima legibilidade (ex: `text-slate-900` para títulos, `text-slate-600` para descrições).
* **Acentos & CTAs (A Identidade Pedii):** O degradê verde-limão vibrante do logotipo (de `#72E890` para `#35C75A`) deve ser usado estritamente para ações principais (botão "Aprovar Pedido", "Adicionar Produto"), *badges* de status positivo ("Em Estoque") e links ativos na barra de navegação.
* **Componentes:** Utilize `lucide-react` para ícones e priorize uma estética de componentes minimalista (estilo *shadcn/ui*), com bordas arredondadas sutis (`rounded-lg` ou `rounded-md`) e sombras leves (`shadow-sm`).

### 2. Modelo de Dados Multi-Tenant (Hierarquia)
A estrutura de dados do Pedii segue esta hierarquia obrigatória:

```
organizations          ← tenant raiz (criada automaticamente no cadastro)
  └── lojas            ← cada org pode ter N lojas (CNPJ, endereço, raio entrega)
        ├── grupos_produto  ← agrupamento de produtos, isolado por loja
        └── produtos        ← estoque offline ou bridge ERP, isolado por loja
```

* Toda tabela transacional deve ter `organization_id` (isolamento de tenant).
* Tabelas que pertencem a uma loja específica devem ter também `loja_id`.
* O Row Level Security (RLS) do Supabase é inegociável. Todo script SQL deve incluir *Policies* usando `get_auth_user_organization_id()` e `is_auth_user_admin()`.

### 3. Front-end Protegido e Hospedagem (Cloudflare)
* O dashboard é hospedado no **Cloudflare Pages** (`pedii-app`).
* A navegação deve ser protegida (Protected Routes). O layout principal do dashboard não deve renderizar até que `supabase.auth.getSession()` confirme que o usuário está logado.
* Chamadas de dados no React devem lidar graciosamente com estados de carregamento (Skeletons) e erros.
* Sempre buscar `organization_id` e `loja_id` do perfil do usuário logado antes de qualquer insert.

### 4. Edge Functions para Lógica de Negócio
* Lógicas pesadas, comunicação com a API do WhatsApp ou processamento complexo de pedidos não ficam no cliente React. Use Supabase Edge Functions.

### 5. Skills da Comunidade a Importar
* `shadcn-ui-automator`: Para gerar tabelas de dados de estoque, modais e formulários com acessibilidade (a11y) nativa.
* `supabase-rls-master`: Para políticas de segurança PostgreSQL blindadas.

### 6. Deploy (Automatizado via GitHub Actions)
O deploy de produção é **automatizado**. Nunca faça deploy manual em produção.

| Ação | Comando |
|---|---|
| Deploy do dashboard | `git push origin main:app-prd` |
| Deploy da landing page | `git push origin main:home-prd` |
| Deploy de Edge Functions | `git push origin main:supabase-prd` (requer mudanças em `functions/**`) |
| Migrations | Aplicar manualmente no Supabase Dashboard |

### 7. Credenciais
* **Cloudflare Pages project_id app:** `pedii-app`
* **Cloudflare Pages project_url app:** `https://pedii-app.pages.dev`
