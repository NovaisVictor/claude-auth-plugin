---
description: "Padrões de autorização e roles com BetterAuth admin plugin. Use quando precisar implementar controle de acesso, verificar permissões, proteger rotas por role, ou tomar decisões sobre separação de rotas por nível de acesso."
---

# Roles e Autorização

## Roles disponíveis

| Role | Nível | Acesso |
|------|-------|--------|
| `user` | 0 | Recursos próprios |
| `manager` | 1 | Recursos do time + próprios |
| `admin` | 2 | Tudo |

A role fica no campo `role` da tabela `users`, gerenciada pelo plugin `admin()` do BetterAuth. Default: `'user'`.

## Hierarquia

A verificação é por nível numérico. `auth: 'manager'` aceita manager (1) e admin (2), rejeita user (0).

```
admin  ≥ manager ≥ user
```

## Padrão de proteção nas rotas

```typescript
// Público — sem macro auth
.get('/public', handler)

// Qualquer autenticado
.get('/me', handler, { auth: true })

// Manager ou superior
.get('/team/members', handler, { auth: 'manager' })

// Apenas admin
.delete('/admin/users/:id', handler, { auth: 'admin' })
```

## Rotas separadas por role (recomendado)

Quando roles diferentes precisam de dados diferentes, criar rotas separadas com use cases separados:

```
GET /users/:id          → auth: true    → GetUserProfileUseCase (dados públicos)
GET /admin/users/:id    → auth: 'admin' → GetUserFullDetailsUseCase (todos os dados)
GET /team/members       → auth: 'manager' → FetchTeamMembersUseCase
GET /admin/users        → auth: 'admin'   → FetchAllUsersUseCase
```

## Regras

- Nunca colocar `if (role === 'admin')` dentro de um use case — criar use cases separados
- O controle de acesso é binário: tem permissão ou não (401/403)
- 401 = sem session válida (não autenticado)
- 403 = session válida mas role insuficiente (não autorizado)
- Rotas públicas simplesmente não usam o macro `auth`
- O user e session ficam disponíveis no contexto da rota via `({ user, session })`
- Para mudar a role de um user, usar a API admin do BetterAuth: `auth.api.setRole()`

## Acessar user nos use cases

O user ID vem do contexto da rota e é passado como input do use case:

```typescript
// No controller
const { product } = await useCase.execute({
  name: body.name,
  userId: user.id,  // vem do macro auth
})
```

O use case nunca acessa session/auth diretamente — recebe apenas os dados necessários via input.
