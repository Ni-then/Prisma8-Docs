# 🗄️ Prisma 8 + Turborepo Setup Guide

This guide explains how to set up **Prisma 8** inside a **Turborepo monorepo** using a shared database package.

The final structure will allow multiple applications to use the same Prisma database client.

---

## 📁 Step 1 — Create the Prisma Package

Go inside the `packages` directory:

```bash
cd packages
mkdir prisma-system
cd prisma-system
```

Create a `package.json` file:

```json
{
  "name": "@repo/prisma-system",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "contract:emit": "prisma contract emit",
    "db:init": "prisma db init",
    "db:update": "prisma db update",
    "db:sign": "prisma db sign",
    "db:studio": "prisma studio",
    "db:migrate": "prisma migration plan && prisma db migrate --yes"
  }
}
```

---

# 📦 Step 2 — Install Dependencies

Inside:

```text
packages/prisma-system
```

Install development dependencies:

```bash
npm install prisma typescript tsx @types/node --save-dev
```

Install runtime dependencies:

```bash
npm install @prisma/orm-postgres pg dotenv
```

### Dependency Overview

| Package                | Purpose                                          |
| ---------------------- | ------------------------------------------------ |
| `prisma`               | Prisma CLI, migrations, contract emit and studio |
| `@prisma/orm-postgres` | Prisma 8 PostgreSQL runtime adapter              |
| `pg`                   | PostgreSQL driver                                |
| `dotenv`               | Loads environment variables                      |
| `typescript`           | TypeScript support                               |
| `tsx`                  | TypeScript execution                             |
| `@types/node`          | Node.js TypeScript types                         |

---

# ⚙️ Step 3 — Create `prisma.config.ts`

Create:

```text
packages/prisma-system/prisma.config.ts
```

```ts
import "dotenv/config";

import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "src/prisma/contract.prisma",

  datasource: {
    url: env("DATABASE_URL"),
  },
});
```

This configuration tells Prisma:

* Where your contract/schema is located.
* Where to find your database connection URL.

---

# 🚀 Step 4 — Initialize Prisma ORM

Run:

```bash
npx prisma@latest orm init
```

This automatically creates the Prisma 8 structure, including:

* `src/prisma/contract.prisma`
* `src/prisma/db.ts`
* Prisma contract files
* ORM configuration

Your package should now start looking similar to this:

```text
packages/prisma-system/
│
├── prisma.config.ts
└── src/
    └── prisma/
        ├── contract.prisma
        └── db.ts
```

---

# 🔐 Step 5 — Configure Environment Variables

Create a `.env` file:

```text
packages/prisma-system/.env
```

Add your PostgreSQL connection string:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/yourdb"
```

> [!IMPORTANT]
> Never commit your real `.env` file to GitHub.

If you are using a managed Prisma/PostgreSQL database, use the connection string provided by your database provider.

---

# 🧩 Step 6 — Define Your Database Contract

Open:

```text
src/prisma/contract.prisma
```

Example:

```prisma
datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)

  authorId Int
  author   User @relation(fields: [authorId], references: [id])
}
```

Here we have two models:

```text
User
  │
  └───────< Post
```

A single user can have multiple posts.

---

# 🏗️ Step 7 — Emit the Contract

Run:

```bash
npm run contract:emit
```

This generates the Prisma contract artifacts:

```text
contract.json
contract.d.ts
```

These generated files are used by the Prisma runtime and query system.

---

# 🗃️ Step 8 — Sign the Database

Run:

```bash
npm run db:sign
```

This records that your database is aligned with the current Prisma contract.

---

# 🔌 Step 9 — Export the Database Client

The Prisma initialization creates:

```text
src/prisma/db.ts
```

It will roughly look like this:

```ts
import "dotenv/config";

import postgres from "@prisma/orm-postgres/runtime";

import contractJson from "./contract.json" with {
  type: "json",
};

const db = postgres({
  url: process.env.DATABASE_URL!,
  contractJson,
});

export default db;
```

Now create:

```text
src/index.ts
```

```ts
export { default as db } from "./prisma/db";

