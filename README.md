# plugin-auth

Plugin de autenticação com **BetterAuth + Elysia + Drizzle**. Email/password, magic link, roles com plugin admin.

## Instalação

```bash
claude plugin install github:NovaisVictor/claude-auth-plugin
```

Usado em conjunto com `plugin-backend`.

## O que inclui

### Skills

| Skill                 | Ativada quando                                                                 |
| --------------------- | ------------------------------------------------------------------------------ |
| `better-auth-setup`   | Configurar BetterAuth, auth.ts, dependências, env                              |
| `elysia-auth-plugin`  | Macro auth, role check, proteção de rotas                                      |
| `auth-schemas`        | Schemas Drizzle das tabelas de auth (users, sessions, accounts, verifications) |
| `roles-authorization` | Padrões de autorização, hierarquia de roles, separação de rotas                |

### Commands

| Command                                                | Uso                     | O que faz                                                 |
| ------------------------------------------------------ | ----------------------- | --------------------------------------------------------- |
| `/setup-auth`                                          | Configurar auth do zero | Guia completo: deps, env, auth.ts, schemas, plugin Elysia |
| `/protected-route POST /products create-product admin` | Criar rota protegida    | Route handler com macro auth e role check                 |

### Agents

| Agent           | Modo             | Função                                                                |
| --------------- | ---------------- | --------------------------------------------------------------------- |
| `auth-reviewer` | plan (read-only) | Review de segurança: proteção de rotas, separação de concerns, config |

## Roles

| Role      | Nível | Acesso                      |
| --------- | ----- | --------------------------- |
| `user`    | 0     | Recursos próprios           |
| `manager` | 1     | Recursos do time + próprios |
| `admin`   | 2     | Tudo                        |

## Proteção de rotas

O macro Elysia aceita `true` ou uma role específica:

```typescript
// Qualquer autenticado
.get('/me', handler, { auth: true })

// Apenas manager e admin
.get('/team', handler, { auth: 'manager' })

// Apenas admin
.delete('/admin/users/:id', handler, { auth: 'admin' })
```

## Princípios

- Auth é concern de HTTP — use cases nunca acessam session/auth
- Roles diferentes com dados diferentes → rotas separadas com use cases separados
- Controle de acesso é binário: acessa (200) ou não (401/403)
- BetterAuth gerencia sessions via cookies HTTP-only
- Schemas de auth ficam separados por tabela, seguindo o padrão Drizzle

## Requisitos

- `plugin-backend` instalado
- BetterAuth (`better-auth`)
- Variáveis `BETTER_AUTH_SECRET` e `BETTER_AUTH_URL` configuradas
