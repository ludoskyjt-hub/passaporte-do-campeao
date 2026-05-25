# Passaporte do Campeão: Destino Paris

PWA mobile-first de previsões da Copa do Mundo 2026, voltado ao Brasil. Usuários cadastram CPF, compram acesso (ticket diário R$30 ou Passaporte Ouro R$499), preveem placares de 104 partidas ao longo de 39 dias e concorrem a prêmios diários + Grand Prix Paris.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — API server (porta 5000 → proxy :80/api)
- `pnpm --filter @workspace/passaporte run dev` — Frontend Vite (porta 18110 → proxy :80)
- `pnpm run typecheck` — typecheck completo de todos os packages
- `pnpm run build` — typecheck + build de todos os packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerar hooks React Query e schemas Zod a partir da spec OpenAPI
- `pnpm --filter @workspace/db run push` — push do schema DB (dev only)
- `pnpm --filter @workspace/scripts run seed-matches` — re-seed dos 106 matchs
- Env obrigatória: `DATABASE_URL` — connection string Postgres

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5
- DB: PostgreSQL + Drizzle ORM
- Validação: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (a partir da spec OpenAPI)
- Build: esbuild (CJS bundle)
- Frontend: React + Vite, TailwindCSS, Wouter, TanStack Query

## Where things live

- `lib/api-spec/openapi.yaml` — source-of-truth do contrato de API
- `lib/api-client-react/src/generated/` — hooks React Query gerados pelo Orval
- `lib/api-zod/src/generated/` — schemas Zod gerados pelo Orval
- `lib/db/src/schema.ts` — schema do banco (Drizzle)
- `artifacts/api-server/src/routes/` — rotas Express
- `artifacts/passaporte/src/pages/` — páginas React
- `artifacts/passaporte/src/components/` — componentes reutilizáveis

## Architecture decisions

- Contract-first: spec OpenAPI → codegen de hooks e Zod antes de escrever backend/frontend
- JWT HS256 armazenado em `localStorage` com chave `passaporte_token`; getter registrado em `main.tsx` via `setAuthTokenGetter`
- PIX simulado: auto-confirmação após 30s via `setTimeout` na rota de pagamentos
- Prize pool: 75% da receita bruta diária, cap de exibição R$50k (surplus visível só no admin)
- Predictions lock: apostas bloqueadas 10 minutos antes do início de cada partida (campo `isLocked` retornado pela API)
- Assiduidade: contagem distinta de dias com previsão por usuário; Grand Prix exige 39 dias

## Product

- **Landing**: apresentação do produto, stats 39 dias / 104 partidas, CTA "Quero ir para Paris"
- **Registro**: 3 passos (dados pessoais, CPF+nascimento, senha) com validação de CPF e idade ≥18
- **Pagamento PIX**: ticket diário R$30 ou Passaporte Ouro R$499 com QR code simulado
- **Dashboard**: partidas do dia com inputs de placar, lock visual 10min antes, contador de prêmio, countdown para próxima partida
- **Leaderboard**: ranking diário com pontos e tiebreaker pelo minuto do primeiro gol
- **Perfil**: stats pessoais, Calendário Paris (grid 39 dias), Regulamento modal, botão Logout
- **Admin** (`/admin-secret-dashboard`): abas Visão Geral / Partidas / Paris; métricas, financeiro real, payout PIX, registro de resultados, exportação CSV dos elegíveis Grand Prix
- **WhatsApp**: botão flutuante em todas as páginas

## User preferences

- UI em português pt-BR
- Tema ouro/preto luxo (`gold-bg`, `gold-text`, `glass-card`, `glow-card`)
- Mobile-first (max-w container estreito)
- Sem emojis no código exceto flags de países nos dados de partida

## Gotchas

- `dateOfBirth` no banco é string (`YYYY-MM-DD`), convertida de `Date` no cadastro
- `req.params.paymentId` precisa de cast explícito para string no Express 5
- Sempre rodar `codegen` antes de `typecheck` ao alterar a spec OpenAPI
- Nunca `pnpm dev` na raiz — usar `restart_workflow` ou workflows do Replit
- Seed destroi previsões existentes (DELETE FROM predictions antes de DELETE FROM matches)
- Admin password: `process.env.ADMIN_PASSWORD ?? "admin2026"`

## Pointers

- Ver skill `pnpm-workspace` para estrutura do workspace, TypeScript e detalhes de packages
- Spec OpenAPI: `lib/api-spec/openapi.yaml`
- Seed de 106 matchs: `scripts/src/seed-matches.ts`
