# Tema 9. Autenticación con Clerk

El panel de administración del tema 8 (parte 2) está abierto a cualquiera. Este tema incorpora **Clerk**, un servicio que gestiona registro, inicio de sesión y sesiones, para proteger las rutas de la tienda. En la parte 2 de este tema se sincronizan esos usuarios con la base de datos propia.

## Configuración en Clerk

1. Crear una cuenta en `clerk.com` y una **aplicación** nueva.
2. Elegir los métodos de acceso (email, Google, etc.).
3. Copiar las claves de la sección *API Keys*.

```bash
# .env.local
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

## Instalación

```bash
npm install @clerk/nextjs
```

## Middleware: `proxy.ts`

En Next.js 16 el archivo `middleware.ts` pasó a llamarse **`proxy.ts`**. En Next.js 15 o anterior se llama `middleware.ts`. Va en la raíz de `src/`.

```ts
// src/proxy.ts
import { clerkMiddleware } from "@clerk/nextjs/server";

export default clerkMiddleware();

export const config = {
  matcher: [
    // Omitir archivos internos y estáticos de Next.js
    "/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)",
    // Ejecutar siempre en rutas de API
    "/(api|trpc)(.*)",
  ],
};
```

Con esta configuración mínima, Clerk conoce el estado de autenticación en toda la aplicación, pero **todas las rutas siguen siendo públicas**.

## Proveedor y componentes de interfaz

```tsx
// src/app/layout.tsx
import { ClerkProvider } from "@clerk/nextjs";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="es">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  );
}
```

```tsx
// src/components/Header.tsx
import { SignedIn, SignedOut, SignInButton, SignUpButton, UserButton } from "@clerk/nextjs";

export default function Header() {
  return (
    <header className="flex justify-between p-4">
      <span>TechShop</span>
      <SignedOut>
        <SignInButton />
        <SignUpButton />
      </SignedOut>
      <SignedIn>
        <UserButton />
      </SignedIn>
    </header>
  );
}
```

## Proteger rutas

```ts
// src/proxy.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const esRutaProtegida = createRouteMatcher(["/admin(.*)", "/pedidos(.*)"]);

export default clerkMiddleware(async (auth, req) => {
  if (esRutaProtegida(req)) await auth.protect();
});
```

`auth.protect()` redirige al inicio de sesión si el usuario no está autenticado.

## Leer el usuario en el servidor

```tsx
import { auth, currentUser } from "@clerk/nextjs/server";

export default async function PedidosPage() {
  const { userId } = await auth();          // liviano: solo el id
  const usuario = await currentUser();      // objeto completo (hace una petición a Clerk)

  return <h1>Pedidos de {usuario?.firstName}</h1>;
}
```

## Leer el usuario en el cliente

```tsx
"use client";
import { useUser } from "@clerk/nextjs";

export default function Saludo() {
  const { isLoaded, isSignedIn, user } = useUser();
  if (!isLoaded) return null;
  return isSignedIn ? <p>Hola, {user.firstName}</p> : <p>Iniciá sesión</p>;
}
```

## Proteger Server Actions y Route Handlers

El `proxy` protege las **páginas**, pero las Server Actions y las rutas de API son endpoints independientes: cada uno debe verificar la sesión.

```ts
// src/app/(admin)/admin/productos/actions.ts
"use server";
import { auth } from "@clerk/nextjs/server";

export async function eliminarProducto(id: number) {
  const { userId } = await auth();
  if (!userId) throw new Error("No autorizado");
  // ...
}
```

```ts
// src/app/api/chat/route.ts
import { auth } from "@clerk/nextjs/server";

export async function POST(request: Request) {
  const { userId } = await auth();
  if (!userId) return Response.json({ error: "No autorizado" }, { status: 401 });
  // ...
}
```

Con esto se limita el uso del asistente de IA (tema 7) a usuarios registrados.

## Autenticación vs. autorización

| Concepto | Pregunta | Ejemplo |
|----------|----------|---------|
| Autenticación | ¿Quién sos? | Iniciar sesión |
| Autorización | ¿Qué podés hacer? | Solo administradores editan productos |

Clerk resuelve la autenticación. Los **roles** (autorización) se guardan en la base de datos propia, que se prepara en la parte 2.
