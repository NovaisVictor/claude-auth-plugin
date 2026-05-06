# claude-auth-plugin

Plugin de autenticação com **BetterAuth + Elysia + Drizzle**. Email/password, magic link, passkeys, 2FA, organizations multi-tenant.

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

## O que muda na v1.2.0

- **Nova skill `organization-plugin`** — plugin `organization()` do BetterAuth com roles `owner/admin/member`, invitations via Resend, gate de 2FA setup.
- **Macro `activeOrg`** documentada em `elysia-auth-plugin.md` (rotas org-scoped que injetam `organization.id` no contexto).
- **`auth-schemas.md`** atualizada com schemas de `organizations`, `members`, `invitations`, `passkeys`, `two_factors` (além dos 4 core).
- **`better-auth-setup.md`** ganhou seção sobre Resend (email transacional) e referências aos plugins `passkey()` e `twoFactor()`.
- **Public route pattern** com payload mínimo (sem `inviterId`) já estava em `roles-authorization.md` — agora referenciado por `organization-plugin`.
- Macro `authWith2FA` documentada explicitamente para rotas sensíveis quando `passkey()` / `twoFactor()` estão ativos.
- `protected-route` command suporta `org` e `2fa` como scope além de `manager`/`admin`.
- Paths atualizados: `src/http/routes/{entity}/` (alinhado com `claude-backend-plugin` v1.2.0).

---

## Adicionando auth a um projeto existente

Pré-requisito: projeto backend já configurado com Elysia + Drizzle (ver claude-backend-plugin).

### 1. Instalar dependências

```bash
bun add better-auth resend
# Opcionais conforme features:
bun add @better-auth/passkey   # WebAuthn / passkeys
```

### 2. Adicionar variáveis de ambiente

No `.env`:

```env
BETTER_AUTH_SECRET=gerar-com-openssl-rand-base64-32
BETTER_AUTH_URL=http://localhost:8080
APP_URL=http://localhost:3000
RESEND_API_KEY=re_xxxxxxxxxxxxxxxx
NODE_ENV=development
```

No `src/env.ts`, adicionar ao schema:

```typescript
BETTER_AUTH_SECRET: z.string().min(1),
BETTER_AUTH_URL: z.string().url(),
APP_URL: z.string().url(),
RESEND_API_KEY: z.string().min(1),
NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
```

### 3. Criar `src/lib/email.ts`

```typescript
import { Resend } from 'resend'
import { env } from '@/env'

export const resend = new Resend(env.RESEND_API_KEY)
export const FROM_EMAIL = 'onboarding@resend.dev'
```

### 4. Criar `src/auth.ts`

Ver skill `better-auth-setup`. Highlights:

- `trustedOrigins` lendo `env.APP_URL` + `localhost` + wildcard de preview
- `advanced.defaultCookieAttributes` com `sameSite`/`secure` por `env.NODE_ENV`
- Plugins relevantes: `openAPI()`, `admin()`, `magicLink()`, opcionalmente `organization()`, `passkey()`, `twoFactor()`

### 5. Criar schemas Drizzle

Seguir skill `auth-schemas`. Core: `users`, `sessions`, `accounts`, `verifications`.

Adicionar conforme plugins ativos:
- `organization()` → `organizations`, `members`, `invitations` + `sessions.activeOrganizationId`
- `passkey()` → `passkeys`
- `twoFactor()` → `two_factors` + `users.twoFactorEnabled`

### 6. Criar `src/http/plugins/better-auth.ts`

Ver skill `elysia-auth-plugin`. Macros disponíveis:

- `auth: true | 'manager' | 'admin'` — sessão + role global
- `activeOrg: true` — sessão + org ativa (injeta `organization`)
- `authWith2FA: true` — sessão + 2FA habilitado

### 7. Configurar CORS coerente

```bash
bun add @elysiajs/cors
```

Em `src/index.ts`, configurar com a **mesma** lista de origens de `trustedOrigins`. Ver skill `elysia-auth-plugin`. Sempre `credentials: true`.

### 8. Gerar e aplicar migrations

```bash
bun run db:generate
bun run db:migrate
```

### 9. Testar

```bash
# Signup
curl -X POST http://localhost:8080/auth/sign-up/email \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"test@test.com","password":"123456"}'

# Login
curl -X POST http://localhost:8080/auth/sign-in/email \
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

// Multi-tenant (org ativa) — injeta `organization` no contexto
.get('/products', handler, { activeOrg: true })

// Sensível (exige 2FA)
.delete('/passkeys/:id', handler, { authWith2FA: true })
```

## O que o plugin inclui

### Skills

| Skill                  | Ativada quando                                                  |
| ---------------------- | --------------------------------------------------------------- |
| `better-auth-setup`    | Configurar auth, deps, env, Resend                              |
| `elysia-auth-plugin`   | Macros `auth`, `activeOrg`, `authWith2FA`, CORS coerente        |
| `auth-schemas`         | Schemas Drizzle (core + organization + passkey + 2FA)           |
| `roles-authorization`  | Hierarquia de roles, separação de rotas, public route pattern   |
| `organization-plugin`  | Multi-tenant: `organization()`, members, invitations, owner/admin/member |

### Commands

| Command                                                | Uso                                                  |
| ------------------------------------------------------ | ---------------------------------------------------- |
| `/setup-auth`                                          | Configurar auth do zero                              |
| `/protected-route POST /products create-product`       | Rota com `auth: true`                                |
| `/protected-route POST /products create-product org`   | Rota multi-tenant (`activeOrg: true`)                |
| `/protected-route GET /admin/users list-users admin`   | Rota admin (`auth: 'admin'`)                         |
| `/protected-route DELETE /passkeys/:id delete 2fa`     | Rota sensível (`authWith2FA: true`)                  |

### Agents

| Agent           | Função                                                              |
| --------------- | ------------------------------------------------------------------- |
| `auth-reviewer` | Review de cookies, trustedOrigins, CORS, multi-tenant, public routes |
