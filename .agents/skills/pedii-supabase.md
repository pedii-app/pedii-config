---
name: pedii-supabase
description: Especialista em banco de dados Supabase para o Pedii.
triggers: ["criar tabela", "criar policy", "criar view", "criar trigger", "criar index", "criar function", "supabase", "postgres", "rls", "multi-tenant", "database", "migration"]
---

# Pedii Supabase
Ao gerar código ou design para o Pedii, siga rigorosamente estes padrões:

### 1. Multi-Tenancy e Segurança (RLS)
A hierarquia de dados é: `organizations → lojas → produtos/grupos_produto`.

* Toda tabela transacional deve ter `organization_id` (isolamento de tenant).
* Tabelas vinculadas a uma loja específica devem ter também `loja_id` (FK para `lojas`).
* O RLS é inegociável. Usar sempre as funções helper já existentes:
  * `get_auth_user_organization_id()` — retorna o org_id do usuário autenticado
  * `is_auth_user_admin()` — retorna true se role for owner ou admin
* Policies padrão por tabela:
  * `SELECT`: qualquer membro da org
  * `INSERT / UPDATE / DELETE`: somente admin/owner (`is_auth_user_admin()`)

### 2. Migrations
* Numerar sequencialmente: `001_`, `002_`, ..., `00N_`.
* Sempre incluir comentário de cabeçalho explicando o propósito.
* Migrations são aplicadas **manualmente** no Supabase Dashboard ou via:
  ```bash
  npx supabase db push --project-ref qiqwylcjoyztqebqglok
  ```

### 3. Deploy de Edge Functions (Automatizado via GitHub Actions)
* Push em `supabase-prd` com mudanças em `functions/**` dispara deploy automático.
* Para deploy manual de emergência:
  ```bash
  npx supabase functions deploy <nome> --project-ref qiqwylcjoyztqebqglok
  ```

### 4. Importação de Skills da Comunidade
* `supabase-rls-master`: Para políticas de segurança PostgreSQL blindadas.

### 5. Credenciais
* **Supabase project_id:** `qiqwylcjoyztqebqglok`
* **Supabase project_url:** `https://qiqwylcjoyztqebqglok.supabase.co`
* **Supabase publishable_key:** `sb_publishable_4mH2-JmYaUmxAxiPuUhJ4w_T9YoaTLn`
* **Supabase anon_key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InFpcXd5bGNqb3l6dHFlYnFnbG9rIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzM1NzgxNDEsImV4cCI6MjA4OTE1NDE0MX0.-F2hUHosFVA1CE3zSKHyciNSUrfLikW_KMF2Ht8EV0w`
* **Supabase db_password:** `X2OL8R&?'_Ah9q#%&Du`
* **Supabase db_user:** `postgres`
* **Supabase db_name:** `postgres`
* **Supabase db_port:** `5432`
* **Supabase db_host:** `qiqwylcjoyztqebqglok.supabase.co`
