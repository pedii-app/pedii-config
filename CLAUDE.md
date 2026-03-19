# Pedii — Instruções do Projeto

Pedii é um SaaS multi-tenant para automação de pedidos via WhatsApp para lojistas. Toda tarefa neste repositório deve seguir as skills abaixo, que definem os padrões de arquitetura, identidade visual, segurança e deploy do projeto.

## Skills Ativas

@.agents/skills/pedii-core.md
@.agents/skills/pedii-saas-fullstack.md
@.agents/skills/pedii-supabase.md
@.agents/skills/pedii-frontend-lp.md
@.agents/skills/pedii-github.md

## Estrutura do Repositório

```
pedii/
├── front/
│   ├── app/          # SaaS dashboard (React + Vite) → Cloudflare Pages: pedii-app
│   └── home/         # Landing page (React + Vite)   → Cloudflare Pages: pedii-home-lp
└── supabase/
    ├── schema.sql
    ├── migrations/
    └── functions/    # Edge Functions                → Supabase: pedii-saas
```

## Fluxo de Deploy (Automatizado via GitHub Actions)

O deploy de produção é **100% automatizado**. Nunca use wrangler ou supabase CLI manualmente para produção.

| O que deployar | Comando git |
|---|---|
| Dashboard (`front/app`) | `git push origin main:app-prd` |
| Landing Page (`front/home`) | `git push origin main:home-prd` |
| Edge Functions (`supabase/functions/**`) | `git push origin main:supabase-prd` |
| Migrations SQL | Aplicar manualmente no Supabase Dashboard |

### Fluxo padrão
```bash
# 1. Commit e push para main (sempre primeiro)
git add <arquivos>
git commit -m "feat: descrição"
git push origin main

# 2. Promover para produção (dispara GitHub Actions)
git push origin main:app-prd      # deploy do dashboard
git push origin main:home-prd     # deploy da landing page
git push origin main:supabase-prd # deploy das Edge Functions
```
