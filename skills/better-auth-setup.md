---
description: "Configuração e setup do BetterAuth com Elysia e Drizzle. Use quando precisar configurar autenticação, BetterAuth, login, signup, session, ou inicializar auth num projeto."
---

# BetterAuth — Setup

## Dependências

```bash
bun add better-auth
```

Plugins opcionais (instalar se for usar):

```bash
bun add @better-auth/passkey   # WebAuthn / passkeys
bun add resend                  # Envio de email transacional (magic link, invites)
```

## Arquivo principal: src/auth.ts

```typescript
import { passkey } from '@better-auth/passkey'
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'
import {
  admin,
  magicLink,
  openAPI,
  organization,
  twoFactor,
} from 'better-auth/plugins'
import { db } from './database/client'
import { env } from './env'
import { FROM_EMAIL, resend } from './lib/email'

const APP_NAME = 'My App'

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
    // Habilitar conforme as features do projeto:
    // organization({ ... }),  // ver skill `organization-plugin`
    // passkey({ rpID: new URL(env.APP_URL).hostname, rpName: APP_NAME, origin: env.APP_URL }),
    // twoFactor({ issuer: APP_NAME }),
    magicLink({
      sendMagicLink: async ({ email, token, url }) => {
        await resend.emails.send({
          from: FROM_EMAIL,
          to: email,
          subject: `Sign in to ${APP_NAME}`,
          html: `<a href="${url}">Click here to sign in</a>`,
        })
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

## Plugins disponíveis

| Plugin | Quando usar | Skill de referência |
|---|---|---|
| `openAPI()` | Sempre — expõe spec do auth | — |
| `admin()` | Quando há campo `role` no users (default `'user'`) | `roles-authorization` |
| `magicLink({ sendMagicLink })` | Login passwordless via email | (este) |
| `organization()` | Multi-tenant (membros, convites, roles `owner/admin/member`) | `organization-plugin` |
| `passkey({ rpID, rpName, origin })` | WebAuthn / biometria | (este) |
| `twoFactor({ issuer })` | TOTP de 6 dígitos + backup codes | (este) |

Quando `passkey()` ou `twoFactor()` estão ativos, o frontend deve forçar setup de 2FA via gate em `_app/layout.tsx`. Ver `organization-plugin` (backend macro `authWith2FA`) e a skill `organization-frontend` do plugin frontend.

## Regras

- `generateId: false` — IDs são gerados pelo Drizzle schema (randomUUIDv7)
- `usePlural: true` — tabelas no plural (users, sessions, accounts)
- `camelCase: false` — column names em snake_case (padrão Drizzle)
- Password hashing via `Bun.password.hash/verify` (nativo, usa argon2)
- Session com cookie cache pra reduzir consultas ao banco
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

## Email transacional (Resend)

Magic link, invite emails e password reset disparam emails. Padrão do projeto: Resend.

```bash
bun add resend
```

Adicionar `RESEND_API_KEY` ao `.env` e ao `src/env.ts`:

```typescript
RESEND_API_KEY: z.string().min(1),
```

Criar `src/lib/email.ts`:

```typescript
import { Resend } from 'resend'
import { env } from '@/env'

export const resend = new Resend(env.RESEND_API_KEY)

export const FROM_EMAIL = 'onboarding@resend.dev'
```

Substituir `'onboarding@resend.dev'` por um sender verificado no domínio do projeto antes de produção.

Os templates HTML ficam inline no callback do BetterAuth (`sendMagicLink`, `sendInvitationEmail`) — para algo mais elaborado, extrair para `src/lib/email-templates/{name}.ts`.

## Variáveis de ambiente

```env
BETTER_AUTH_SECRET=your-secret-here
BETTER_AUTH_URL=http://localhost:8080
APP_URL=http://localhost:3000
RESEND_API_KEY=re_xxxxxxxxxxxxxxxx
NODE_ENV=development
```

Adicionar ao `src/env.ts`:

```typescript
const envSchema = z.object({
  // ...existentes
  BETTER_AUTH_SECRET: z.string().min(1),
  BETTER_AUTH_URL: z.string().url(),
  APP_URL: z.string().url(),
  RESEND_API_KEY: z.string().min(1),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
})
```

## Gerar schemas do banco

Após configurar `auth.ts`, rodar:

```bash
bunx @better-auth/cli generate --config ./src/auth.ts
```

Isso gera as migrations. Depois mover os schemas gerados para arquivos separados por tabela em `src/database/schema/` (ver skill `auth-schemas`).
