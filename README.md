# claude-auth-plugin

Plugin de autenticação com **BetterAuth + Elysia + Drizzle**. Email/password, magic link, roles (user/manager/admin).

## Instalação

### 1. Adicionar o marketplace (apenas na primeira vez)

```bash
claude plugin marketplace add NovaisVictor/claude-marketplace
```

### 2. Instalar o plugin

```bash
claude plugin install claude-auth-plugin@novais-plugins
```

Usado em conjunto com [claude-backend-plugin](https://github.com/NovaisVictor/claude-backend-plugin).

---

## Adicionando auth a um projeto existente

Pré-requisito: projeto backend já configurado com Elysia + Drizzle (ver claude-backend-plugin).

### 1. Instalar dependência

```bash
bun add better-auth
```

### 2. Adicionar variáveis de ambiente

No `.env`:

```env
BETTER_AUTH_SECRET=gerar-com-openssl-rand-base64-32
BETTER_AUTH_URL=http://localhost:3333
```

No `src/env.ts`, adicionar ao schema:

```typescript
BETTER_AUTH_SECRET: z.string().min(1),
BETTER_AUTH_URL: z.string().url(),
```

### 3. Criar src/auth.ts

```typescript
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { admin, magicLink, openAPI } from "better-auth/plugins";
import { db } from "./database/client";

export const auth = betterAuth({
  basePath: "/auth",
  database: drizzleAdapter(db, {
    provider: "pg",
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
    expiresIn: 60 * 60 * 24 * 7,
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5,
    },
  },
});
```

### 4. Criar schemas Drizzle

Criar um arquivo por tabela em `src/database/schema/`:

- `users.ts` — tabela de usuários (com campo `role`)
- `sessions.ts` — sessões ativas
- `accounts.ts` — contas de login (email, OAuth)
- `verifications.ts` — tokens de verificação

Ver skill `auth-schemas` do plugin para o conteúdo de cada arquivo.

Atualizar `src/database/schema/index.ts` com os imports.

### 5. Gerar e aplicar migrations

```bash
bun run db:generate
bun run db:migrate
```

### 6. Criar plugin Elysia

**src/http/plugins/better-auth.ts**

```typescript
import Elysia from "elysia";
import { auth } from "@/auth";

type UserRole = "user" | "manager" | "admin";

const ROLE_HIERARCHY: Record<UserRole, number> = {
  user: 0,
  manager: 1,
  admin: 2,
};

export const betterAuthPlugin = new Elysia({ name: "better-auth" })
  .mount(auth.handler)
  .macro({
    auth: {
      async resolve({ status, request: { headers } }, params: true | UserRole) {
        const session = await auth.api.getSession({ headers });

        if (!session) {
          return status(401, { message: "Unauthorized" });
        }

        const requiredRole = params === true ? "user" : params;
        const userRole = (session.user.role as UserRole) ?? "user";

        if (ROLE_HIERARCHY[userRole] < ROLE_HIERARCHY[requiredRole]) {
          return status(403, { message: "Forbidden" });
        }

        return {
          user: session.user,
          session: session.session,
        };
      },
    },
  });
```

### 7. Registrar no app

Em `src/index.ts`:

```typescript
import { betterAuthPlugin } from "./http/plugins/better-auth";

export const app = new Elysia()
  .use(openapi())
  .use(betterAuthPlugin)
  .listen(env.PORT);
```

### 8. Testar

```bash
# Signup
curl -X POST http://localhost:3333/auth/sign-up/email \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"test@test.com","password":"123456"}'

# Login
curl -X POST http://localhost:3333/auth/sign-in/email \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456"}'
```

---

## Protegendo rotas

```typescript
// Qualquer autenticado
.get('/me', handler, { auth: true })

// Apenas manager e admin
.get('/team', handler, { auth: 'manager' })

// Apenas admin
.delete('/admin/users/:id', handler, { auth: 'admin' })
```

## O que o plugin inclui

### Skills

| Skill                 | Ativada quando                            |
| --------------------- | ----------------------------------------- |
| `better-auth-setup`   | Configurar auth, deps, env                |
| `elysia-auth-plugin`  | Macro auth, role check, proteção de rotas |
| `auth-schemas`        | Schemas Drizzle das tabelas de auth       |
| `roles-authorization` | Hierarquia de roles, separação de rotas   |

### Commands

| Command                                                | Uso                                     |
| ------------------------------------------------------ | --------------------------------------- |
| `/setup-auth`                                          | Configurar auth do zero (guia completo) |
| `/protected-route POST /products create-product admin` | Criar rota protegida com role           |

### Agents

| Agent           | Função                                 |
| --------------- | -------------------------------------- |
| `auth-reviewer` | Review de segurança e auth (read-only) |
