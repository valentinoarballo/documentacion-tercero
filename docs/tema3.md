# Tema 3. Routing

En React Router las rutas se declaran en código; en Next.js se definen con la **estructura de carpetas**. Este tema arma las rutas de la tienda: catálogo, detalle de producto y secciones agrupadas. Usa la carpeta `app/` del tema 2 y prepara el tema 4, donde se comparten layouts entre rutas.

## Reglas básicas

- Cada carpeta dentro de `app/` es un segmento de la URL.
- Una carpeta es ruta pública solo si contiene un archivo `page.tsx`.

```text
src/app/
├── page.tsx                    → /
├── productos/
│   ├── page.tsx                → /productos
│   └── [id]/
│       └── page.tsx            → /productos/42
└── carrito/
    └── page.tsx                → /carrito
```

## Página estática

```tsx
// src/app/carrito/page.tsx
export default function CarritoPage() {
  return <h1>Tu carrito</h1>;
}
```

## Rutas dinámicas

Una carpeta entre corchetes es un parámetro. Desde Next.js 15, `params` es una **promesa** y se lee con `await`.

```tsx
// src/app/productos/[id]/page.tsx
import { notFound } from "next/navigation";

type Props = { params: Promise<{ id: string }> };

export default async function ProductoPage({ params }: Props) {
  const { id } = await params;

  const res = await fetch(`https://dummyjson.com/products/${id}`);
  if (!res.ok) notFound();

  const producto = await res.json();
  return <h1>{producto.title}</h1>;
}
```

## Parámetros de búsqueda

`searchParams` también es una promesa. Una página que lo lee se renderiza en cada petición.

```tsx
// /productos?categoria=laptops
type Props = { searchParams: Promise<{ categoria?: string }> };

export default async function ProductosPage({ searchParams }: Props) {
  const { categoria } = await searchParams;
  return <h1>Categoría: {categoria ?? "todas"}</h1>;
}
```

## Segmentos catch-all

`[...slug]` captura varios niveles; `[[...slug]]` además acepta cero niveles.

```text
app/categorias/[...slug]/page.tsx   → /categorias/tecnologia/monitores
```

## Grupos de rutas

Una carpeta entre paréntesis organiza el código **sin afectar la URL**.

```text
app/
├── (tienda)/
│   ├── productos/page.tsx      → /productos
│   └── carrito/page.tsx        → /carrito
└── (admin)/
    └── admin/page.tsx          → /admin
```

## Navegación

```tsx
import Link from "next/link";

<Link href="/productos">Ver productos</Link>
<Link href={`/productos/${producto.id}`}>Ver detalle</Link>
```

`Link` precarga la ruta cuando es visible, por lo que la navegación es inmediata. Para navegar desde código (en un componente de cliente):

```tsx
"use client";
import { useRouter, usePathname } from "next/navigation";

const router = useRouter();
const pathname = usePathname();

router.push("/carrito");
router.replace("/login");
router.back();
```

## Página 404

```tsx
// src/app/not-found.tsx  (404 global)
export default function NotFound() {
  return <h1>La página que buscás no existe</h1>;
}
```

`notFound()`, llamada desde una página, renderiza el `not-found.tsx` más cercano.

## Generación estática de rutas dinámicas

Para pre-generar en el build las páginas de producto:

```tsx
export async function generateStaticParams() {
  const res = await fetch("https://dummyjson.com/products?limit=20");
  const { products } = await res.json();
  return products.map((p: { id: number }) => ({ id: String(p.id) }));
}
```

## Comparación con React Router

| React Router | Next.js |
|--------------|---------|
| `<Route path="/productos/:id" />` | `app/productos/[id]/page.tsx` |
| `useParams()` | `await params` |
| `useSearchParams()` | `await searchParams` (servidor) |
| `<Outlet />` | `children` en `layout.tsx` (tema 4) |
