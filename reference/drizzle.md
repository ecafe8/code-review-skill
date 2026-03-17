# Drizzle ORM Code Review Guide

Drizzle 审查重点：Schema 类型安全定义、查询构造器正确用法、关系查询（RQB）与 JOIN 的选择、Mutations 安全性、事务完整性、以及 drizzle-kit 迁移管理规范。

> **版本基线**：drizzle-orm v0.30+ / drizzle-kit v0.20+，适用 PostgreSQL / MySQL / SQLite。

## 目录

- [Schema 定义](#schema-定义)
- [类型推断](#类型推断)
- [查询：SELECT](#查询select)
- [关系查询（RQB）](#关系查询rqb)
- [Mutations：INSERT / UPDATE / DELETE](#mutationsinsert--update--delete)
- [事务与 Batch API](#事务与-batch-api)
- [动态查询构建](#动态查询构建)
- [迁移管理（drizzle-kit）](#迁移管理drizzle-kit)
- [Review Checklists](#review-checklists)

---

## Schema 定义

### 基础表结构

```ts
// schema.ts
import { pgTable, serial, text, integer, timestamp, boolean, uniqueIndex, index } from 'drizzle-orm/pg-core';

// ✅ 推荐：明确列名、类型约束、默认值
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  age: integer('age').notNull(),
  verified: boolean('verified').notNull().default(false),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().$onUpdate(() => new Date()),
}, (table) => [
  index('name_idx').on(table.name),
  uniqueIndex('email_idx').on(table.email),
]);

// ✅ 外键关联
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: integer('author_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});

// ❌ 不声明类型约束——运行时报错，TypeScript 也无法保护
export const badTable = pgTable('bad', {
  id: serial('id'),          // 缺少 primaryKey()
  name: text('name'),        // 缺少 notNull()
  email: text('email'),      // 缺少 unique()
});
```

### MySQL / SQLite 方言

```ts
// MySQL
import { mysqlTable, serial, varchar, int } from 'drizzle-orm/mysql-core';

export const usersTable = mysqlTable('users', {
  id: serial().primaryKey(),
  name: varchar({ length: 255 }).notNull(),
  email: varchar({ length: 255 }).notNull().unique(),
});

// SQLite
import { sqliteTable, integer, text } from 'drizzle-orm/sqlite-core';

export const usersTable = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
});
```

### $onUpdate 自动更新列

```ts
// ✅ 使用 $onUpdate 自动更新时间戳，无需在每次更新时手动传入
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  // 行更新时自动计算新值
  updateCounter: integer('update_counter').default(1)
    .$onUpdateFn(() => sql`update_counter + 1`),
  updatedAt: timestamp('updated_at').$onUpdate(() => new Date()),
});

// ❌ 手动在每个 update 调用里传入 updatedAt
await db.update(users).set({ name: 'new', updatedAt: new Date() });  // 容易遗漏
```

### 索引规范

```ts
// ✅ 复合索引与唯一索引
export const orders = pgTable('orders', {
  id: serial('id').primaryKey(),
  userId: integer('user_id').notNull().references(() => users.id),
  status: text('status').notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
}, (table) => [
  index('orders_user_status_idx').on(table.userId, table.status),  // 复合索引
  index('orders_created_at_idx').on(table.createdAt),               // 排序/分页优化
]);

// ✅ 大小写不敏感唯一索引（email）
import { sql } from 'drizzle-orm';
import { AnyPgColumn, uniqueIndex } from 'drizzle-orm/pg-core';

function lower(col: AnyPgColumn) {
  return sql`lower(${col})`;
}

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: text('email').notNull(),
}, (table) => [
  uniqueIndex('email_unique_idx').on(lower(table.email)),
]);
```

---

## 类型推断

```ts
// ✅ 使用 $inferInsert / $inferSelect 提取表类型，避免手写重复接口
import { users } from './schema';
import type { InferInsertModel, InferSelectModel } from 'drizzle-orm';

type User = InferSelectModel<typeof users>;
// { id: number; name: string; email: string; ... }

type NewUser = InferInsertModel<typeof users>;
// { name: string; email: string; age: number; id?: number; createdAt?: Date; ... }

// ❌ 手写类型接口——与 schema 容易失步
interface User {
  id: number;
  name: string;
  email: string;
  // 字段增减时需要同步维护
}

// ✅ 导出类型供整个应用使用
export type { User, NewUser };
```

---

## 查询：SELECT

### 基础查询与过滤

```ts
import { eq, and, or, lt, gte, ne, like, isNull, inArray, desc, asc } from 'drizzle-orm';

// ✅ 类型安全的过滤条件
const user = await db.select().from(users).where(eq(users.id, 42));

// ✅ 组合条件
const results = await db.select()
  .from(users)
  .where(and(
    eq(users.verified, true),
    gte(users.age, 18),
  ));

// ✅ OR 条件
await db.select().from(users).where(
  or(eq(users.email, 'a@b.com'), eq(users.email, 'c@d.com'))
);

// ✅ 多值过滤
await db.select().from(users).where(
  inArray(users.id, [1, 2, 3])
);

// ❌ 原始字符串过滤——SQL 注入风险且无类型安全
await db.execute(sql`SELECT * FROM users WHERE id = ${userInput}`);  // 仅用于无法用构建器表达的复杂场景
```

### 部分选择与聚合

```ts
// ✅ 只选择需要的列（减少传输量）
const names = await db.select({ id: users.id, name: users.name }).from(users);

// ✅ 聚合查询
import { count, sum, avg } from 'drizzle-orm';

const stats = await db.select({
  userId: orders.userId,
  orderCount: count(orders.id),
  totalAmount: sum(orders.amount),
}).from(orders)
  .groupBy(orders.userId)
  .orderBy(desc(count(orders.id)));

// ❌ SELECT * 后在 JS 侧过滤——传输浪费
const allUsers = await db.select().from(users);
const names = allUsers.map(u => u.name);  // 不必要的全列传输
```

### JOIN

```ts
// ✅ innerJoin：只返回匹配行
const postsWithAuthor = await db.select({
  post: posts,
  author: { id: users.id, name: users.name },
}).from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .orderBy(posts.id);

// ✅ leftJoin：保留左表所有行，右表可为 null
// 注意：leftJoin 后右表列类型自动变为 T | null
const result = await db.select({
  userId: users.id,
  postId: posts.id,              // posts 列可能为 null
}).from(users)
  .leftJoin(posts, eq(users.id, posts.authorId));

// ✅ 子查询 JOIN（分页性能优化）
const sq = db.select({ id: users.id }).from(users)
  .where(eq(users.verified, true))
  .limit(10)
  .offset(20)
  .as('sq');

const data = await db.select().from(users)
  .innerJoin(sq, eq(users.id, sq.id));
```

### 分页

```ts
// ✅ limit/offset 分页（适合小数据集）
const page = await db.select().from(users)
  .orderBy(asc(users.id))
  .limit(20)
  .offset((pageNum - 1) * 20);

// ✅ 游标分页（大数据集，性能更好）
async function nextPage(cursor?: number, pageSize = 20) {
  return db.select().from(users)
    .where(cursor ? gt(users.id, cursor) : undefined)
    .orderBy(asc(users.id))
    .limit(pageSize);
}

// ❌ 海量数据直接 offset 分页——越往后越慢
await db.select().from(users).limit(20).offset(1000000);
```

### exists 子查询

```ts
import { exists } from 'drizzle-orm';

// ✅ 选择至少有一篇文章的用户
const sq = db.select({ id: sql`1` }).from(posts)
  .where(eq(posts.authorId, users.id));

const activeAuthors = await db.select().from(users).where(exists(sq));
```

---

## 关系查询（RQB）

### 定义 Relations（v2 推荐方式）

```ts
// relations.ts
import { defineRelations } from 'drizzle-orm';
import * as schema from './schema';

export const relations = defineRelations(schema, (r) => ({
  users: {
    posts: r.many.posts(),
    // 自关联（被邀请者 → 邀请人）
    invitedBy: r.one.users({
      from: r.users.invitedById,
      to: r.users.id,
    }),
  },
  posts: {
    author: r.one.users({
      from: r.posts.authorId,
      to: r.users.id,
    }),
    comments: r.many.comments(),
  },
}));

// ✅ 初始化时注入 relations
import { drizzle } from 'drizzle-orm/node-postgres';
const db = drizzle(client, { relations });
```

### 嵌套关系查询（避免 N+1）

```ts
// ❌ N+1 问题：每个用户发一次查询
const users = await db.select().from(users);
for (const user of users) {
  const posts = await db.select().from(posts).where(eq(posts.authorId, user.id));
}

// ✅ 使用 findMany + with，Drizzle 输出恰好 1 条 SQL
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: {
      with: {
        comments: true,  // 三层嵌套，仍然是 1 条 SQL
      },
    },
  },
});

// ✅ 带过滤的嵌套查询
const activeAuthors = await db.query.users.findMany({
  where: eq(users.verified, true),
  with: {
    posts: {
      where: gte(posts.createdAt, new Date('2024-01-01')),
      orderBy: desc(posts.createdAt),
      limit: 5,
    },
  },
  columns: {
    id: true,
    name: true,
    // email 不包括（安全：不暴露敏感字段）
  },
});
```

### findFirst vs findMany

```ts
// ✅ findFirst：查询单条（内部加 LIMIT 1）
const user = await db.query.users.findFirst({
  where: eq(users.id, 42),
});

// ❌ findMany 后取第一条——多余的传输与内存消耗
const [user] = await db.query.users.findMany({
  where: eq(users.id, 42),
});
```

---

## Mutations：INSERT / UPDATE / DELETE

### INSERT

```ts
// ✅ 单行插入 + returning（PostgreSQL/SQLite）
const [newUser] = await db.insert(users)
  .values({ name: 'Alice', email: 'alice@example.com', age: 25 })
  .returning();

// ✅ 批量插入
await db.insert(posts).values([
  { title: 'Post 1', content: 'Content 1', authorId: 1 },
  { title: 'Post 2', content: 'Content 2', authorId: 1 },
]);

// ✅ 仅返回必要字段
const [{ insertedId }] = await db.insert(users)
  .values({ name: 'Bob', email: 'bob@example.com', age: 30 })
  .returning({ insertedId: users.id });
```

### Upsert（onConflict）

```ts
// ✅ 冲突时更新（PostgreSQL/SQLite）
await db.insert(users)
  .values({ id: 1, name: 'John', email: 'john@example.com', age: 28 })
  .onConflictDoUpdate({
    target: users.id,         // 冲突目标列
    set: { name: 'John Updated' },
  });

// ✅ 带条件的冲突更新（setWhere）
await db.insert(products)
  .values({ id: 1, stock: 10, price: '99.99', lastUpdated: new Date() })
  .onConflictDoUpdate({
    target: products.id,
    set: {
      stock: sql`excluded.stock`,
      price: sql`excluded.price`,
      lastUpdated: sql`excluded.last_updated`,
    },
    setWhere: or(
      sql`${products.stock} != excluded.stock`,
      sql`${products.price} != excluded.price`,
    ),
  });

// ✅ 冲突时什么都不做
await db.insert(users)
  .values({ id: 1, name: 'John', email: 'john@example.com', age: 28 })
  .onConflictDoNothing();

// ✅ MySQL：onDuplicateKeyUpdate
await db.insert(users)
  .values({ id: 1, lastLogin: new Date() })
  .onDuplicateKeyUpdate({
    set: { lastLogin: sql`values(${users.lastLogin})` },
  });
```

### UPDATE

```ts
// ✅ 带 where 条件的安全更新
await db.update(users)
  .set({ name: 'Alice Updated', verified: true })
  .where(eq(users.id, 42));

// ❌ 无 where 的全表更新——危险！
await db.update(users).set({ verified: true });  // 更新所有行

// ✅ 需要全表更新时明确注释
// 注意：全表更新，确认这是预期行为
await db.update(users).set({ legacyMigrated: true });
```

### DELETE

```ts
// ✅ 带 where 条件的安全删除
await db.delete(posts).where(eq(posts.authorId, userId));

// ✅ 软删除（推荐用于有审计需求的场景）
await db.update(users).set({ deletedAt: new Date() }).where(eq(users.id, id));

// ❌ 无 where 的全表删除——极度危险
await db.delete(users);  // 清空整张表！
```

---

## 事务与 Batch API

### 事务

```ts
import { drizzle } from 'drizzle-orm/node-postgres';

// ✅ 原子操作：所有步骤成功才提交，任意失败则整体回滚
await db.transaction(async (tx) => {
  // 使用 tx 而不是 db——确保在同一事务内
  await tx.update(accounts)
    .set({ balance: sql`${accounts.balance} - 100` })
    .where(eq(accounts.userId, senderId));

  await tx.update(accounts)
    .set({ balance: sql`${accounts.balance} + 100` })
    .where(eq(accounts.userId, receiverId));
});

// ✅ 条件回滚（业务校验失败时主动回滚）
await db.transaction(async (tx) => {
  const [account] = await tx.select({ balance: accounts.balance })
    .from(accounts)
    .where(eq(accounts.userId, senderId));

  if (account.balance < 100) {
    tx.rollback();  // 显式回滚，后续代码不再执行
  }

  await tx.update(accounts)
    .set({ balance: sql`${accounts.balance} - 100` })
    .where(eq(accounts.userId, senderId));
});

// ✅ 嵌套事务（SavePoint）
await db.transaction(async (tx) => {
  await tx.update(users).set({ name: 'Dan' }).where(eq(users.id, 1));

  await tx.transaction(async (tx2) => {
    // tx2 失败只回滚到 SavePoint，不影响外层事务
    await tx2.update(accounts).set({ balance: 0 }).where(eq(accounts.userId, 1));
  });
});

// ❌ 事务内混用 db 和 tx——不保证原子性
await db.transaction(async (tx) => {
  await tx.update(accounts).set({ balance: 0 }).where(eq(accounts.userId, 1));
  await db.insert(logs).values({ action: 'reset' });  // 不在事务中！
});
```

### Batch API（LibSQL / Neon / D1）

```ts
// ✅ 批量执行减少网络往返（适用于 serverless 高延迟场景）
type BatchResponse = [
  { id: number }[],  // insert returning
  typeof users.$inferSelect[],  // findMany 结果
];

const [insertedIds, allUsers] = await db.batch<BatchResponse>([
  db.insert(users).values({ name: 'John' }).returning({ id: users.id }),
  db.query.users.findMany(),
]);

// ❌ 在 serverless 中对每个操作各自发起请求——高延迟
const id = await db.insert(users).values({ name: 'John' }).returning({ id: users.id });
const allUsers = await db.query.users.findMany();  // 两次网络往返
```

---

## 动态查询构建

```ts
import { PgSelect } from 'drizzle-orm/pg-core';

// ✅ 使用 $dynamic() + withPagination 工具函数动态组合查询
function withPagination<T extends PgSelect>(
  qb: T,
  page: number,
  pageSize = 20,
) {
  return qb.limit(pageSize).offset((page - 1) * pageSize);
}

function withUserFilters<T extends PgSelect>(
  qb: T,
  filters: { verified?: boolean; minAge?: number },
) {
  if (filters.verified !== undefined) {
    qb = qb.where(eq(users.verified, filters.verified)) as T;
  }
  if (filters.minAge !== undefined) {
    qb = qb.where(gte(users.age, filters.minAge)) as T;
  }
  return qb;
}

// 使用时
const query = db.select().from(users).orderBy(asc(users.id)).$dynamic();
const result = await withPagination(withUserFilters(query, { verified: true }), page);

// ❌ 手工拼接条件数组——容易出错且无类型安全
const conditions: SQL[] = [];
if (filters.verified) conditions.push(eq(users.verified, true));
const result = await db.select().from(users).where(and(...conditions));
// ↑ and() 接受空数组时行为因版本而异，推荐用 $dynamic 模式代替
```

---

## 迁移管理（drizzle-kit）

### drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

// ✅ 统一管理数据库连接与迁移配置
export default defineConfig({
  schema: './src/db/schema.ts',        // schema 文件路径
  out: './drizzle',                     // 迁移文件输出目录（提交到 git）
  dialect: 'postgresql',               // 'postgresql' | 'mysql' | 'sqlite'
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,  // 破坏性变更需要确认
});
```

### 迁移工作流

```bash
# ✅ 开发阶段：快速同步 schema 到本地数据库（不生成文件）
npx drizzle-kit push

# ✅ 准备部署：生成 SQL 迁移文件（提交到版本控制）
npx drizzle-kit generate

# ✅ CI/生产：执行已生成的迁移文件
npx drizzle-kit migrate

# ✅ 从已有数据库反向生成 schema（初次接入遗留数据库）
npx drizzle-kit pull
```

### 程序化运行迁移

```ts
// ✅ 在应用启动时自动执行迁移（适合 serverless / Docker 启动）
import { drizzle } from 'drizzle-orm/node-postgres';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

const db = drizzle(client);
await migrate(db, { migrationsFolder: './drizzle' });
```

### 迁移规范

```ts
// ✅ 迁移文件示例（生成后**不应**手动修改）
/*
  drizzle/0001_create_users.sql
  ---
  CREATE TABLE IF NOT EXISTS "users" (
    "id" serial PRIMARY KEY NOT NULL,
    "email" text NOT NULL,
    CONSTRAINT "users_email_unique" UNIQUE("email")
  );
*/

// ❌ 手动修改已提交的迁移文件——破坏迁移历史链
// ❌ 直接在生产 schema 上执行 drizzle-kit push——应使用 generate + migrate
```

---

## Review Checklists

### Schema 定义

- [ ] 所有必填列都有 `.notNull()`
- [ ] 自动更新时间戳用 `$onUpdate(() => new Date())`，不依赖手动传入
- [ ] 外键列有 `onDelete` / `onUpdate` 级联策略声明
- [ ] 查询频率高的过滤/排序列有对应索引
- [ ] email 等大小写不敏感唯一键使用 `uniqueIndex(...).on(lower(...))`
- [ ] 未使用 `any` 或原始 SQL 字符串绕过类型系统

### 类型推断

- [ ] 使用 `InferSelectModel<typeof table>` / `InferInsertModel<typeof table>` 而非手写接口
- [ ] 导出类型供应用层复用（避免重复定义）

### SELECT 查询

- [ ] 只选取必要列（不 `SELECT *` 后再 JS 过滤）
- [ ] 过滤条件使用 Drizzle 操作符（`eq`、`and`、`or`...），不手写 SQL 字符串
- [ ] leftJoin 后的右表列类型已手动标注 `sql<T | null>`（避免类型 unknown）
- [ ] 大数据集使用游标分页而非高 offset 翻页

### 关系查询（RQB）

- [ ] 批量加载关联数据使用 `findMany({ with: {...} })`，避免循环中逐条查询（N+1）
- [ ] 只查单条记录使用 `findFirst`，不用 `findMany()[0]`
- [ ] `with` 嵌套中对子集合使用 `limit`/`where` 防止过度加载
- [ ] 敏感字段通过 `columns` 选项显式排除

### Mutations

- [ ] UPDATE / DELETE 操作都带有 `where` 条件（无条件更新/删除需要明确注释）
- [ ] 使用 `returning()` 获取插入/更新后的数据，不再发一次额外查询
- [ ] Upsert 使用 `onConflictDoUpdate` / `onDuplicateKeyUpdate`，不在代码层先查后写
- [ ] `$onUpdate` 自动更新的列不需要在每次 `set()` 时手动传入

### 事务

- [ ] 多步骤原子操作都包在 `db.transaction(async (tx) => {...})` 中
- [ ] 事务内所有操作使用 `tx` 而非 `db`
- [ ] 业务校验失败时调用 `tx.rollback()` 显式回滚
- [ ] Serverless 场景（LibSQL / Neon / D1）考虑用 `db.batch()` 代替多次独立请求

### 动态查询

- [ ] 多条件动态拼接使用 `$dynamic()` + 工具函数，避免空 `and()` 语义问题

### 迁移管理

- [ ] 迁移文件提交到版本控制（`drizzle/` 目录）
- [ ] 生产环境使用 `generate` + `migrate`，不使用 `push`
- [ ] 已提交的迁移文件不手动修改
- [ ] `drizzle.config.ts` 中 `DATABASE_URL` 等凭据从环境变量读取，不硬编码
