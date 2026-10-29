# Tema 8. Configuración de Prisma

Hasta ahora los productos venían de una API externa. Este tema incorpora una base de datos PostgreSQL propia, manejada con **Prisma**, un ORM que traduce el modelo de datos a tablas y expone un cliente con tipos. Los datos de este tema los consume el CRUD del tema 8 (parte 2) y se publican en Supabase en el tema 12.

Esta guía corresponde a **Prisma 7**, que cambió la configuración respecto de versiones anteriores: la URL de conexión va en `prisma.config.ts`, el generador es `prisma-client` con ruta de salida, y el cliente requiere un *driver adapter*.

## PostgreSQL local con Docker

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: tienda
      POSTGRES_PASSWORD: tienda
      POSTGRES_DB: tienda
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d
```

## Instalación

```bash
npm install @prisma/client @prisma/adapter-pg pg dotenv
npm install -D prisma tsx @types/pg
npx prisma init --datasource-provider postgresql --output ../src/generated/prisma
```

El comando crea `prisma/schema.prisma`, `prisma.config.ts` y agrega `DATABASE_URL` al `.env`. Dejar la URL así:

```bash
# .env
DATABASE_URL="postgresql://tienda:tienda@localhost:5432/tienda"
```

Agregar al `.gitignore`: `/src/generated`.

## Configuración de la CLI

```ts
// prisma.config.ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
    seed: "tsx prisma/seed.ts",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
});
```

La CLI no carga `.env` automáticamente, por eso se importa `dotenv/config`.

## Modelo de datos

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model Category {
  id       Int       @id @default(autoincrement())
  name     String    @unique
  products Product[]
}

model Product {
  id          Int      @id @default(autoincrement())
  name        String
  description String?
  price       Decimal  @db.Decimal(10, 2)
  stock       Int      @default(0)
  image       String?
  category    Category @relation(fields: [categoryId], references: [id])
  categoryId  Int
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

Conceptos:

- `@id`, `@default`, `@unique`: restricciones de la tabla.
- `Category` y `Product` se relacionan uno a muchos: la clave foránea es `categoryId`.
- `?` indica que el campo es opcional (acepta `null`).

## Migraciones

```bash
npx prisma migrate dev --name init
```

Crea la carpeta `prisma/migrations/` con el SQL, lo aplica a la base y genera el cliente. Esa carpeta **se versiona en Git**.

| Comando | Uso |
|---------|-----|
| `prisma migrate dev` | Desarrollo: crea y aplica migraciones |
| `prisma migrate deploy` | Producción: aplica las existentes (tema 12) |
| `prisma generate` | Regenera el cliente tras cambiar el schema |
| `prisma studio` | Interfaz visual para ver y editar datos |
| `prisma db seed` | Ejecuta el script de datos iniciales |

## Cliente de Prisma

Una única instancia compartida evita agotar las conexiones por las recargas de desarrollo.

```ts
// src/lib/prisma.ts
import "server-only";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "@/generated/prisma/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

function crearCliente() {
  const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
  return new PrismaClient({ adapter });
}

export const prisma = globalForPrisma.prisma ?? crearCliente();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

## Datos iniciales (seed)

```ts
// prisma/seed.ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../src/generated/prisma/client";

const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }),
});

async function main() {
  const perifericos = await prisma.category.upsert({
    where: { name: "Periféricos" },
    update: {},
    create: { name: "Periféricos" },
  });

  await prisma.product.createMany({
    data: [
      { name: "Teclado mecánico", price: 89.9, stock: 12, categoryId: perifericos.id },
      { name: "Mouse inalámbrico", price: 35, stock: 30, categoryId: perifericos.id },
    ],
  });
}

main().finally(() => prisma.$disconnect());
```

```bash
npx prisma db seed
```

## Primera consulta

```tsx
// src/app/(tienda)/productos/page.tsx
import { prisma } from "@/lib/prisma";

export default async function ProductosPage() {
  const productos = await prisma.product.findMany({
    include: { category: true },
    orderBy: { createdAt: "desc" },
  });

  return (
    <ul>
      {productos.map((p) => (
        <li key={p.id}>{p.name} — ${Number(p.price)}</li>
      ))}
    </ul>
  );
}
```

`Decimal` devuelve un objeto especial que **no se puede pasar a un componente de cliente**. Se convierte con `Number(p.price)` antes de enviarlo.
