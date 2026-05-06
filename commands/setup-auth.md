---
description: Configurar BetterAuth num projeto existente com Bun + Elysia + Drizzle.
---

## Pré-requisitos

Verificar que o projeto já tem: Elysia, Drizzle configurado, `src/env.ts`, `src/database/client.ts`.

## Passos

### 1. Instalar dependência

```bash
bun add better-auth
```

### 2. Adicionar variáveis de ambiente

Em `.env`:

```env
BETTER_AUTH_SECRET=gerar-com-openssl-rand-base64-32
BETTER_AUTH_URL=http://localhost:8080
APP_URL=http://localhost:3000
NODE_ENV=development
```

Em `src/env.ts`, adicionar ao schema:

```typescript
BETTER_AUTH_SECRET: z.string().min(1),
BETTER_AUTH_URL: z.string().url(),
APP_URL: z.string().url(),
NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
```

### 3. Criar src/auth.ts

Seguir o padrão da skill `better-auth-setup`. Garantir que o `auth.ts` gerado inclui:

- `trustedOrigins` lendo `env.APP_URL` + `localhost:3000` + wildcard de preview se aplicável
- `advanced.defaultCookieAttributes` com `sameSite`/`secure` condicionados a `env.NODE_ENV`

### 4. Criar schemas Drizzle

Criar os arquivos de schema seguindo a skill `auth-schemas`. Os 4 sempre obrigatórios:

- `src/database/schema/users.ts`
- `src/database/schema/sessions.ts`
- `src/database/schema/accounts.ts`
- `src/database/schema/verifications.ts`

Adicionar conforme os plugins ativos:

- `organizations.ts`, `members.ts`, `invitations.ts` — se usar `organization()` (ver skill `organization-plugin`)
- `pass-keys.ts` — se usar `passkey()`
- `two-factors.ts` — se usar `twoFactor()`

Atualizar `src/database/schema/index.ts`.

### 5. Gerar e aplicar migrations

```bash
bun run db:generate
bun run db:migrate
```

### 6. Criar plugin Elysia

Criar `src/http/plugins/better-auth.ts` seguindo a skill `elysia-auth-plugin`.

### 7. Configurar CORS coerente

Instalar `@elysiajs/cors` (`bun add @elysiajs/cors`) e configurar em `src/index.ts` com a **mesma** lista de origens de `trustedOrigins`. Ver skill `elysia-auth-plugin` para o snippet completo. Sempre `credentials: true`.

### 8. Registrar no app

Em `src/index.ts`:

```typescript
import { cors } from '@elysiajs/cors'
import { betterAuthPlugin } from "./http/plugins/better-auth";

export const app = new Elysia()
  .use(cors({ /* ver skill elysia-auth-plugin */ }))
  .use(openapi())
  .use(betterAuthPlugin)
  // ...rotas
  .listen(env.PORT);
```

### 9. Informar próximos passos

1. Configurar Resend (`bun add resend`, `RESEND_API_KEY` no env, `src/lib/email.ts`) para `sendMagicLink` / `sendInvitationEmail`. Ver skill `better-auth-setup`.
2. Testar signup: `POST /auth/sign-up/email`
3. Testar login: `POST /auth/sign-in/email`
4. Proteger rotas com `auth: true`, `auth: 'manager'`, `auth: 'admin'`. Para multi-tenant, usar `activeOrg: true` (ver skill `organization-plugin`).
5. Em prod cross-site: confirmar que cookies estão sendo aceitos pelo browser (DevTools → Application → Cookies)
