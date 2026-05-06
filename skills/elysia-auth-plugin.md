---
description: "Plugin Elysia para BetterAuth: macro de autenticação, role check, proteção de rotas. Use quando precisar proteger rotas, verificar roles, acessar session/user em controllers, ou configurar o middleware de auth no Elysia."
---

# Elysia Auth Plugin — Macro e Role Check

## CORS coerente com BetterAuth

CORS e `trustedOrigins` (em `src/auth.ts`) precisam aceitar a **mesma** lista de origens. Se um aceita e o outro rejeita, login quebra.

```typescript
import { cors } from '@elysiajs/cors'
import { env } from '@/env'

app.use(cors({
  origin: (request) => {
    const origin = request.headers.get('origin')
    if (!origin) return false

    const allowed = [env.APP_URL, 'http://localhost:3000']
    if (allowed.includes(origin)) return true

    // Wildcard de preview (ajustar regex ao padrão do seu provider)
    if (/^https:\/\/front-app-[a-z0-9-]+\.vercel\.app$/.test(origin)) return true

    return false
  },
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization'],
}))
```

- `credentials: true` é **obrigatório** para cookies de sessão atravessarem CORS.
- A regex de preview deve casar com a URL real (ex: Vercel: `https://<project>-<hash>.vercel.app`).
- Se `trustedOrigins` aceita um domínio, o CORS também precisa aceitar — auditar em conjunto.

## Plugin: src/http/plugins/better-auth.ts

```typescript
import Elysia from 'elysia'
import { auth } from '@/auth'

type UserRole = 'user' | 'manager' | 'admin'

const ROLE_HIERARCHY: Record<UserRole, number> = {
  user: 0,
  manager: 1,
  admin: 2,
}

export const betterAuthPlugin = new Elysia({ name: 'better-auth' })
  .mount(auth.handler)
  .macro({
    auth: {
      async resolve({ status, request: { headers } }, params: true | UserRole) {
        const session = await auth.api.getSession({ headers })

        if (!session) {
          return status(401, { message: 'Unauthorized' })
        }

        const requiredRole = params === true ? 'user' : params
        const userRole = (session.user.role as UserRole) ?? 'user'

        if (ROLE_HIERARCHY[userRole] < ROLE_HIERARCHY[requiredRole]) {
          return status(403, { message: 'Forbidden' })
        }

        return {
          user: session.user,
          session: session.session,
        }
      },
    },
  })
```

## Uso nas rotas

```typescript
// Qualquer usuário autenticado
export const getProfileRoute = new Elysia()
  .use(betterAuthPlugin)
  .get('/me', ({ user }) => ({ user }), {
    auth: true,
  })

// Apenas manager e admin
export const listTeamRoute = new Elysia()
  .use(betterAuthPlugin)
  .get('/team', ({ user }) => { /* ... */ }, {
    auth: 'manager',
  })

// Apenas admin
export const listAllUsersRoute = new Elysia()
  .use(betterAuthPlugin)
  .get('/admin/users', ({ user }) => { /* ... */ }, {
    auth: 'admin',
  })
```

## Hierarquia de roles

```
admin  (nível 2) → acessa tudo
manager (nível 1) → acessa manager + user
user   (nível 0) → acessa apenas user
```

`auth: true` equivale a `auth: 'user'` — aceita qualquer autenticado.
`auth: 'manager'` aceita manager e admin.
`auth: 'admin'` aceita apenas admin.

> Em projetos multi-tenant (com plugin `organization()`), as roles são `owner/admin/member` e ficam por organização — não no `users.role`. Ver skill `organization-plugin`.

## Regras

- O `betterAuthPlugin` deve ser registrado via `.use()` em cada rota ou grupo de rotas que precisa de auth
- Rotas públicas simplesmente não usam o macro `auth`
- Se precisar de dados diferentes por role → criar rotas separadas com use cases separados
- O macro injeta `user` e `session` no contexto da rota
- 401 = não autenticado (sem session válida)
- 403 = autenticado mas sem permissão (role insuficiente)

## Macro `authWith2FA`

Quando o projeto habilita o plugin `twoFactor()` ou `passkey()`, criar uma macro adicional que exige 2FA verificado para rotas sensíveis:

```typescript
.macro({
  authWith2FA: {
    async resolve({ status, request: { headers } }) {
      const session = await auth.api.getSession({ headers })
      if (!session) return status(401, { message: 'Unauthorized' })
      if (!session.user.twoFactorEnabled) {
        return status(403, {
          message: '2FA_REQUIRED',
          error: 'You must enable two-factor authentication to access this resource',
        })
      }
      return { user: session.user, session: session.session }
    },
  },
})
```

Usar nas rotas sensíveis: `{ authWith2FA: true }`. Não substitui o macro `auth` — coexistem.

## Macro `activeOrg` (multi-tenant)

Quando o projeto usa o plugin `organization()`, rotas org-scoped precisam:

1. Sessão válida.
2. Organização ativa selecionada na sessão.
3. Injeção de `organization.id` no contexto pra o use case filtrar por `organizationId`.

```typescript
.macro({
  activeOrg: {
    async resolve({ status, request: { headers } }) {
      const session = await auth.api.getSession({ headers })
      if (!session) return status(401, { message: 'Unauthorized' })

      if (!session.session.activeOrganizationId) {
        return status(400, { message: 'No active organization' })
      }

      const organization = await auth.api.getFullOrganization({ headers })
      if (!organization) {
        return status(404, { message: 'Active organization not found' })
      }

      return {
        user: session.user,
        organization: {
          id: organization.id,
          slug: organization.slug,
        },
      }
    },
  },
})
```

Uso na rota:

```typescript
export const createProductRoute = new Elysia()
  .use(betterAuthPlugin)
  .post(
    '/products',
    async ({ body, organization, status }) => {
      const useCase = makeCreateProductUseCase()
      const { product } = await useCase.execute({
        organizationId: organization.id,  // ← injetado pelo macro
        ...body,
      })
      return status(201, product)
    },
    {
      activeOrg: true,
      body: z.object({ name: z.string().min(1), priceInCents: z.number().int() }),
    },
  )
```

**Regra:** todo CRUD multi-tenant usa `activeOrg: true` — nunca aceita `organizationId` no body (cliente não escolhe). Use cases recebem `organizationId` da sessão e filtram tudo por ele.

Status:
- 401 — sem sessão.
- 400 — sessão sem `activeOrganizationId`.
- 404 — `activeOrganizationId` aponta pra org inexistente.

## OpenAPI para rotas de auth

```typescript
let _schema: ReturnType<typeof auth.api.generateOpenAPISchema>
const getSchema = async () => (_schema ??= auth.api.generateOpenAPISchema())

export const AuthOpenAPI = {
  getPaths: (prefix = '/auth') =>
    getSchema().then(({ paths }) => {
      const reference: typeof paths = Object.create(null)
      for (const path of Object.keys(paths)) {
        const key = prefix + path
        reference[key] = paths[path]
        for (const method of Object.keys(paths[path])) {
          const operation = (reference[key] as any)[method]
          operation.tags = ['Auth']
        }
      }
      return reference
    }) as Promise<any>,
  components: getSchema().then(({ components }) => components) as Promise<any>,
} as const
```
