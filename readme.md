Step 1: Package create karo
cd packages/
mkdir prisma-system
cd prisma-system

package.json banao:

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


prisma8 turbo repo install guide

View all
Step 2: Dependencies install karo
cd packages/prisma-system

# Dev dependencies
npm install prisma typescript tsx @types/node --save-dev

# Runtime dependencies
npm install @prisma/orm-postgres pg dotenv

Package	Kaam
prisma	CLI (migrations, contract emit, studio)
@prisma/orm-postgres	Prisma 8 ka Postgres runtime (adapter)
pg	PostgreSQL driver
dotenv	Env vars load karna

Step 3: prisma.config.ts banao
// packages/prisma-system/prisma.config.ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "src/prisma/contract.prisma",
  datasource: {
    url: env("DATABASE_URL"),
  },
});

Step 4: prisma orm init chalao
npx prisma@latest orm init

Yeh automatically:

src/prisma/contract.prisma scaffold karta hai (starter contract)
src/prisma/db.ts banata hai (client file)
Contract emit karta hai (contract.json + contract.d.ts)
Agent skills register karta hai 
Step 5: .env file setup karo
packages/prisma-system/.env mein:

DATABASE_URL="postgresql://postgres:postgres@localhost:5432/yourdb"

Agar Prisma Postgres (managed) use kar rahe ho toh npx create-db se connection string milayega. 

Step 6: Contract (schema) edit karo
src/prisma/contract.prisma mein apne models define karo:

datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
}

Prisma 8 mein schema PSL ya TypeScript — dono mein likh sakte ho. 

Step 7: Contract emit karo
npm run contract:emit

Yeh contract.json + contract.d.ts generate karta hai — yehi hai jo runtime aur queries use karti hain.

Step 8: Database sign karo
npm run db:sign

Yeh record karta hai ki live DB contract ke saath aligned hai.

Step 9: Client export karo
src/prisma/db.ts (jo orm init ne banaya hoga) roughly aisa dikhega:

import "dotenv/config";
import postgres from "@prisma/orm-postgres/runtime";
import contractJson from "./contract.json" with { type: "json" };

const db = postgres({
  url: process.env.DATABASE_URL!,
  contractJson,
});

export default db;

Ab src/index.ts se export karo:

// packages/prisma-system/src/index.ts
export { default as db } from "./prisma/db";
export * from "./prisma/contract";

Step 10: turbo.json mein tasks wire karo
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
    "contract:emit": { "cache": false },
    "db:init":     { "cache": false },
    "db:update":   { "cache": false },
    "db:sign":     { "cache": false },
    "db:migrate":  { "cache": false },
    "db:studio":   { "cache": false }
  }
}

Root package.json mein proxy scripts:

{
  "scripts": {
    "db:init":      "turbo run db:init",
    "db:update":    "turbo run db:update",
    "db:sign":      "turbo run db:sign",
    "db:migrate":   "turbo run db:migrate",
    "db:studio":    "turbo run db:studio",
    "contract:emit":"turbo run contract:emit"
  }
}

Step 11: App mein use karo
Kisi bhi app (e.g. apps/web) mein:

{
  "dependencies": {
    "@repo/prisma-system": "workspace:*"
  }
}

// apps/web/app/api/users/route.ts
import { db } from "@repo/prisma-system";

export async function GET() {
  const users = await db.orm.public.User.findMany();
  return Response.json(users);
}

Note: Prisma 8 mein model access namespace-qualified hota hai: db.orm.public.User (schema ke saath). 

Quick Reference: Prisma 7 vs 8
Prisma 7	Prisma 8
Init	prisma init	prisma orm init
Schema file	schema.prisma	contract.prisma
Generate	prisma generate	prisma contract emit
Client package	@prisma/client	@prisma/orm-postgres
Migrate	prisma migrate dev	prisma db update / prisma db sign
Query style	prisma.user.findMany()	db.orm.public.User.findMany()

Final Structure
packages/prisma-system/
├── prisma.config.ts
├── .env
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts              ← export db + types
    └── prisma/
        ├── contract.prisma   ← tumhara schema
        ├── contract.json     ← emitted (gitignore karo)
        ├── contract.d.ts     ← emitted (gitignore karo)
        └── db.ts             ← Prisma 8 client

.gitignore mein add karo:

src/prisma/contract.json
src/prisma/contract.d.ts

Bas, done! Ab npm run db:init → npm run contract:emit → npm run db:sign — aur tumhara Prisma 8 ready hai. 🚀 


