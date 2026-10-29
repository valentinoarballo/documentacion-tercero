# Tema 5. Componentes de servidor

Esta es la diferencia central de Next.js: por defecto, **todos los componentes se ejecutan en el servidor**. Pueden ser `async`, leer datos directamente y enviar al navegador solo HTML. Este tema carga el catálogo de la tienda desde una API sin `useEffect`. En el tema 6 se agrega la interactividad que los componentes de servidor no pueden tener.

## Qué es un componente de servidor

Un componente de servidor (RSC) se renderiza en el servidor y **no envía su JavaScript al navegador**.

| Puede | No puede |
|-------|----------|
| Ser `async` y usar `await` | Usar `useState`, `useEffect` |
| Leer bases de datos y variables secretas | Usar eventos (`onClick`) |
| Reducir el JavaScript enviado | Usar APIs del navegador (`window`, `localStorage`) |

## Obtener datos

```tsx
// src/app/(tienda)/productos/page.tsx
import ProductCard from "@/components/ProductCard";

type Producto = { id: number; title: string; price: number; thumbnail: string };

async function obtenerProductos(): Promise<Producto[]> {
  const res = await fetch("https://dummyjson.com/products?limit=12");
  if (!res.ok) throw new Error("No se pudieron cargar los productos");
  const data = await res.json();
  return data.products;
}

export default async function ProductosPage() {
  const productos = await obtenerProductos();

  return (
    <section className="grid grid-cols-2 gap-4 md:grid-cols-4">
      {productos.map((p) => (
        <ProductCard key={p.id} producto={p} />
      ))}
    </section>
  );
}
```

```tsx
// src/components/ProductCard.tsx
import Image from "next/image";
import Link from "next/link";

export default function ProductCard({ producto }: { producto: { id: number; title: string; price: number; thumbnail: string } }) {
  return (
    <article className="rounded border p-3">
      <Image src={producto.thumbnail} alt={producto.title} width={300} height={300} />
      <h2>{producto.title}</h2>
      <p>${producto.price}</p>
      <Link href={`/productos/${producto.id}`}>Ver detalle</Link>
    </article>
  );
}
```

Para usar `next/image` con imágenes externas hay que autorizar el dominio:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    remotePatterns: [{ protocol: "https", hostname: "cdn.dummyjson.com" }],
  },
};
```

## Caché y revalidación

Desde Next.js 15, las peticiones `fetch` **no se cachean por defecto**. Se controla por petición:

```tsx
// Cachear por tiempo: se vuelve a pedir como máximo cada 60 segundos
fetch(url, { next: { revalidate: 60 } });

// Siempre datos frescos
fetch(url, { cache: "no-store" });

// Cachear indefinidamente
fetch(url, { cache: "force-cache" });
```

Estrategia para la tienda: el catálogo se revalida cada pocos minutos; el stock y el carrito, nunca se cachean.

## Peticiones en paralelo

Evitar encadenar `await` innecesarios:

```tsx
const [producto, relacionados] = await Promise.all([
  obtenerProducto(id),
  obtenerRelacionados(id),
]);
```

## Estados de carga con `loading.tsx`

Un archivo `loading.tsx` se muestra mientras la página espera sus datos. Next.js lo envuelve automáticamente en un `Suspense`.

```tsx
// src/app/(tienda)/productos/loading.tsx
export default function Loading() {
  return <p>Cargando productos...</p>;
}
```

Para cargar partes de la página de forma independiente (*streaming*), se usa `Suspense` explícito:

```tsx
import { Suspense } from "react";

<h1>Detalle</h1>
<Suspense fallback={<p>Cargando reseñas...</p>}>
  <Resenas productoId={id} />
</Suspense>
```

## Manejo de errores con `error.tsx`

Debe ser un componente de cliente. Recibe el error y una función para reintentar.

```tsx
// src/app/(tienda)/productos/error.tsx
"use client";

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Algo salió mal: {error.message}</p>
      <button onClick={reset}>Reintentar</button>
    </div>
  );
}
```

## Código que nunca debe llegar al cliente

El paquete `server-only` hace fallar el build si un módulo con secretos se importa desde un componente de cliente.

```bash
npm install server-only
```

```ts
// src/lib/productos.ts
import "server-only";
```
