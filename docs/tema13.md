# Tema 13. Deploy de Next.js en Vercel

Con la base publicada en Supabase (tema 12), se publica la aplicación en **Vercel**, la plataforma creada por el equipo de Next.js. Se conectan las variables de entorno, Prisma y Clerk en producción. Cierra el bloque de Next.js; el tema 14 empieza con Astro.

## Antes de desplegar

Verificar en local:

```bash
npm run lint
npm run test:run
npm run build
```

Si `npm run build` falla en local, también fallará en Vercel.

## Prisma en el build

Vercel cachea las dependencias, por lo que el cliente generado de Prisma puede quedar desactualizado. Se regenera en cada instalación:

```json
// package.json
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

Como el cliente se genera en `src/generated` (ignorado por Git), este paso es **obligatorio**.

## Importar el proyecto

1. Subir el repositorio a GitHub (rama `main`).
2. En `vercel.com`: **Add New → Project** e importar el repositorio.
3. Vercel detecta Next.js y configura el build automáticamente.

## Variables de entorno

En **Settings → Environment Variables**, cargar las mismas variables que en `.env.local`, con los valores de producción:

| Variable | Valor |
|----------|-------|
| `DATABASE_URL` | URL del pooler de Supabase (puerto 6543) |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Clave pública de Clerk |
| `CLERK_SECRET_KEY` | Clave secreta de Clerk |
| `CLERK_WEBHOOK_SIGNING_SECRET` | Secreto del webhook de producción |
| `ANTHROPIC_API_KEY` | Clave del servicio de IA |
| `AI_MODEL` | Modelo a utilizar |

`DIRECT_URL` solo se necesita para migrar desde tu equipo, no en Vercel.

Cada variable se asigna a uno o más entornos: *Production*, *Preview* y *Development*. Los cambios en variables **solo se aplican a los despliegues nuevos**: hay que volver a desplegar.

## Imágenes remotas

Todo dominio de imágenes externas debe estar autorizado en `next.config.ts` (tema 5). Si una imagen falla en producción, es lo primero a revisar.

## Clerk en producción

Clerk separa los entornos de desarrollo y producción, con claves distintas.

1. En el dashboard de Clerk, crear la instancia de **producción**.
2. Configurar el dominio de la aplicación.
3. Reemplazar las claves `pk_test_` / `sk_test_` por las `pk_live_` / `sk_live_` en Vercel.
4. Crear el **webhook de producción** apuntando a `https://tu-dominio.vercel.app/api/webhooks/clerk` y copiar su secreto a `CLERK_WEBHOOK_SIGNING_SECRET`.

## Desplegar

Hacer clic en **Deploy**. Cada `git push` posterior genera un despliegue automático:

| Rama | Resultado |
|------|-----------|
| `main` | Despliegue de **producción** |
| Cualquier otra rama / Pull Request | Despliegue de **preview** con URL propia |

Los *previews* permiten revisar una funcionalidad antes de integrarla, lo que encaja con el flujo de Gitflow del curso anterior.

## Región de las funciones

Para minimizar la latencia con la base, las funciones deben ejecutarse cerca de ella. En **Settings → Functions**, elegir la región más próxima a la de Supabase.

## Verificar el despliegue

- [ ] El catálogo carga productos desde Supabase.
- [ ] Las imágenes se ven correctamente.
- [ ] El registro e inicio de sesión funcionan.
- [ ] Un usuario nuevo aparece en la tabla `User` (webhook).
- [ ] `/admin` redirige a quien no es administrador.
- [ ] El asistente de IA responde.

## Diagnosticar errores

- **Logs de build:** pestaña *Deployments → seleccionar despliegue → Build Logs*.
- **Logs de ejecución:** pestaña *Logs*, filtrando por nivel *Error*.

| Síntoma | Causa probable |
|---------|----------------|
| `Cannot find module '@/generated/prisma/client'` | Falta `postinstall: prisma generate` |
| Error 500 en páginas con datos | `DATABASE_URL` ausente o incorrecta |
| Redirección infinita al iniciar sesión | Claves de Clerk de otro entorno |
| El webhook no llega | URL o secreto de producción incorrectos |
| Imágenes sin cargar | Dominio no autorizado en `next.config.ts` |

## Dominio propio

En **Settings → Domains** se agrega el dominio. Vercel indica los registros DNS a configurar. Luego, actualizar el dominio en Clerk y en el webhook.
