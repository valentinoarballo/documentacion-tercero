# Tema 10. React Hook Form y Zod

Los formularios del panel (tema 8, parte 2) leen el `FormData` sin validar. Este tema incorpora **React Hook Form** (manejo del formulario) y **Zod** (definición de reglas) para validar en el cliente y, con el mismo esquema, en el servidor. Retoma los formularios controlados de la materia anterior y las Server Actions del tema 8.

## Por qué estas dos librerías

| Herramienta | Resuelve |
|-------------|----------|
| React Hook Form | Estado del formulario, errores y envío, con pocos re-renderizados |
| Zod | Reglas de validación y tipos de TypeScript a partir de un único esquema |

## Instalación

```bash
npm install react-hook-form zod @hookform/resolvers
```

## Esquema de validación

El esquema se define una sola vez y se usa en cliente y servidor.

```ts
// src/lib/schemas/producto.ts
import { z } from "zod";

export const productoSchema = z.object({
  name: z.string().min(3, "El nombre debe tener al menos 3 caracteres"),
  description: z.string().max(500, "Máximo 500 caracteres").optional(),
  price: z.number({ message: "Ingresá un precio válido" }).positive("El precio debe ser mayor a 0"),
  stock: z.number({ message: "Ingresá el stock" }).int("Debe ser un entero").min(0, "No puede ser negativo"),
  categoryId: z.number({ message: "Elegí una categoría" }),
});

export type ProductoInput = z.infer<typeof productoSchema>;
```

`z.infer` genera el tipo de TypeScript desde el esquema: no hay que escribirlo dos veces.

## Formulario en el cliente

```tsx
// src/components/ProductoForm.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { productoSchema, type ProductoInput } from "@/lib/schemas/producto";
import { crearProducto } from "@/app/(admin)/admin/productos/actions";

type Props = { categorias: { id: number; name: string }[] };

export default function ProductoForm({ categorias }: Props) {
  const {
    register,
    handleSubmit,
    setError,
    reset,
    formState: { errors, isSubmitting },
  } = useForm<ProductoInput>({ resolver: zodResolver(productoSchema) });

  async function onSubmit(values: ProductoInput) {
    const resultado = await crearProducto(values);

    if (!resultado.ok) {
      // errores que detectó el servidor
      Object.entries(resultado.errores ?? {}).forEach(([campo, mensajes]) =>
        setError(campo as keyof ProductoInput, { message: mensajes?.[0] })
      );
      return;
    }
    reset();
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="grid gap-3" noValidate>
      <label>
        Nombre
        <input {...register("name")} />
        {errors.name && <span role="alert">{errors.name.message}</span>}
      </label>

      <label>
        Precio
        <input type="number" step="0.01" {...register("price", { valueAsNumber: true })} />
        {errors.price && <span role="alert">{errors.price.message}</span>}
      </label>

      <label>
        Stock
        <input type="number" {...register("stock", { valueAsNumber: true })} />
        {errors.stock && <span role="alert">{errors.stock.message}</span>}
      </label>

      <label>
        Categoría
        <select {...register("categoryId", { valueAsNumber: true })}>
          <option value="">Elegir...</option>
          {categorias.map((c) => (
            <option key={c.id} value={c.id}>{c.name}</option>
          ))}
        </select>
        {errors.categoryId && <span role="alert">{errors.categoryId.message}</span>}
      </label>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Guardando..." : "Guardar"}
      </button>
    </form>
  );
}
```

`register` conecta cada campo con el formulario sin crear un estado por campo. `valueAsNumber` convierte el texto del `input` a número antes de validar.

## Validar de nuevo en el servidor

La validación del cliente mejora la experiencia, pero **se puede saltear**: la del servidor es la que protege la base de datos.

```ts
// src/app/(admin)/admin/productos/actions.ts
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";
import { prisma } from "@/lib/prisma";
import { requerirAdmin } from "@/lib/usuario";
import { productoSchema, type ProductoInput } from "@/lib/schemas/producto";

export type ResultadoAccion =
  | { ok: true }
  | { ok: false; errores?: Record<string, string[] | undefined>; mensaje?: string };

export async function crearProducto(values: ProductoInput): Promise<ResultadoAccion> {
  await requerirAdmin();

  const parsed = productoSchema.safeParse(values);
  if (!parsed.success) {
    return { ok: false, errores: z.flattenError(parsed.error).fieldErrors };
  }

  try {
    await prisma.product.create({
      data: {
        name: parsed.data.name,
        description: parsed.data.description,
        price: parsed.data.price,
        stock: parsed.data.stock,
        categoryId: parsed.data.categoryId,
      },
    });
  } catch {
    return { ok: false, mensaje: "No se pudo guardar el producto" };
  }

  revalidatePath("/productos");
  return { ok: true };
}
```

## Reglas más avanzadas de Zod

```ts
// Validaciones entre campos
const registroSchema = z
  .object({
    email: z.email("Email inválido"),
    password: z.string().min(8, "Mínimo 8 caracteres"),
    confirmar: z.string(),
  })
  .refine((d) => d.password === d.confirmar, {
    message: "Las contraseñas no coinciden",
    path: ["confirmar"],
  });

// Valores permitidos
const estado = z.enum(["PENDIENTE", "PAGADO", "ENVIADO"]);

// Arrays
const pedidoSchema = z.object({
  items: z.array(z.object({ id: z.number(), cantidad: z.number().int().min(1) })).min(1, "El pedido está vacío"),
});
```

## Edición con valores iniciales

```tsx
useForm<ProductoInput>({
  resolver: zodResolver(productoSchema),
  defaultValues: { name: producto.name, price: producto.price, stock: producto.stock, categoryId: producto.categoryId },
});
```

## Buenas prácticas

- Un único esquema por entidad, en `src/lib/schemas/`.
- Mensajes de error claros y en el idioma de la interfaz.
- Validar **siempre** en el servidor, aunque el cliente ya lo haya hecho.
- Deshabilitar el botón mientras se envía (`isSubmitting`) para evitar envíos duplicados.
