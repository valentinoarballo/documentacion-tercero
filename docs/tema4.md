# Tema 4. Layouts

Un layout es la estructura que comparten varias rutas (encabezado, pie, menú) y que **se conserva al navegar**. Este tema arma el marco de la tienda sobre las rutas del tema 3. En el tema 5 se llenan esas páginas con datos obtenidos en el servidor.

## Layout raíz

Es obligatorio y define `<html>` y `<body>`. Se crea con el proyecto.

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: { default: "TechShop", template: "%s | TechShop" },
  description: "Tecnología al mejor precio",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="es">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

`next/font` descarga la fuente en el build y la sirve desde el propio dominio, sin peticiones externas.

## Layouts anidados

Cada carpeta puede tener su `layout.tsx`, que envuelve a sus páginas y subcarpetas.

```text
app/
├── layout.tsx              # raíz
└── (tienda)/
    ├── layout.tsx          # encabezado y pie de la tienda
    ├── productos/page.tsx
    └── carrito/page.tsx
```

```tsx
// src/app/(tienda)/layout.tsx
import Link from "next/link";

export default function TiendaLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <header className="flex gap-4 border-b p-4">
        <Link href="/">TechShop</Link>
        <Link href="/productos">Productos</Link>
        <Link href="/carrito">Carrito</Link>
      </header>
      <main className="p-6">{children}</main>
      <footer className="border-t p-4 text-sm">© TechShop</footer>
    </>
  );
}
```

## Layouts distintos por sección

Los grupos de rutas (tema 3) permiten que la tienda y el panel de administración tengan estructuras diferentes sin cambiar la URL.

```tsx
// src/app/(admin)/layout.tsx
export default function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-[200px_1fr]">
      <aside className="border-r p-4">Panel de administración</aside>
      <section className="p-6">{children}</section>
    </div>
  );
}
```

## Conservación del estado

Al navegar entre páginas que comparten layout, este **no se vuelve a renderizar** ni pierde su estado. Si se necesita que se reinicie en cada navegación, se usa `template.tsx` con la misma firma.

## Metadatos

Estáticos, con `export const metadata`. Dinámicos, con `generateMetadata`, que recibe los mismos `params` que la página.

```tsx
// src/app/(tienda)/productos/[id]/page.tsx
import type { Metadata } from "next";

type Props = { params: Promise<{ id: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const producto = await fetch(`https://dummyjson.com/products/${id}`).then((r) => r.json());

  return {
    title: producto.title,
    description: producto.description,
    openGraph: { images: [producto.thumbnail] },
  };
}
```

Gracias a la plantilla `%s | TechShop` del layout raíz, el título final es `Nombre del producto | TechShop`.

## Componentes del layout

Para mantener los layouts limpios, el encabezado se extrae a un componente:

```text
src/
├── app/
└── components/
    ├── Header.tsx
    └── Footer.tsx
```

```tsx
// src/app/(tienda)/layout.tsx
import Header from "@/components/Header";
import Footer from "@/components/Footer";
```

## Rutas especiales por carpeta

Además de `page.tsx` y `layout.tsx`, cada carpeta admite archivos con función reservada: `loading.tsx`, `error.tsx` y `not-found.tsx`. Se estudian en el tema 5.
