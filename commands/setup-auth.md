Configurar BetterAuth num projeto existente com Bun + Elysia + Drizzle.

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
BETTER_AUTH_URL=http://localhost:3333
```

Em `src/env.ts`, adicionar ao schema:
```typescript
BETTER_AUTH_SECRET: z.string().min(1),
BETTER_AUTH_URL: z.string().url(),
```

### 3. Criar src/auth.ts

Seguir o padrão da skill `better-auth-setup`.

### 4. Criar schemas Drizzle

Criar os 4 arquivos de schema seguindo a skill `auth-schemas`:
- `src/database/schema/users.ts`
- `src/database/schema/sessions.ts`
- `src/database/schema/accounts.ts`
- `src/database/schema/verifications.ts`

Atualizar `src/database/schema/index.ts`.

### 5. Gerar e aplicar migrations

```bash
bun run db:generate
bun run db:migrate
```

### 6. Criar plugin Elysia

Criar `src/http/plugins/better-auth.ts` seguindo a skill `elysia-auth-plugin`.

### 7. Registrar no app

Em `src/index.ts`:
```typescript
import { betterAuthPlugin } from './http/plugins/better-auth'

export const app = new Elysia()
  .use(openapi())
  .use(betterAuthPlugin)
  // ...rotas
  .listen(env.PORT)
```

### 8. Informar próximos passos

1. Implementar envio de email no `sendMagicLink`
2. Testar signup: `POST /auth/sign-up/email`
3. Testar login: `POST /auth/sign-in/email`
4. Proteger rotas com `auth: true`, `auth: 'manager'`, `auth: 'admin'`
