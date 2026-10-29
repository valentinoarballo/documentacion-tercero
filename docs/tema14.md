# Tema 14. Introducción a Astro

Next.js es ideal para aplicaciones con mucha interacción y datos de usuario. **Astro** apunta a otro caso: sitios centrados en contenido (vitrinas, catálogos, blogs) donde conviene enviar **el mínimo de JavaScript posible**. Este tema crea la vitrina de la tienda con Astro y la compara con Next.js. Los temas 15 a 17 profundizan en rutas, islas y pruebas.

## Idea central

Astro genera HTML y **no envía JavaScript al navegador salvo que se lo pidas**. Las partes interactivas son "islas" aisladas dentro de una página estática.

| | Next.js | Astro |
|---|---------|-------|
| Enfoque | Aplicaciones | Sitios de contenido |
| JavaScript por defecto | Sí (React) | No |
| Frameworks de UI | React | React, Vue, Svelte, etc. (mezclables) |
| Ideal para | Panel, carrito, usuarios | Catálogo, landing, documentación |

## Crear el proyecto

```bash
npm create astro@latest vitrina
cd vitrina
npm run dev
```

Elegir la plantilla mínima ("Empty") y TypeScript en modo estricto. El servidor corre en `http://localhost:4321`.

```text
vitrina/
├── public/                 # archivos estáticos
├── src/
│   ├── components/
│   ├── layouts/
│   └── pages/              # cada archivo es una ruta
│       └── index.astro
├── astro.config.mjs
└── package.json
```

## Componentes `.astro`

Un archivo `.astro` tiene dos partes: el **frontmatter** (código que corre en el servidor o en el build) y la **plantilla** (HTML con expresiones).

```astro
---
// src/components/ProductCard.astro
interface Props {
  nombre: string;
  precio: number;
  imagen: string;
}

const { nombre, precio, imagen } = Astro.props;
---

<article class="tarjeta">
  <img src={imagen} alt={nombre} width="300" height="300" loading="lazy" />
  <h2>{nombre}</h2>
  <p>${precio}</p>
</article>

<style>
  .tarjeta {
    border: 1px solid #ddd;
    padding: 0.75rem;
    border-radius: 0.5rem;
  }
</style>
```

Los estilos de `<style>` están **acotados al componente** por defecto.

## Layouts y slots

```astro
---
// src/layouts/Layout.astro
interface Props { titulo: string }
const { titulo } = Astro.props;
---

<!doctype html>
<html lang="es">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
    <title>{titulo} | TechShop</title>
  </head>
  <body>
    <header><a href="/">TechShop</a> <a href="/productos">Productos</a></header>
    <main><slot /></main>
  </body>
</html>
```

`<slot />` cumple el rol de `children` en React.

## Obtener datos

El frontmatter admite `await` de nivel superior. Por defecto se ejecuta **en el build**, y el HTML resultante ya contiene los datos.

```astro
---
// src/pages/productos/index.astro
import Layout from "../../layouts/Layout.astro";
import ProductCard from "../../components/ProductCard.astro";

const res = await fetch("https://dummyjson.com/products?limit=12");
const { products } = await res.json();
---

<Layout titulo="Productos">
  <h1>Catálogo</h1>
  <section class="grid">
    {products.map((p: any) => (
      <ProductCard nombre={p.title} precio={p.price} imagen={p.thumbnail} />
    ))}
  </section>
</Layout>
```

## Islas de interactividad

Para agregar un componente interactivo se usa un framework de UI. Se instala la integración de React:

```bash
npx astro add react
```

```tsx
// src/components/AddToCart.tsx
import { useState } from "react";

export default function AddToCart({ stock }: { stock: number }) {
  const [cantidad, setCantidad] = useState(1);
  return (
    <div>
      <button onClick={() => setCantidad((c) => Math.max(1, c - 1))}>-</button>
      <span>{cantidad}</span>
      <button onClick={() => setCantidad((c) => Math.min(stock, c + 1))}>+</button>
    </div>
  );
}
```

Sin una directiva `client:*`, el componente se renderiza como HTML estático y **no es interactivo**:

```astro
---
import AddToCart from "../components/AddToCart.tsx";
---

<AddToCart stock={10} client:visible />
```

| Directiva | Cuándo se carga el JavaScript |
|-----------|-------------------------------|
| `client:load` | Al cargar la página |
| `client:idle` | Cuando el navegador está desocupado |
| `client:visible` | Cuando el componente entra en pantalla |
| `client:media="(max-width: 600px)"` | Si se cumple una media query |
| `client:only="react"` | Solo en el cliente (sin renderizado en servidor) |

Para la tienda, `client:visible` en el botón de agregar evita cargar JavaScript de productos que el usuario nunca ve.

## Compilar

```bash
npm run build     # genera dist/
npm run preview
```

El resultado es una carpeta de archivos estáticos que se puede publicar en cualquier hosting.

## Cuándo elegir cada herramienta para la tienda

| Parte de la tienda | Herramienta |
|--------------------|-------------|
| Vitrina pública, landing, fichas de producto | Astro |
| Carrito, checkout, panel de administración | Next.js |
