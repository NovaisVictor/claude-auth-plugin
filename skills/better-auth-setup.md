---
description: "Configuração e setup do BetterAuth com Elysia e Drizzle. Use quando precisar configurar autenticação, BetterAuth, login, signup, session, ou inicializar auth num projeto."
---

# BetterAuth — Setup

## Dependências

```bash
bun add better-auth
```

## Arquivo principal: src/auth.ts

```typescript
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'
import { admin, magicLink, openAPI } from 'better-auth/plugins'
import { db } from './database/client'
import { env } from './env'

export const auth = betterAuth({
  basePath: '/auth',
  database: drizzleAdapter(db, {
    provider: 'pg',
    usePlural: true,
    camelCase: false,
  }),
  trustedOrigins: [
    env.APP_URL,
    'http://localhost:3000',
    // adicionar wildcard de preview se aplicável (e.g. 'https://front-app-*.vercel.app')
  ],
  advanced: {
    database: {
      generateId: false,
    },
    defaultCookieAttributes: {
      sameSite: env.NODE_ENV === 'development' ? 'lax' : 'none',
      secure: env.NODE_ENV !== 'development',
    },
  },
  plugins: [
    openAPI(),
    admin(),
    magicLink({
      sendMagicLink: async ({ email, token, url }, ctx) => {
        // TODO: implementar envio de email
      },
    }),
  ],
  emailAndPassword: {
    enabled: true,
    password: {
      hash: (password: string) => Bun.password.hash(password),
      verify: ({ password, hash }) => Bun.password.verify(password, hash),
    },
  },
  session: {
    expiresIn: 60 * 60 * 24 * 7, // 7 dias
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5, // 5 minutos
    },
  },
})
```

## Regras

- `generateId: false` — IDs são gerados pelo Drizzle schema (randomUUIDv7)
- `usePlural: true` — tabelas no plural (users, sessions, accounts)
- `camelCase: false` — column names em snake_case (padrão Drizzle)
- Password hashing via `Bun.password.hash/verify` (nativo, usa argon2)
- Session com cookie cache pra reduzir consultas ao banco
- Plugin `admin()` adiciona campo `role` no user (default: 'user')
- Plugin `openAPI()` expõe schema OpenAPI das rotas de auth

## Cookies cross-site (`defaultCookieAttributes`)

- **Dev (localhost):** `sameSite: 'lax'` + `secure: false`.
- **Prod / preview cross-site:** `sameSite: 'none'` + `secure: true`.

Sem isso, login passa em dev e a sessão "some" em prod (cookie rejeitado pelo browser).

## `trustedOrigins`

Lista obrigatória para BetterAuth aceitar requisições. Inclui:

- `env.APP_URL` (URL do frontend de produção)
- `http://localhost:3000` (frontend dev)
- Wildcard de preview se houver (ex: `'https://front-app-*.vercel.app'`)

Sem isso, BetterAuth retorna `403 INVALID_ORIGIN`. Sempre via env var — nunca hardcoded.

**Troubleshooting:**
- 403 `INVALID_ORIGIN` → falta entrada em `trustedOrigins`.
- Login OK mas sessão "some" entre requests → falta `defaultCookieAttributes`.

`trustedOrigins` e a config de CORS do Elysia precisam ser **coerentes** — ver skill `elysia-auth-plugin`.

## Variáveis de ambiente

```env
BETTER_AUTH_SECRET=your-secret-here
BETTER_AUTH_URL=http://localhost:3333
APP_URL=http://localhost:3000
NODE_ENV=development
```

Adicionar ao `src/env.ts`:

```typescript
const envSchema = z.object({
  // ...existentes
  BETTER_AUTH_SECRET: z.string().min(1),
  BETTER_AUTH_URL: z.string().url(),
  APP_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
})
```

## Gerar schemas do banco

Após configurar `auth.ts`, rodar:

```bash
bunx @better-auth/cli generate --config ./src/auth.ts
```

Isso gera as migrations. Depois mover os schemas gerados para arquivos separados por tabela em `src/database/schema/`.
