---
description: "Plugin Elysia para BetterAuth: macro de autenticação, role check, proteção de rotas. Use quando precisar proteger rotas, verificar roles, acessar session/user em controllers, ou configurar o middleware de auth no Elysia."
---

# Elysia Auth Plugin — Macro e Role Check

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

## Regras

- O `betterAuthPlugin` deve ser registrado via `.use()` em cada rota ou grupo de rotas que precisa de auth
- Rotas públicas simplesmente não usam o macro `auth`
- Se precisar de dados diferentes por role → criar rotas separadas com use cases separados
- O macro injeta `user` e `session` no contexto da rota
- 401 = não autenticado (sem session válida)
- 403 = autenticado mas sem permissão (role insuficiente)

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
