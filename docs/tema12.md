# Tema 12. Deploy de Prisma en Supabase

La base local del tema 8 solo existe en la computadora del desarrollador. Este tema publica la base en **Supabase**, que ofrece PostgreSQL administrado, y conecta Prisma con ella. Es el paso previo al despliegue de la aplicación en Vercel (tema 13).

## Crear el proyecto en Supabase

1. Crear una cuenta en `supabase.com` y un proyecto nuevo.
2. Elegir una **región cercana a la de Vercel** (por ejemplo, `us-east`) para reducir la latencia.
3. Definir y **guardar** la contraseña de la base de datos.

## Tipos de conexión

Supabase ofrece varias cadenas de conexión. En el botón **Connect** del proyecto aparecen todas.

| Conexión | Puerto | Uso |
|----------|--------|-----|
| Directa | 5432 | Migraciones desde tu equipo |
| Pooler en modo sesión | 5432 | Servidores persistentes |
| Pooler en modo transacción | 6543 | **Aplicaciones serverless** (Vercel) |

Vercel ejecuta la aplicación como funciones efímeras. Cada una abre su propia conexión, y sin un *pooler* se agota el límite de conexiones de PostgreSQL.

## Variables de entorno

```bash
# .env
# Para la aplicación (pooler, modo transacción)
DATABASE_URL="postgresql://postgres.PROYECTO:CONTRASEÑA@aws-0-REGION.pooler.supabase.com:6543/postgres"

# Para migraciones (conexión directa)
DIRECT_URL="postgresql://postgres:CONTRASEÑA@db.PROYECTO.supabase.co:5432/postgres"
```

Copiar las URLs exactas desde el panel **Connect** de Supabase: el formato del host puede variar. Si la contraseña tiene caracteres especiales (`@`, `#`, `/`), deben codificarse en la URL.

## Configurar Prisma

La CLI usa la conexión directa para migrar; la aplicación usa la del pooler.

```ts
// prisma.config.ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations", seed: "tsx prisma/seed.ts" },
  datasource: {
    url: process.env.DIRECT_URL ?? process.env.DATABASE_URL!,
  },
});
```

El cliente de `src/lib/prisma.ts` sigue usando `DATABASE_URL`, es decir, el pooler.

## Aplicar las migraciones

```bash
npx prisma migrate deploy
npx prisma db seed
```

`migrate deploy` aplica las migraciones ya existentes sin crear nuevas ni borrar datos: es el comando para entornos remotos. **No usar `migrate dev` contra una base de producción.**

## Verificar

```bash
npx prisma studio
```

También se pueden ver las tablas en Supabase, en **Table Editor**. Deberían aparecer `Category`, `Product`, `User` y `_prisma_migrations`.

## Seguridad: Row Level Security

Supabase expone automáticamente una API REST sobre las tablas del esquema `public`. Como la tienda accede a la base **solo mediante Prisma desde el servidor**, esa API no se necesita y conviene bloquearla:

1. En **Table Editor**, abrir cada tabla.
2. Activar **Enable RLS** (Row Level Security).
3. No crear políticas: así la API pública no puede leer ni escribir.

Prisma se conecta con el usuario `postgres`, que no está sujeto a RLS, por lo que la aplicación sigue funcionando.

## Flujo de cambios de esquema

```text
1. Modificar schema.prisma
2. npx prisma migrate dev --name descripcion   (base local)
3. Commit de la carpeta prisma/migrations
4. npx prisma migrate deploy                    (base de Supabase)
```

## Problemas frecuentes

| Síntoma | Causa probable |
|---------|----------------|
| `Can't reach database server` | Región, firewall o URL incorrecta |
| `too many connections` | La aplicación usa la conexión directa en lugar del pooler |
| Error de autenticación | Contraseña con caracteres especiales sin codificar |
| Errores de SSL | Agregar `?sslmode=require` a la URL |
| Las migraciones se cuelgan | Ejecutarlas con `DIRECT_URL`, no con el pooler |