export * from "./prisma/contract";
```

Now other applications in the monorepo can import the shared database client.

---

# ⚡ Step 10 — Configure Turborepo

Add the Prisma tasks to your root `turbo.json`.

```json
{
  "globalEnv": ["DATABASE_URL"],

  "tasks": {
    "dev": {
      "cache": false,
      "persistent": true,
      "dependsOn": ["^contract:emit"]
    },

    "build": {
      "dependsOn": ["^contract:emit"],
      "outputs": [".next/**", "dist/**"]
    },

    "contract:emit": {
      "cache": false
    },

    "db:init": {
      "cache": false
    },

    "db:update": {
      "cache": false
    },

    "db:sign": {
      "cache": false
    },

    "db:migrate": {
      "cache": false
    },

    "db:studio": {
      "cache": false
    }
  }
}
```

---

# 🎯 Add Root-Level Database Scripts

In the root `package.json`:

```json
{
  "scripts": {
    "db:init": "turbo run db:init",

    "db:update": "turbo run db:update",

    "db:sign": "turbo run db:sign",

    "db:migrate": "turbo run db:migrate",

    "db:studio": "turbo run db:studio",

    "contract:emit": "turbo run contract:emit"
  }
}
```

Now you can run database commands directly from the monorepo root.

For example:

```bash
npm run db:init
```

or:

```bash
npm run contract:emit
```

---

# 🧑‍💻 Step 11 — Use Prisma Inside an Application

For example, inside:

```text
apps/web
```

Add the shared database package:

```json
{
  "dependencies": {
    "@repo/prisma-system": "workspace:*"
  }
}
```

Now use it anywhere inside your application.

Example:

```text
apps/web/app/api/users/route.ts
```

```ts
import { db } from "@repo/prisma-system";

export async function GET() {
  const users = await db.orm.public.User.findMany();

  return Response.json(users);
}
```

### Prisma 8 Query Style

Prisma 8 uses namespace-qualified model access.

```ts
db.orm.public.User.findMany();
```

Instead of the older Prisma Client style:

```ts
prisma.user.findMany();
```

---

# 🔄 Prisma 7 vs Prisma 8

| Feature               | Prisma 7                 | Prisma 8                        |
| --------------------- | ------------------------ | ------------------------------- |
| Initialization        | `prisma init`            | `prisma orm init`               |
| Schema                | `schema.prisma`          | `contract.prisma`               |
| Generate Client       | `prisma generate`        | `prisma contract emit`          |
| Client Package        | `@prisma/client`         | `@prisma/orm-postgres`          |
| Development Migration | `prisma migrate dev`     | `prisma db update`              |
| Database Alignment    | —                        | `prisma db sign`                |
| Query Style           | `prisma.user.findMany()` | `db.orm.public.User.findMany()` |

---

# 📂 Final Project Structure

```text
packages/
└── prisma-system/
    │
    ├── .env
    ├── package.json
    ├── prisma.config.ts
    ├── tsconfig.json
    │
    └── src/
        ├── index.ts
        │
        └── prisma/
            ├── contract.prisma
            ├── contract.json
            ├── contract.d.ts
            └── db.ts
```

---

# 🚫 Update `.gitignore`

Add the generated Prisma contract files:

```gitignore
src/prisma/contract.json
src/prisma/contract.d.ts
```

Also make sure your environment file is ignored:

```gitignore
.env
```

---

# 🚀 Quick Start

From the root of your monorepo:

```bash
npm run db:init
```

Then emit the Prisma contract:

```bash
npm run contract:emit
```

Finally, sign the database:

```bash
npm run db:sign
```

---

## 🎉 Done!

Your Prisma 8 setup is now ready inside a **Turborepo monorepo**.

You now have:

* 🗄️ Shared PostgreSQL database package
* 📦 Reusable Prisma package
* 🔄 Turborepo task integration
* 🧩 Shared database client across applications
* 🔐 Environment-based database configuration
* ⚡ Prisma 8 contract-based workflow

```text
Apps
 │
 ├── apps/web
 │
 ├── apps/api
 │
 └── apps/worker
       │
       ▼
┌──────────────────────┐
│ @repo/prisma-system  │
│                      │
│   Prisma 8 ORM       │
│   PostgreSQL         │
└──────────┬───────────┘
           │
           ▼
      PostgreSQL DB
```

**Happy Building! 🚀**
