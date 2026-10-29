# Tema 6. Componentes de cliente, estados y contexto

Los componentes de servidor no pueden reaccionar a clics ni guardar estado. Para eso existen los **componentes de cliente**. Este tema agrega a la tienda el botón de agregar al carrito y un carrito global con contexto, retomando `useState` y `useContext` de la materia anterior. En el tema 7 el carrito se conecta con rutas de API propias.

## La directiva `"use client"`

Se escribe en la primera línea del archivo. Ese componente y todo lo que importa se envían al navegador.

```tsx
// src/components/AddToCartButton.tsx
"use client";

import { useState } from "react";

export default function AddToCartButton({ stock }: { stock: number }) {
  const [cantidad, setCantidad] = useState(1);

  return (
    <div>
      <button onClick={() => setCantidad((c) => Math.max(1, c - 1))}>-</button>
      <span>{cantidad}</span>
      <button onClick={() => setCantidad((c) => Math.min(stock, c + 1))}>+</button>
      <button disabled={stock === 0}>Agregar al carrito</button>
    </div>
  );
}
```

## Cuándo usar cada tipo

| Necesito | Tipo |
|----------|------|
| Leer datos, acceder a la base de datos | Servidor |
| `useState`, `useEffect`, eventos | Cliente |
| `localStorage`, `window` | Cliente |
| Contexto y proveedores | Cliente |
| Reducir JavaScript | Servidor |

Regla práctica: **empezar en el servidor y mover a cliente solo la parte interactiva**.

## Cliente en las hojas

Marcar como cliente una página completa envía todo su código al navegador. Es mejor aislar lo interactivo en componentes pequeños:

```tsx
// Componente de servidor: obtiene datos y compone
export default async function ProductoPage({ params }: Props) {
  const { id } = await params;
  const producto = await obtenerProducto(id);

  return (
    <article>
      <h1>{producto.title}</h1>
      <p>{producto.description}</p>
      <AddToCartButton stock={producto.stock} /> {/* única parte de cliente */}
    </article>
  );
}
```

Las props que pasan del servidor al cliente deben ser **serializables** (números, strings, objetos simples). No pueden ser funciones ni instancias de clases.

## Contexto del carrito

```tsx
// src/context/CartContext.tsx
"use client";

import { createContext, useContext, useEffect, useState } from "react";

type Item = { id: number; nombre: string; precio: number; cantidad: number };

type CartContextType = {
  items: Item[];
  agregar: (item: Omit<Item, "cantidad">) => void;
  quitar: (id: number) => void;
  total: number;
};

const CartContext = createContext<CartContextType | null>(null);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  const [listo, setListo] = useState(false);

  // Leer localStorage solo en el cliente, después del primer render
  useEffect(() => {
    const guardado = localStorage.getItem("carrito");
    if (guardado) setItems(JSON.parse(guardado));
    setListo(true);
  }, []);

  useEffect(() => {
    if (listo) localStorage.setItem("carrito", JSON.stringify(items));
  }, [items, listo]);

  const agregar: CartContextType["agregar"] = (nuevo) =>
    setItems((prev) => {
      const existe = prev.find((i) => i.id === nuevo.id);
      return existe
        ? prev.map((i) => (i.id === nuevo.id ? { ...i, cantidad: i.cantidad + 1 } : i))
        : [...prev, { ...nuevo, cantidad: 1 }];
    });

  const quitar = (id: number) => setItems((prev) => prev.filter((i) => i.id !== id));
  const total = items.reduce((acc, i) => acc + i.precio * i.cantidad, 0);

  return <CartContext.Provider value={{ items, agregar, quitar, total }}>{children}</CartContext.Provider>;
}

export function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error("useCart debe usarse dentro de CartProvider");
  return ctx;
}
```

## Por qué se lee `localStorage` en un efecto

El servidor genera el HTML sin acceso a `localStorage`. Si el primer render del cliente usara datos guardados, no coincidiría con el HTML recibido y React lanzaría un error de *hydration mismatch*. Leer en `useEffect` evita el problema.

## Colocar el proveedor en el layout

Un componente de servidor puede renderizar un proveedor de cliente y pasarle `children`: el contenido de servidor **sigue siendo de servidor**.

```tsx
// src/app/(tienda)/layout.tsx
import { CartProvider } from "@/context/CartContext";

export default function TiendaLayout({ children }: { children: React.ReactNode }) {
  return (
    <CartProvider>
      <Header />
      {children}
    </CartProvider>
  );
}
```

## Usar el contexto

```tsx
"use client";
import { useCart } from "@/context/CartContext";

export default function AddToCartButton({ producto }: { producto: { id: number; title: string; price: number } }) {
  const { agregar } = useCart();
  return (
    <button onClick={() => agregar({ id: producto.id, nombre: producto.title, precio: producto.price })}>
      Agregar al carrito
    </button>
  );
}
```

## Filtros con parámetros de URL

Un componente de cliente que lee `useSearchParams` debe estar dentro de `Suspense` para no impedir la generación estática de la página.

```tsx
"use client";
import { useRouter, useSearchParams, usePathname } from "next/navigation";

export default function FiltroCategoria() {
  const router = useRouter();
  const pathname = usePathname();
  const params = useSearchParams();

  const cambiar = (categoria: string) => {
    const nuevos = new URLSearchParams(params);
    nuevos.set("categoria", categoria);
    router.push(`${pathname}?${nuevos.toString()}`);
  };

  return <button onClick={() => cambiar("laptops")}>Laptops</button>;
}
```

```tsx
<Suspense fallback={null}>
  <FiltroCategoria />
</Suspense>
```
