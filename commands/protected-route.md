---
description: Gerar uma rota Elysia protegida por autenticação. Formato de $ARGUMENTS: `{METHOD} /{path} {use-case-kebab} {scope}`
---

O `scope` controla o tipo de proteção:

- omitido / `true` → `auth: true` (qualquer autenticado)
- `manager` / `admin` → `auth: 'manager'` ou `auth: 'admin'` (role global)
- `org` → `activeOrg: true` (rota multi-tenant — injeta `organization` no contexto)
- `2fa` → `authWith2FA: true` (rota sensível que exige 2FA setup)

Exemplos:

- `POST /products create-product` → `auth: true`
- `GET /admin/users list-all-users admin` → `auth: 'admin'`
- `GET /team/members list-team-members manager` → `auth: 'manager'`
- `POST /products create-product org` → `activeOrg: true` (org-scoped)
- `DELETE /passkeys/:id delete-passkey 2fa` → `authWith2FA: true`

## Nomes derivados

Mesma lógica do command `/route` do plugin-backend, com adição do scope.

- HTTP method: primeiro token
- Path: segundo token
- Use case name: terceiro token (kebab-case)
- Entity: primeiro segmento do path — pasta em `src/http/routes/{entity}/`
- Action variable: camelCase do use case + `Route`
- Factory function: `make` + PascalCase do use case + `UseCase`
- Arquivo de rota: `{use-case-kebab}.route.ts`

## Gerar route handler

`src/http/routes/{entity}/{use-case-kebab}.route.ts`:

```typescript
import Elysia from 'elysia'
import z from 'zod'
import { betterAuthPlugin } from '@/http/plugins/better-auth'
import { make{UseCasePascal}UseCase } from '@/use-cases/factories/make-{use-case-kebab}-use-case'

export const {actionVariable} = new Elysia()
  .use(betterAuthPlugin)
  .{method}(
    '{path}',
    async ({ body, params, user, organization, status }) => {
      const useCase = make{UseCasePascal}UseCase()

      const result = await useCase.execute({
        // TODO: mapear input
        // user.id disponível via macro auth/authWith2FA
        // organization.id disponível via macro activeOrg (multi-tenant)
      })

      // TODO: status e response
    },
    {
      {scopeKey}: {scopeValue},
      detail: {
        summary: 'TODO: descrição',
        tags: ['{Entity}s'],
      },
      // TODO: body, params, response schemas
    },
  )
```

Mapeamento de `scope` → opções do macro:

| scope | Chave | Valor | Contexto adicional |
|-------|-------|-------|--------------------|
| (omitido) ou `true` | `auth` | `true` | `user`, `session` |
| `manager` | `auth` | `'manager'` | `user`, `session` |
| `admin` | `auth` | `'admin'` | `user`, `session` |
| `org` | `activeOrg` | `true` | `user`, `organization` |
| `2fa` | `authWith2FA` | `true` | `user`, `session` |

## Sem try/catch

**Não capturar domain errors aqui.** O `errorHandlerPlugin` central mapeia (ver skill `error-handling` do plugin backend). A rota apenas `throw`s. Garanta que cada novo `DomainError` está registrado em `src/http/plugins/error-handler.ts:statusFor()`.

## Diferenças do `/route` padrão

- Importa `betterAuthPlugin` e usa `.use(betterAuthPlugin)`.
- Adiciona `auth | activeOrg | authWith2FA` nas opções.
- O contexto da rota tem `user`, `session` (e `organization` se `activeOrg`).
- `user.id` e `organization.id` podem ser passados como input do use case.

## Próximos passos

Informar ao usuário:

1. Preencher Zod schemas (body, params, response).
2. Mapear input — incluir `user.id` e/ou `organization.id` quando aplicável.
3. Verificar que cada `DomainError` lançado está mapeado em `error-handler.ts:statusFor()`.
4. Adicionar a rota no barrel `src/http/routes/{entity}/index.ts`.
5. `/e2e-test {METHOD} /{path}` para gerar teste E2E (com header de auth se aplicável).
