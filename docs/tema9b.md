# Tema 9 (parte 2). Usuarios de Clerk en la base de datos local

Clerk guarda a los usuarios en sus servidores, pero la tienda necesita relacionarlos con sus propios datos: pedidos, direcciones y **roles**. Este tema crea una tabla `User` local y la mantiene sincronizada con Clerk mediante *webhooks*. Continúa el tema 9 y habilita la autorización por roles en el panel de administración.

## Modelo local

```prisma
enum Role {
  USER
  ADMIN
}

model User {
  id        Int      @id @default(autoincrement())
  clerkId   String   @unique
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
}
```

```bash
npx prisma migrate dev --name add-user
```

`clerkId` es el vínculo entre ambos sistemas: el identificador que Clerk asigna al usuario.

## Estrategia 1: webhook

Clerk avisa a la aplicación cuando un usuario se crea, actualiza o elimina, mediante una petición HTTP.

### Endpoint

```ts
// src/app/api/webhooks/clerk/route.ts
import { verifyWebhook } from "@clerk/nextjs/webhooks";
import type { NextRequest } from "next/server";
import { prisma } from "@/lib/prisma";

export async function POST(req: NextRequest) {
  try {
    const evt = await verifyWebhook(req); // valida la firma del mensaje

    if (evt.type === "user.created" || evt.type === "user.updated") {
      const { id, email_addresses, first_name, last_name } = evt.data;
      const email = email_addresses[0]?.email_address;
      const name = [first_name, last_name].filter(Boolean).join(" ") || null;

      await prisma.user.upsert({
        where: { clerkId: id },
        update: { email, name },
        create: { clerkId: id, email, name },
      });
    }

    if (evt.type === "user.deleted" && evt.data.id) {
      await prisma.user.deleteMany({ where: { clerkId: evt.data.id } });
    }

    return new Response("ok", { status: 200 });
  } catch {
    return new Response("Firma inválida", { status: 400 });
  }
}
```

`verifyWebhook` comprueba la firma: sin esa verificación, cualquiera podría enviar eventos falsos al endpoint.

### Configurar en Clerk

1. En el dashboard: **Webhooks → Add Endpoint**.
2. URL: `https://tu-dominio/api/webhooks/clerk`.
3. Suscribirse a `user.created`, `user.updated` y `user.deleted`.
4. Copiar el *Signing Secret* a `.env.local`:

```bash
CLERK_WEBHOOK_SIGNING_SECRET=whsec_...
```

La ruta del webhook debe ser pública: no incluirla en las rutas protegidas del `proxy`.

### Probar en local

Clerk necesita una URL pública. Se expone el puerto local con un túnel:

```bash
npx ngrok http 3000
```

Usar la URL de ngrok (`https://xxxx.ngrok-free.app/api/webhooks/clerk`) como endpoint en el entorno de desarrollo de Clerk.

## Estrategia 2: sincronización bajo demanda

Alternativa sin webhooks: crear el registro la primera vez que el usuario necesita algo de la base.

```ts
// src/lib/usuario.ts
import "server-only";
import { auth, currentUser } from "@clerk/nextjs/server";
import { prisma } from "@/lib/prisma";

export async function obtenerUsuarioActual() {
  const { userId } = await auth();
  if (!userId) return null;

  const existente = await prisma.user.findUnique({ where: { clerkId: userId } });
  if (existente) return existente;

  const u = await currentUser();
  if (!u) return null;

  return prisma.user.create({
    data: {
      clerkId: userId,
      email: u.emailAddresses[0].emailAddress,
      name: u.fullName,
    },
  });
}
```

| | Webhook | Bajo demanda |
|---|---------|--------------|
| Datos siempre actualizados | Sí | Solo al primer uso |
| Requiere URL pública | Sí | No |
| Complejidad | Mayor | Menor |
| Usuarios eliminados | Se reflejan | No se reflejan |

Para la tienda se recomienda el webhook, con la función bajo demanda como respaldo.

## Autorización por rol

```ts
// src/lib/usuario.ts
export async function requerirAdmin() {
  const usuario = await obtenerUsuarioActual();
  if (!usuario || usuario.role !== "ADMIN") {
    throw new Error("No autorizado");
  }
  return usuario;
}
```

Se usa al comienzo de cada Server Action del panel:

```ts
"use server";
import { requerirAdmin } from "@/lib/usuario";

export async function crearProducto(formData: FormData) {
  await requerirAdmin();
  // ...escritura en la base
}
```

También se protege la página:

```tsx
// src/app/(admin)/layout.tsx
import { redirect } from "next/navigation";
import { obtenerUsuarioActual } from "@/lib/usuario";

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const usuario = await obtenerUsuarioActual();
  if (usuario?.role !== "ADMIN") redirect("/");
  return <>{children}</>;
}
```

## Asignar el primer administrador

Se promueve manualmente desde Prisma Studio o por SQL:

```bash
npx prisma studio   # editar el campo role del usuario a ADMIN
```

## Relacionar con otros datos

Con el usuario local, se pueden vincular pedidos y direcciones:

```prisma
model Order {
  id        Int      @id @default(autoincrement())
  user      User     @relation(fields: [userId], references: [id])
  userId    Int
  total     Decimal  @db.Decimal(10, 2)
  createdAt DateTime @default(now())
}
```

Y agregar `orders Order[]` al modelo `User`.
