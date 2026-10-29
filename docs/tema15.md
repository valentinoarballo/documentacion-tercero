# Tema 15. Rutas dinámicas y View Transitions en Astro

Continúa la vitrina del tema 14 con una página de detalle por producto y navegación animada entre páginas, sin recargar el documento. Las rutas dinámicas se resuelven en el *build* (o bajo demanda con un adaptador), y las **View Transitions** dan una sensación de aplicación a un sitio multipágina. En el tema 16 se agregan partes dinámicas dentro de estas páginas.

## Rutas basadas en archivos

| Archivo | Ruta |
|---------|------|
| `pages/index.astro` | `/` |
| `pages/productos/index.astro` | `/productos` |
| `pages/productos/[id].astro` | `/productos/42` |
| `pages/categorias/[...slug].astro` | `/categorias/tecnologia/monitores` |

## Rutas dinámicas estáticas con `getStaticPaths`

En modo estático Astro necesita saber **qué páginas generar** durante el build.

```astro
---
// src/pages/productos/[id].astro
import Layout from "../../layouts/Layout.astro";

export async function getStaticPaths() {
  const res = await fetch("https://dummyjson.com/products?limit=100");
  const { products } = await res.json();

  return products.map((p: any) => ({
    params: { id: String(p.id) },
    props: { producto: p },
  }));
}

const { producto } = Astro.props;
---

<Layout titulo={producto.title}>
  <h1>{producto.title}</h1>
  <img src={producto.thumbnail} alt={producto.title} width="400" height="400" />
  <p>{producto.description}</p>
  <p>${producto.price}</p>
</Layout>
```

- `params` define los valores del segmento dinámico (siempre `string`).
- `props` entrega los datos a cada página, evitando volver a pedirlos.
- `getStaticPaths` no puede usar variables definidas en el frontmatter: es una función independiente.

## Rutas dinámicas bajo demanda (SSR)

Para páginas que deben generarse en cada petición (por ejemplo, el stock en tiempo real) se necesita un **adaptador** y se desactiva el prerenderizado de esa página.

```bash
npx astro add vercel
```

```astro
---
// src/pages/productos/[id].astro
export const prerender = false;

const { id } = Astro.params;
const res = await fetch(`https://dummyjson.com/products/${id}`);

if (!res.ok) return Astro.redirect("/404");
const producto = await res.json();
---
```

| Modo | Cuándo se genera la página | Usar para |
|------|---------------------------|-----------|
| Estático (`getStaticPaths`) | En el build | Catálogos que cambian poco |
| Bajo demanda (`prerender = false`) | En cada petición | Datos en tiempo real, usuarios |

## Enlaces entre rutas

```astro
<a href={`/productos/${p.id}`}>Ver detalle</a>
```

## View Transitions

Se activan agregando el componente `ClientRouter` al `<head>` del layout. Astro intercepta la navegación, descarga la nueva página y reemplaza el contenido con una transición, sin recargar completamente.

```astro
---
// src/layouts/Layout.astro
import { ClientRouter } from "astro:transitions";
---

<head>
  <ClientRouter />
</head>
```

Con una línea, todos los enlaces del sitio pasan a navegar con animación (fundido por defecto).

### Transiciones compartidas entre páginas

Dos elementos con el mismo `transition:name` en páginas distintas se animan como si fueran el mismo. Es el efecto "la imagen del producto crece hacia el detalle".

```astro
<!-- En la tarjeta del listado -->
<img src={p.thumbnail} alt={p.title} transition:name={`img-${p.id}`} />

<!-- En la página de detalle -->
<img src={producto.thumbnail} alt={producto.title} transition:name={`img-${producto.id}`} />
```

### Tipos de animación

```astro
<main transition:animate="slide">...</main>
<aside transition:animate="none">...</aside>
```

Valores: `fade` (por defecto), `slide`, `initial`, `none`.

### Persistir elementos entre páginas

`transition:persist` conserva un elemento (y el estado de una isla) al navegar, en lugar de recrearlo.

```astro
<CartIcon client:load transition:persist />
```

Para que el contenido del carrito sobreviva a la navegación y se comparta entre islas, se usa un almacén externo, como **nanostores**:

```bash
npm install nanostores @nanostores/react
```

```ts
// src/stores/cart.ts
import { atom, computed } from "nanostores";

export type Item = { id: number; nombre: string; precio: number; cantidad: number };

export const carrito = atom<Item[]>([]);
export const total = computed(carrito, (items) =>
  items.reduce((acc, i) => acc + i.precio * i.cantidad, 0)
);

export function agregar(item: Omit<Item, "cantidad">) {
  const actual = carrito.get();
  const existe = actual.find((i) => i.id === item.id);
  carrito.set(
    existe
      ? actual.map((i) => (i.id === item.id ? { ...i, cantidad: i.cantidad + 1 } : i))
      : [...actual, { ...item, cantidad: 1 }]
  );
}
```

```tsx
// src/components/CartCount.tsx
import { useStore } from "@nanostores/react";
import { carrito } from "../stores/cart";

export default function CartCount() {
  const items = useStore(carrito);
  return <span>Carrito ({items.reduce((n, i) => n + i.cantidad, 0)})</span>;
}
```

A diferencia de un contexto de React, el almacén es un módulo y funciona entre islas independientes.

## Scripts y eventos con transiciones

Con `ClientRouter` los scripts de la página no se vuelven a ejecutar al navegar. Se usan los eventos del ciclo de vida:

```astro
<script>
  document.addEventListener("astro:page-load", () => {
    console.log("La página terminó de cargar (inicial o por navegación)");
  });
</script>
```

| Evento | Momento |
|--------|---------|
| `astro:before-preparation` | Antes de pedir la nueva página |
| `astro:after-swap` | Tras reemplazar el contenido |
| `astro:page-load` | La página está lista (también en la carga inicial) |

## Precarga de enlaces

```astro
<a href="/productos" data-astro-prefetch>Productos</a>
```

Astro descarga la página al pasar el cursor, haciendo la navegación casi instantánea.

## Excluir una navegación

```astro
<a href="/descarga" data-astro-reload>Descargar catálogo</a>
```
