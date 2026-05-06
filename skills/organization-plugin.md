---
description: "Plugin organization() do BetterAuth: multi-tenant com members, invitations, roles owner/admin/member. Use quando o projeto tem múltiplas organizações, convites por email, ou rotas org-scoped."
---

# BetterAuth Organization — Multi-tenant

Plugin oficial do BetterAuth que adiciona organizations, members, invitations e o conceito de "active organization" na sessão.

## Quando usar

- App multi-tenant onde cada usuário pertence a uma ou mais organizações.
- Recursos do app são escopados por organização (ex: produtos da org X não aparecem para usuários da org Y).
- Convites de membro por email com fluxo de aceite.

## Setup no `auth.ts`

```typescript
import { organization } from 'better-auth/plugins'
import { FROM_EMAIL, resend } from './lib/email'

export const auth = betterAuth({
  // ... outras configs
  plugins: [
    organization({
      allowUserToCreateOrganization: async (user) => {
        return user.role === 'admin'
      },
      sendInvitationEmail: async (data) => {
        const inviteLink = `${env.APP_URL}/invite/${data.id}`

        try {
          await resend.emails.send({
            from: FROM_EMAIL,
            to: data.email,
            subject: `You were invited to ${data.organization.name}`,
            html: `
              <p>You were invited to join <strong>${data.organization.name}</strong>.</p>
              <p><a href="${inviteLink}">Accept invitation</a></p>
            `,
          })
        } catch (error) {
          console.error('Failed to send invitation:', error)
          if (env.NODE_ENV === 'production') throw error
        }
      },
    }),
    // ... outros plugins
  ],
})
```

## Roles em multi-tenant

Quando o `organization()` está ativo, as roles relevantes são **por organização** (não por user):

| Role | Permissões |
|------|------------|
| `owner` | Tudo (incluindo deletar a org) |
| `admin` | Gerenciar members, invitations, settings |
| `member` | Acesso aos recursos da org |

Essas roles são **diferentes** das `user/manager/admin` do plugin `admin()`. As duas coexistem:

- `users.role` (`'user' | 'manager' | 'admin'`) — controla permissões globais (ex: criar org).
- `members.role` (`'owner' | 'admin' | 'member'`) — controla permissões dentro de uma org.

Nas rotas:

- Use macro `auth: 'admin'` quando a regra é por **role global** (`user.role`).
- Use macro `activeOrg: true` quando a regra é por **org ativa**. Verificações de role dentro da org ficam no use case (ex: `if (member.role !== 'owner') throw ...`).

## Tabelas Drizzle adicionadas

O plugin `organization()` espera 3 tabelas: `organizations`, `members`, `invitations`. Ver skill `auth-schemas` para os schemas.

## Sessão e org ativa

Após login, a sessão do BetterAuth contém:

```typescript
session.session.activeOrganizationId  // string | null
```

O frontend muda isso via `authClient.organization.setActive({ organizationId })`. O backend lê via macro `activeOrg`.

## Macro `activeOrg` (rotas org-scoped)

Definida no `betterAuthPlugin` (ver skill `elysia-auth-plugin`). Garante:

1. Sessão válida.
2. `session.activeOrganizationId` existe.
3. Injeta `organization: { id, slug }` no contexto da rota.

```typescript
export const createProductRoute = new Elysia()
  .use(betterAuthPlugin)
  .post(
    '/products',
    async ({ body, organization }) => {
      const useCase = makeCreateProductUseCase()
      const { product } = await useCase.execute({
        organizationId: organization.id,
        ...body,
      })
      return product
    },
    { activeOrg: true, body: z.object({ name: z.string() }) },
  )
```

**Regra:** `organizationId` **nunca** vem do body — sempre da sessão via macro. O cliente não escolhe a org no payload.

## Use case multi-tenant pattern

```typescript
export class CreateProductUseCase {
  constructor(private productsRepository: ProductsRepository) {}

  async execute({ organizationId, name, sku, priceInCents }: CreateProductRequest) {
    const existing = await this.productsRepository.findByOrganizationAndSku(
      organizationId, sku,
    )
    if (existing) throw new ProductAlreadyExistsError(sku)

    const product = await this.productsRepository.create({
      organizationId, name, sku, priceInCents,
    })
    return { product }
  }
}
```

Repositories filtram **sempre** por `organizationId`. O método `findById` deve validar org match no use case (ou retornar `null` se não bater).

## Convites — fluxo

1. Admin/owner clica "convidar" no frontend → `authClient.organization.inviteMember({ email, role })`.
2. BetterAuth chama o callback `sendInvitationEmail` com `{ id, email, organization, role, ... }`.
3. Email com link `${APP_URL}/invite/{id}` é enviado.
4. Convidado abre o link no frontend.
5. Frontend faz `GET /invitations/:id` (rota **pública** customizada — ver `roles-authorization`) pra mostrar "você foi convidado pra X com o email Y".
6. Se já tem conta, `authClient.organization.acceptInvitation({ id })`. Se não, signup-via-invite + accept.

A rota pública `GET /invitations/:id` é uma **rota custom** (não vem do BetterAuth). Retorna payload mínimo (organizationName, email, role, expiresAt, status) — sem `inviterId`. Ver skill `roles-authorization` (Public route pattern).

## Setup mandatório de 2FA (gate)

Quando `twoFactor()` ou `passkey()` estão ativos, o frontend deve forçar setup de 2FA antes de o user acessar qualquer rota org-scoped:

```tsx
// _app/layout.tsx (frontend)
const needsTwoFactorSetup =
  session.user.twoFactorEnabled === false &&
  !location.pathname.startsWith('/setup-2fa')

if (needsTwoFactorSetup) return <Navigate to="/setup-2fa" />
```

No backend, rotas que mexem com dados sensíveis (ex: regenerar backup codes, deletar passkey) usam macro `authWith2FA` em vez de `auth: true`. Ver skill `elysia-auth-plugin`.

## Regras

- `organizationId` **sempre** vem da sessão via macro `activeOrg` — nunca do body.
- Repositories de entidades multi-tenant **sempre** filtram por `organizationId` em todos os métodos.
- Rotas que listam recursos da org usam `activeOrg: true`. Rotas globais (admin) usam `auth: 'admin'`.
- Verificações de role dentro da org (`owner` vs `admin` vs `member`) ficam no use case, não no macro.
- `sendInvitationEmail` callback usa Resend. Se falhar em prod, lançar — log local em dev.
- Rota pública `/invitations/:id` deve retornar payload mínimo (sem `inviterId`).

## Auth-reviewer checks

- Rota multi-tenant sem macro `activeOrg` → ERRO.
- Use case multi-tenant aceitando `organizationId` no body em vez de via macro → ERRO.
- Repository multi-tenant sem filtro por `organizationId` → ERRO.
- Rota pública de invitation expondo `inviterId` ou `userId` → ERRO.
- Frontend sem gate de 2FA quando `twoFactor()`/`passkey()` ativos → AVISO.
