Gerar uma rota Elysia protegida por autenticação. Formato de $ARGUMENTS: `{METHOD} /{path} {use-case-kebab} {role}`

O `role` é opcional. Se omitido, default é `true` (qualquer autenticado).

Exemplos:
- `POST /products create-product` → auth: true
- `GET /admin/users list-all-users admin` → auth: 'admin'
- `GET /team/members list-team-members manager` → auth: 'manager'

## Nomes derivados

Mesma lógica do command `/route` do plugin-backend, com adição do role.

## Gerar route handler

`src/http/controllers/{domain}/{use-case-kebab}.ts`:

```typescript
import Elysia from 'elysia'
import z from 'zod'
import { betterAuthPlugin } from '@/http/plugins/better-auth'
import { make{UseCasePascal}UseCase } from '@/use-cases/factories/make-{use-case-kebab}-use-case'
import { ResourceNotFoundError } from '@/use-cases/errors/resource-not-found-error'

export const {actionVariable} = new Elysia()
  .use(betterAuthPlugin)
  .{method}(
    '{path}',
    async ({ body, params, user, status }) => {
      const useCase = make{UseCasePascal}UseCase()

      try {
        const result = await useCase.execute({
          // TODO: mapear input
          // user.id disponível via macro auth
        })

        // TODO: status e response
      } catch (err) {
        if (err instanceof ResourceNotFoundError) {
          return status(404, { message: err.message })
        }
        throw err
      }
    },
    {
      auth: {role_value},
      detail: {
        summary: 'TODO: descrição',
        tags: ['{Domain}'],
      },
      // TODO: body, params, response schemas
    },
  )
```

Onde `{role_value}` é:
- `true` se nenhum role foi especificado
- `'manager'` ou `'admin'` se especificado

## Diferenças do /route padrão

- Importa `betterAuthPlugin` e usa `.use(betterAuthPlugin)`
- Adiciona `auth: true | 'manager' | 'admin'` nas opções
- O contexto da rota tem `user` e `session` disponíveis
- `user.id` pode ser passado como input do use case

## Próximos passos

Informar ao usuário:
1. Preencher Zod schemas
2. Mapear input (incluindo `user.id` se necessário)
3. Ajustar error handling
4. Registrar no routes barrel
5. `/e2e-test {METHOD} /{path}` para gerar teste
