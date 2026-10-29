# Tema 8 (parte 2). CRUD de productos con Prisma

Con la base configurada, se construye el panel de administración de productos: crear, leer, actualizar y eliminar. Las escrituras se hacen con **Server Actions**, funciones que se ejecutan en el servidor y se invocan desde formularios sin definir un endpoint. Continúa el tema 8; la validación robusta de estos formularios se agrega en el tema 10.

## Estructura

```text
src/
├── lib/productos.ts                       # consultas
└── app/(admin)/admin/productos/
    ├── page.tsx                           # listado
    ├── nuevo/page.tsx                     # alta
    ├── [id]/editar/page.tsx               # edición
    └── actions.ts                         # Server Actions
```

## Read: consultas

```ts
// src/lib/productos.ts
import "server-only";
import { prisma } from "@/lib/prisma";

export async function listarProductos() {
  const productos = await prisma.product.findMany({
    include: { category: true },
    orderBy: { id: "desc" },
  });
  return productos.map((p) => ({ ...p, price: Number(p.price) }));
}

export async function obtenerProducto(id: number) {
  const p = await prisma.product.findUnique({ where: { id } });
  return p ? { ...p, price: Number(p.price) } : null;
}

export function listarCategorias() {
  return prisma.category.findMany({ orderBy: { name: "asc" } });
}
```

```tsx
// src/app/(admin)/admin/productos/page.tsx
import Link from "next/link";
import { listarProductos } from "@/lib/productos";
import { eliminarProducto } from "./actions";

export default async function AdminProductos() {
  const productos = await listarProductos();

  return (
    <>
      <Link href="/admin/productos/nuevo">Nuevo producto</Link>
      <table>
        <tbody>
          {productos.map((p) => (
            <tr key={p.id}>
              <td>{p.name}</td>
              <td>${p.price}</td>
              <td>{p.stock}</td>
              <td>
                <Link href={`/admin/productos/${p.id}/editar`}>Editar</Link>
                <form action={eliminarProducto.bind(null, p.id)}>
                  <button type="submit">Eliminar</button>
                </form>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}
```

`bind(null, p.id)` pre-carga el argumento de la acción para cada fila.

## Create, Update y Delete: Server Actions

```ts
// src/app/(admin)/admin/productos/actions.ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { prisma } from "@/lib/prisma";

function leerDatos(formData: FormData) {
  return {
    name: String(formData.get("name") ?? "").trim(),
    description: String(formData.get("description") ?? "").trim() || null,
    price: Number(formData.get("price")),
    stock: Number(formData.get("stock")),
    categoryId: Number(formData.get("categoryId")),
  };
}

export async function crearProducto(formData: FormData) {
  await prisma.product.create({ data: leerDatos(formData) });
  revalidatePath("/productos");
  redirect("/admin/productos");
}

export async function actualizarProducto(id: number, formData: FormData) {
  await prisma.product.update({ where: { id }, data: leerDatos(formData) });
  revalidatePath("/productos");
  revalidatePath(`/productos/${id}`);
  redirect("/admin/productos");
}

export async function eliminarProducto(id: number) {
  await prisma.product.delete({ where: { id } });
  revalidatePath("/productos");
  revalidatePath("/admin/productos");
}
```

- `"use server"` marca todas las funciones del archivo como Server Actions.
- `revalidatePath` invalida la caché para que el listado muestre los cambios.
- `redirect` debe llamarse **fuera** de un bloque `try/catch`, porque funciona lanzando una excepción interna.

## Formulario de alta

```tsx
// src/app/(admin)/admin/productos/nuevo/page.tsx
import { listarCategorias } from "@/lib/productos";
import { crearProducto } from "../actions";

export default async function NuevoProducto() {
  const categorias = await listarCategorias();

  return (
    <form action={crearProducto} className="grid gap-3">
      <input name="name" placeholder="Nombre" required />
      <textarea name="description" placeholder="Descripción" />
      <input name="price" type="number" step="0.01" min="0" required />
      <input name="stock" type="number" min="0" required />
      <select name="categoryId" required>
        {categorias.map((c) => (
          <option key={c.id} value={c.id}>{c.name}</option>
        ))}
      </select>
      <button type="submit">Guardar</button>
    </form>
  );
}
```

## Formulario de edición

```tsx
// src/app/(admin)/admin/productos/[id]/editar/page.tsx
import { notFound } from "next/navigation";
import { listarCategorias, obtenerProducto } from "@/lib/productos";
import { actualizarProducto } from "../../actions";

type Props = { params: Promise<{ id: string }> };

export default async function EditarProducto({ params }: Props) {
  const { id } = await params;
  const producto = await obtenerProducto(Number(id));
  if (!producto) notFound();

  const categorias = await listarCategorias();

  return (
    <form action={actualizarProducto.bind(null, producto.id)} className="grid gap-3">
      <input name="name" defaultValue={producto.name} required />
      <textarea name="description" defaultValue={producto.description ?? ""} />
      <input name="price" type="number" step="0.01" defaultValue={producto.price} required />
      <input name="stock" type="number" defaultValue={producto.stock} required />
      <select name="categoryId" defaultValue={producto.categoryId}>
        {categorias.map((c) => (
          <option key={c.id} value={c.id}>{c.name}</option>
        ))}
      </select>
      <button type="submit">Actualizar</button>
    </form>
  );
}
```

## Consultas útiles de Prisma

```ts
// Filtros y búsqueda
prisma.product.findMany({
  where: { stock: { gt: 0 }, name: { contains: "teclado", mode: "insensitive" } },
});

// Paginación
prisma.product.findMany({ skip: (pagina - 1) * 12, take: 12 });

// Agregaciones
prisma.product.aggregate({ _avg: { price: true }, _count: true });

// Transacción: todo o nada
await prisma.$transaction([
  prisma.product.update({ where: { id }, data: { stock: { decrement: 1 } } }),
  // ...crear el pedido
]);
```

## Seguridad

Las Server Actions son **endpoints públicos**: cualquiera puede invocarlas, no solo quien ve el formulario. Por eso, en el tema 9 cada acción verifica que el usuario esté autenticado y sea administrador antes de escribir en la base.
