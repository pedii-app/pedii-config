---
name: pedii-github
description: Especialista em versionamento e deploy automatizado via GitHub Actions para o Pedii.
triggers: ["github", "git", "deploy", "branch", "push", "commit", "actions", "ci", "cd", "pipeline"]
---

# Pedii Github
Para controlar versionamento de código seguir rigorosamente os padrões abaixo:

### 1. Credenciais
* **Organização GitHub:** `https://github.com/pedii-app`
* **Token:** armazenado em `$GITHUB_TOKEN` (variável de ambiente local — nunca commitar)

### 2. Repositórios
* **Frontend (app + LP):** `https://github.com/pedii-app/pedii-front`
* **Backend Supabase:** `https://github.com/pedii-app/pedii-supabase`

### 3. Fluxo de Deploy Automatizado (GitHub Actions)

O deploy de produção é **100% automatizado via GitHub Actions**. Nunca use wrangler ou supabase CLI manualmente para deploys de produção — use sempre o fluxo de branches abaixo.

#### pedii-front
| Branch de destino | Trigger | O que faz |
|---|---|---|
| `main` | desenvolvimento contínuo | integração, sem deploy |
| `app-prd` | push → GitHub Actions | build + deploy `front/app` → Cloudflare Pages `pedii-app` |
| `home-prd` | push → GitHub Actions | build + deploy `front/home` → Cloudflare Pages `pedii-home-lp` |

**Fluxo correto para enviar ao front:**
```bash
# 1. Commit e push para main
git add <arquivos>
git commit -m "feat: descrição"
git push origin main

# 2. Promover para produção (dispara o GitHub Actions automaticamente)
git push origin main:app-prd    # deploy do dashboard
git push origin main:home-prd   # deploy da landing page
```

#### pedii-supabase
| Branch de destino | Trigger | O que faz |
|---|---|---|
| `main` | desenvolvimento contínuo | integração, sem deploy |
| `supabase-prd` | push com mudanças em `functions/**` → GitHub Actions | deploy de todas Edge Functions → Supabase `pedii-saas` |

**Fluxo correto para Edge Functions:**
```bash
# 1. Commit e push para main
git add functions/
git commit -m "feat: descrição"
git push origin main

# 2. Promover para produção (dispara o GitHub Actions automaticamente)
git push origin main:supabase-prd
```

> ⚠️ O workflow de `supabase-prd` só dispara se houver mudanças em `functions/**`.
> Migrations SQL são aplicadas manualmente no Supabase Dashboard ou via `npx supabase db push`.

### 4. Padrão de Commits
Seguir Conventional Commits:
* `feat:` nova funcionalidade
* `fix:` correção de bug
* `ci:` mudanças em pipelines/actions
* `chore:` tarefas de manutenção
* `refactor:` refatoração sem mudança de comportamento
