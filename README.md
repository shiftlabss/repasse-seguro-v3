# repasse-seguro-v3

Monorepo/Turborepo do Repasse Seguro v3, com workspace web em Vite/React e Supabase na raiz.

## Estrutura

- `apps/web`: frontend web principal
- `packages/ui`: workspace compartilhado para componentes e exports cross-app
- `supabase`: configuração local do Supabase CLI
- `docs` e `frameworks`: documentação funcional e operacional

## Setup

```bash
npm install
cp apps/web/.env.example apps/web/.env.local
npm run dev:web
```

## Scripts do Monorepo

```bash
npm run dev
npm run dev:web
npm run build
npm run lint
npm run typecheck
```

## Supabase

```bash
npm run supabase:start
npm run supabase:status
npm run supabase:stop
```

Para vincular o projeto local a um projeto cloud existente:

```bash
npm run supabase:link -- --project-ref <project-ref>
```

## Variáveis de ambiente do Web

Preencha `apps/web/.env.local` com:

```bash
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
SUPABASE_PROJECT_REF=
```
