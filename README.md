# NestJs-Drizzle-ORM
## Integrating Drizzle ORM with NestJS

### 1. Installation

```bash
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg
```

> For MySQL, replace `pg` with `mysql2`. For SQLite, use `better-sqlite3`.

---

### 2. Project Structure

```
src/
├── database/
│   ├── database.module.ts
│   ├── database.providers.ts
│   ├── drizzle.provider.ts
│   └── schema/
│       ├── index.ts
│       └── users.ts
├── users/
│   ├── users.module.ts
│   ├── users.service.ts
│   └── users.controller.ts
├── app.module.ts
└── main.ts
drizzle.config.ts
```

---

### 3. Define Your Schema

```typescript
// src/database/schema/users.ts
import { pgTable, serial, varchar, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Infer types
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

```typescript
// src/database/schema/index.ts
export * from './users';
```

---

### 4. Create the Drizzle Provider

```typescript
// src/database/drizzle.provider.ts
import { drizzle, NodePgDatabase } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

export const DRIZZLE = Symbol('drizzle-connection');

export const drizzleProvider = [
  {
    provide: DRIZZLE,
    useFactory: async () => {
      const pool = new Pool({
        host: process.env.DB_HOST || 'localhost',
        port: Number(process.env.DB_PORT) || 5432,
        user: process.env.DB_USER || 'postgres',
        password: process.env.DB_PASSWORD || 'postgres',
        database: process.env.DB_NAME || 'mydb',
      });

      const db = drizzle(pool, { schema });
      return db;
    },
    // Export type: NodePgDatabase<typeof schema>
  },
];
```

---

### 5. Create the Database Module

```typescript
// src/database/database.module.ts
import { Global, Module } from '@nestjs/common';
import { drizzleProvider, DRIZZLE } from './drizzle.provider';

@Global()
@Module({
  providers: [...drizzleProvider],
  exports: [DRIZZLE],
})
export class DatabaseModule {}
```

---

### 6. Create a Service Using Drizzle

```typescript
// src/users/users.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { eq } from 'drizzle-orm';
import { DRIZZLE } from '../database/drizzle.provider';
import * as schema from '../database/schema';
import { users, NewUser, User } from '../database/schema/users';

@Injectable()
export class UsersService {
  constructor(
    @Inject(DRIZZLE)
    private db: NodePgDatabase<typeof schema>,
  ) {}

  async findAll(): Promise<User[]> {
    return this.db.select().from(users);
  }

  async findById(id: number): Promise<User | undefined> {
    const result = await this.db
      .select()
      .from(users)
      .where(eq(users.id, id));
    return result[0];
  }

  async create(data: NewUser): Promise<User> {
    const result = await this.db
      .insert(users)
      .values(data)
      .returning();
    return result[0];
  }

  async update(id: number, data: Partial<NewUser>): Promise<User> {
    const result = await this.db
      .update(users)
      .set(data)
      .where(eq(users.id, id))
      .returning();
    return result[0];
  }

  async delete(id: number): Promise<void> {
    await this.db.delete(users).where(eq(users.id, id));
  }
}
```

---

### 7. Create the Controller

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, ParseIntPipe } from '@nestjs/common';
import { UsersService } from './users.service';
import { NewUser } from '../database/schema/users';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findById(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findById(id);
  }

  @Post()
  create(@Body() data: NewUser) {
    return this.usersService.create(data);
  }

  @Put(':id')
  update(@Param('id', ParseIntPipe) id: number, @Body() data: Partial<NewUser>) {
    return this.usersService.update(id, data);
  }

  @Delete(':id')
  delete(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.delete(id);
  }
}
```

---

### 8. Create the Users Module

```typescript
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

---

### 9. Wire Everything in AppModule

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { DatabaseModule } from './database/database.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule,
    UsersModule,
  ],
})
export class AppModule {}
```

---

### 10. Drizzle Kit Config (Migrations)

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/database/schema/index.ts',
  out: './src/database/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    host: process.env.DB_HOST || 'localhost',
    port: Number(process.env.DB_PORT) || 5432,
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'postgres',
    database: process.env.DB_NAME || 'mydb',
  },
});
```

### Run migrations:

```bash
# Generate migration files
npx drizzle-kit generate

# Push schema directly (dev)
npx drizzle-kit push

# Run migrations (production)
npx drizzle-kit migrate
```

---

### 11. (Optional) Using ConfigService Instead of `process.env`

```typescript
// src/database/drizzle.provider.ts
import { ConfigService } from '@nestjs/config';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

export const DRIZZLE = Symbol('drizzle-connection');

export const drizzleProvider = [
  {
    provide: DRIZZLE,
    inject: [ConfigService],
    useFactory: async (configService: ConfigService) => {
      const pool = new Pool({
        host: configService.get<string>('DB_HOST'),
        port: configService.get<number>('DB_PORT'),
        user: configService.get<string>('DB_USER'),
        password: configService.get<string>('DB_PASSWORD'),
        database: configService.get<string>('DB_NAME'),
      });

      return drizzle(pool, { schema });
    },
  },
];
```

Make sure `ConfigModule` is imported before `DatabaseModule` in `AppModule`.

---

### 12. (Optional) Transactions

```typescript
// In any service
async transferExample() {
  await this.db.transaction(async (tx) => {
    await tx.update(accounts).set({ balance: 0 }).where(eq(accounts.id, 1));
    await tx.update(accounts).set({ balance: 100 }).where(eq(accounts.id, 2));
  });
}
```

---

### Key Takeaways

| Concept | Approach |
|---|---|
| **DI Token** | Use a `Symbol` (`DRIZZLE`) as the injection token |
| **Global Module** | `@Global()` on `DatabaseModule` makes it available everywhere |
| **Type Safety** | Use `NodePgDatabase<typeof schema>` for full type inference |
| **Schema Types** | Use `$inferSelect` / `$inferInsert` for TS types |
| **Migrations** | Use `drizzle-kit` CLI for generate/push/migrate |
| **Relations** | Define with `relations()` from `drizzle-orm` and use `db.query.*` (relational queries) |
