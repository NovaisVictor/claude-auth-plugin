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

export const auth = betterAuth({
  basePath: '/auth',
  database: drizzleAdapter(db, {
    provider: 'pg',
    usePlural: true,
    camelCase: false,
  }),
  advanced: {
    database: {
      generateId: false,
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

## Variáveis de ambiente

```env
BETTER_AUTH_SECRET=your-secret-here
BETTER_AUTH_URL=http://localhost:3333
```

Adicionar ao `src/env.ts`:

```typescript
const envSchema = z.object({
  // ...existentes
  BETTER_AUTH_SECRET: z.string().min(1),
  BETTER_AUTH_URL: z.string().url(),
})
```

## Gerar schemas do banco

Após configurar `auth.ts`, rodar:

```bash
bunx @better-auth/cli generate --config ./src/auth.ts
```

Isso gera as migrations. Depois mover os schemas gerados para arquivos separados por tabela em `src/database/schema/`.
