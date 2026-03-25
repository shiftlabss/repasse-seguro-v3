# repasse-seguro-v3

Base frontend do Repasse Seguro v3 com React, TypeScript, Vite, shadcn/ui e Supabase.

## Setup

```bash
npm install
cp .env.example .env
npm run dev
```

Para desenvolvimento local com Supabase CLI, o projeto também pode usar um `.env.local`.

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

## Variáveis de ambiente

Preencha o arquivo `.env` ou `.env.local` com:

```bash
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
SUPABASE_PROJECT_REF=
```
