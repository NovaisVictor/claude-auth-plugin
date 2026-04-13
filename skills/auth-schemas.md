---
description: "Schemas Drizzle para tabelas do BetterAuth: users, sessions, accounts, verifications. Use quando precisar criar, modificar ou consultar tabelas de autenticação, ou quando o BetterAuth gerar migrations."
---

# Auth Schemas — Drizzle

Os schemas do BetterAuth ficam separados por tabela em `src/database/schema/`, seguindo o mesmo padrão das entidades de domínio.

## src/database/schema/users.ts

```typescript
import { randomUUIDv7 } from 'bun'
import { pgTable, text, timestamp, uuid, boolean } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: uuid('id')
    .primaryKey()
    .$defaultFn(() => randomUUIDv7()),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified').default(false).notNull(),
  image: text('image'),
  role: text('role').default('user').notNull(),
  banned: boolean('banned').default(false),
  banReason: text('ban_reason'),
  banExpires: timestamp('ban_expires'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at')
    .defaultNow()
    .$onUpdate(() => new Date())
    .notNull(),
})
```

## src/database/schema/sessions.ts

```typescript
import { randomUUIDv7 } from 'bun'
import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { users } from './users'

export const sessions = pgTable('sessions', {
  id: uuid('id')
    .primaryKey()
    .$defaultFn(() => randomUUIDv7()),
  token: text('token').notNull().unique(),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  userId: uuid('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at')
    .defaultNow()
    .$onUpdate(() => new Date())
    .notNull(),
})
```

## src/database/schema/accounts.ts

```typescript
import { randomUUIDv7 } from 'bun'
import { index, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { users } from './users'

export const accounts = pgTable(
  'accounts',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    accountId: text('account_id').notNull(),
    providerId: text('provider_id').notNull(),
    userId: uuid('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    accessToken: text('access_token'),
    refreshToken: text('refresh_token'),
    idToken: text('id_token'),
    accessTokenExpiresAt: timestamp('access_token_expires_at'),
    refreshTokenExpiresAt: timestamp('refresh_token_expires_at'),
    scope: text('scope'),
    password: text('password'),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at')
      .defaultNow()
      .$onUpdate(() => new Date())
      .notNull(),
  },
  (table) => [index('accounts_userId_idx').on(table.userId)],
)
```

## src/database/schema/verifications.ts

```typescript
import { randomUUIDv7 } from 'bun'
import { pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'

export const verifications = pgTable('verifications', {
  id: uuid('id')
    .primaryKey()
    .$defaultFn(() => randomUUIDv7()),
  identifier: text('identifier').notNull(),
  value: text('value').notNull(),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at')
    .defaultNow()
    .$onUpdate(() => new Date())
    .notNull(),
})
```

## Atualizar barrel

Em `src/database/schema/index.ts`, adicionar todas as tabelas de auth:

```typescript
import { users } from './users'
import { sessions } from './sessions'
import { accounts } from './accounts'
import { verifications } from './verifications'

export const schema = {
  // Auth tables
  users,
  sessions,
  accounts,
  verifications,
  // Domain tables
  // ...
}
```

## Regras

- Cada tabela do BetterAuth em arquivo separado (não num único auth-schema.ts)
- Seguem o mesmo padrão das entidades de domínio: randomUUIDv7, snake_case columns, timestamps
- O campo `role` no users é adicionado pelo plugin `admin()` do BetterAuth
- Relations do BetterAuth são opcionais — o adapter gerencia as queries internamente
- Ao rodar `bunx @better-auth/cli generate`, revisar o SQL gerado e ajustar os schemas se necessário
- Não criar relations Drizzle para tabelas de auth — o BetterAuth usa o adapter diretamente
