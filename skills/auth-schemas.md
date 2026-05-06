---
description: "Schemas Drizzle para tabelas do BetterAuth: users, sessions, accounts, verifications, organizations, members, invitations, passkeys, two-factors. Use quando precisar criar, modificar ou consultar tabelas de autenticação, ou quando o BetterAuth gerar migrations."
---

# Auth Schemas — Drizzle

Os schemas do BetterAuth ficam separados por tabela em `src/database/schema/`, seguindo o mesmo padrão das entidades de domínio.

A lista de tabelas depende dos plugins ativos no `auth.ts`:

| Plugin | Tabelas adicionadas |
|---|---|
| Core BetterAuth | `users`, `sessions`, `accounts`, `verifications` |
| `admin()` | adiciona campos `role`, `banned`, `banReason`, `banExpires` em `users` |
| `organization()` | `organizations`, `members`, `invitations` (+ campo `activeOrganizationId` em `sessions`) |
| `passkey()` | `passkeys` |
| `twoFactor()` | `two_factors` (+ campo `twoFactorEnabled` em `users`) |

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
  // admin() plugin
  role: text('role').default('user').notNull(),
  banned: boolean('banned').default(false),
  banReason: text('ban_reason'),
  banExpires: timestamp('ban_expires'),
  // twoFactor() plugin
  twoFactorEnabled: boolean('two_factor_enabled').default(false),
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
  // organization() plugin
  activeOrganizationId: uuid('active_organization_id'),
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

## src/database/schema/organizations.ts (plugin organization)

```typescript
import { randomUUIDv7 } from 'bun'
import { relations } from 'drizzle-orm'
import {
  pgTable,
  text,
  timestamp,
  uniqueIndex,
  uuid,
} from 'drizzle-orm/pg-core'
import { invitations } from './invitations'
import { members } from './members'

export const organizations = pgTable(
  'organizations',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    name: text('name').notNull(),
    slug: text('slug').notNull().unique(),
    logo: text('logo'),
    metadata: text('metadata'),
    createdAt: timestamp('created_at').notNull(),
  },
  (table) => [uniqueIndex('organizations_slug_uidx').on(table.slug)],
)

export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  invitations: many(invitations),
}))
```

## src/database/schema/members.ts (plugin organization)

```typescript
import { randomUUIDv7 } from 'bun'
import { relations } from 'drizzle-orm'
import { index, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { organizations } from './organizations'
import { users } from './users'

export const members = pgTable(
  'members',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    organizationId: uuid('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    userId: uuid('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    role: text('role').default('member').notNull(), // 'owner' | 'admin' | 'member'
    createdAt: timestamp('created_at').notNull(),
  },
  (table) => [
    index('members_organizationId_idx').on(table.organizationId),
    index('members_userId_idx').on(table.userId),
  ],
)

export const membersRelations = relations(members, ({ one }) => ({
  organizations: one(organizations, {
    fields: [members.organizationId],
    references: [organizations.id],
  }),
  users: one(users, {
    fields: [members.userId],
    references: [users.id],
  }),
}))
```

## src/database/schema/invitations.ts (plugin organization)

```typescript
import { randomUUIDv7 } from 'bun'
import { relations } from 'drizzle-orm'
import { index, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { organizations } from './organizations'
import { users } from './users'

export const invitations = pgTable(
  'invitations',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    organizationId: uuid('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    email: text('email').notNull(),
    role: text('role'),
    status: text('status').default('pending').notNull(),
    expiresAt: timestamp('expires_at').notNull(),
    inviterId: uuid('inviter_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    createdAt: timestamp('created_at').defaultNow().notNull(),
  },
  (table) => [
    index('invitations_organizationId_idx').on(table.organizationId),
    index('invitations_email_idx').on(table.email),
  ],
)

export const invitationsRelations = relations(invitations, ({ one }) => ({
  organizations: one(organizations, {
    fields: [invitations.organizationId],
    references: [organizations.id],
  }),
  users: one(users, {
    fields: [invitations.inviterId],
    references: [users.id],
  }),
}))
```

## src/database/schema/pass-keys.ts (plugin passkey)

```typescript
import { randomUUIDv7 } from 'bun'
import { index, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { users } from './users'

export const passkeys = pgTable(
  'passkeys',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    name: text('name'),
    publicKey: text('public_key').notNull(),
    userId: uuid('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    credentialId: text('credential_id').notNull().unique(),
    counter: text('counter').notNull(),
    deviceType: text('device_type').notNull(),
    backedUp: text('backed_up').notNull(),
    transports: text('transports'),
    aaguid: text('aaguid'),
    createdAt: timestamp('created_at').defaultNow().notNull(),
  },
  (table) => [index('passkeys_userId_idx').on(table.userId)],
)
```

## src/database/schema/two-factors.ts (plugin twoFactor)

```typescript
import { randomUUIDv7 } from 'bun'
import { index, pgTable, text, timestamp, uuid } from 'drizzle-orm/pg-core'
import { users } from './users'

export const twoFactors = pgTable(
  'two_factors',
  {
    id: uuid('id')
      .primaryKey()
      .$defaultFn(() => randomUUIDv7()),
    secret: text('secret').notNull(),
    backupCodes: text('backup_codes').notNull(),
    userId: uuid('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    createdAt: timestamp('created_at').defaultNow().notNull(),
  },
  (table) => [index('two_factors_userId_idx').on(table.userId)],
)
```

## Atualizar barrel

Em `src/database/schema/index.ts`, adicionar tabelas + relations conforme os plugins ativos:

```typescript
import { accounts } from './accounts'
import { invitations, invitationsRelations } from './invitations'
import { members, membersRelations } from './members'
import { organizations, organizationsRelations } from './organizations'
import { passkeys } from './pass-keys'
import { sessions } from './sessions'
import { twoFactors } from './two-factors'
import { users } from './users'
import { verifications } from './verifications'

export const schema = {
  // Core
  users, accounts, sessions, verifications,
  // Organization
  organizations, members, invitations,
  // Passkey
  passkeys,
  // 2FA
  twoFactors,

  // Relations
  organizationsRelations, membersRelations, invitationsRelations,
}
```

## Regras

- Cada tabela do BetterAuth em arquivo separado (não num único auth-schema.ts)
- Seguem o mesmo padrão das entidades de domínio: randomUUIDv7, snake_case columns, timestamps
- Adicionar apenas as tabelas dos plugins efetivamente ativos no `auth.ts`
- `sessions.activeOrganizationId` só é necessário se `organization()` estiver ativo
- `users.twoFactorEnabled` só é necessário se `twoFactor()` estiver ativo
- `users.role`/`banned` só são necessários se `admin()` estiver ativo
- Ao rodar `bunx @better-auth/cli generate`, revisar o SQL gerado e ajustar os schemas se necessário
