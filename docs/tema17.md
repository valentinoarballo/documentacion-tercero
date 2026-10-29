# Tema 17. Tests y validaciones en Astro

Se aplican a la vitrina de Astro las mismas ideas de calidad que en el tema 11: **pruebas automatizadas** con Vitest y **validación de datos** con Zod. Astro tiene la particularidad de que la entrada de datos suele llegar por endpoints o formularios de página, por lo que la validación se concentra allí.

## Vitest en Astro

Astro provee `getViteConfig` para que Vitest comparta la configuración del proyecto.

```bash
npm install -D vitest
```

```ts
// vitest.config.ts
/// <reference types="vitest/config" />
import { getViteConfig } from "astro/config";

export default getViteConfig({
  test: {
    include: ["src/**/*.test.ts"],
  },
});
```

```json
{ "scripts": { "test": "vitest", "test:run": "vitest run" } }
```

## Separar la lógica de los componentes

Los archivos `.astro` son difíciles de probar de forma aislada. La práctica recomendada es extraer la lógica a módulos `.ts` y probar esos módulos.

```ts
// src/lib/precios.ts
export function aplicarDescuento(precio: number, porcentaje: number) {
  if (porcentaje < 0 || porcentaje > 100) throw new Error("Porcentaje inválido");
  return Math.round(precio * (1 - porcentaje / 100) * 100) / 100;
}

export function formatearPrecio(valor: number) {
  return new Intl.NumberFormat("es-AR", { style: "currency", currency: "USD" }).format(valor);
}
```

```ts
// src/lib/precios.test.ts
import { describe, it, expect } from "vitest";
import { aplicarDescuento } from "./precios";

describe("aplicarDescuento", () => {
  it("aplica el porcentaje", () => {
    expect(aplicarDescuento(100, 10)).toBe(90);
  });

  it("no modifica el precio con descuento cero", () => {
    expect(aplicarDescuento(59.9, 0)).toBe(59.9);
  });

  it("rechaza porcentajes fuera de rango", () => {
    expect(() => aplicarDescuento(100, 150)).toThrow("Porcentaje inválido");
  });
});
```

## Esquemas de validación con Zod

```bash
npm install zod
```

```ts
// src/lib/schemas.ts
import { z } from "zod";

export const contactoSchema = z.object({
  nombre: z.string().trim().min(2, "Ingresá tu nombre"),
  email: z.email("Email inválido"),
  mensaje: z.string().trim().min(10, "El mensaje es muy corto").max(1000),
});

export type Contacto = z.infer<typeof contactoSchema>;
```

```ts
// src/lib/schemas.test.ts
import { describe, it, expect } from "vitest";
import { contactoSchema } from "./schemas";

const valido = { nombre: "Ana", email: "ana@mail.com", mensaje: "Consulta sobre un pedido" };

describe("contactoSchema", () => {
  it("acepta datos válidos", () => {
    expect(contactoSchema.safeParse(valido).success).toBe(true);
  });

  it("rechaza un email inválido", () => {
    expect(contactoSchema.safeParse({ ...valido, email: "no-es-email" }).success).toBe(false);
  });

  it("recorta espacios en el nombre", () => {
    const r = contactoSchema.parse({ ...valido, nombre: "  Ana  " });
    expect(r.nombre).toBe("Ana");
  });
});
```

## Endpoint con validación

Un endpoint (`src/pages/api/*.ts`) recibe datos del formulario de contacto de la tienda. Requiere un adaptador y `prerender = false`.

```ts
// src/pages/api/contacto.ts
import type { APIRoute } from "astro";
import { z } from "zod";
import { contactoSchema } from "../../lib/schemas";

export const prerender = false;

export const POST: APIRoute = async ({ request }) => {
  let datos: unknown;
  try {
    datos = await request.json();
  } catch {
    return Response.json({ error: "JSON inválido" }, { status: 400 });
  }

  const parsed = contactoSchema.safeParse(datos);
  if (!parsed.success) {
    return Response.json({ errores: z.flattenError(parsed.error).fieldErrors }, { status: 422 });
  }

  // ...guardar o enviar el mensaje
  return Response.json({ ok: true }, { status: 201 });
};
```

| Código | Significado |
|--------|-------------|
| 400 | Petición mal formada (JSON inválido) |
| 422 | JSON válido pero con datos que no cumplen las reglas |
| 201 | Recurso creado |

## Prueba del endpoint

El endpoint es una función que recibe una petición y devuelve una respuesta, así que se prueba sin levantar el servidor.

```ts
// src/pages/api/contacto.test.ts
import { describe, it, expect } from "vitest";
import { POST } from "./contacto";

function peticion(body: unknown) {
  return { request: new Request("http://localhost/api/contacto", {
    method: "POST",
    body: typeof body === "string" ? body : JSON.stringify(body),
  }) } as Parameters<typeof POST>[0];
}

describe("POST /api/contacto", () => {
  it("responde 201 con datos válidos", async () => {
    const res = await POST(peticion({ nombre: "Ana", email: "ana@mail.com", mensaje: "Hola, tengo una consulta" }));
    expect(res.status).toBe(201);
  });

  it("responde 422 con un email inválido", async () => {
    const res = await POST(peticion({ nombre: "Ana", email: "x", mensaje: "Hola, tengo una consulta" }));
    expect(res.status).toBe(422);
    expect((await res.json()).errores.email).toBeDefined();
  });

  it("responde 400 con JSON inválido", async () => {
    const res = await POST(peticion("{no es json"));
    expect(res.status).toBe(400);
  });
});
```

## Validación en el cliente

El formulario puede reutilizar el mismo esquema en una isla de React, igual que en el tema 10:

```tsx
import { contactoSchema } from "../lib/schemas";

const resultado = contactoSchema.safeParse(valores);
if (!resultado.success) {
  // mostrar resultado.error.issues
}
```

## Validar datos externos

Los datos de una API de terceros también deben validarse: el contrato puede cambiar sin aviso.

```ts
const productoApiSchema = z.object({
  id: z.number(),
  title: z.string(),
  price: z.number(),
  stock: z.number(),
});

const { products } = await res.json();
const productos = z.array(productoApiSchema).parse(products);
```

Si la API cambia su formato, el build falla con un error claro en lugar de publicar un sitio con datos rotos.

## Pruebas de extremo a extremo

Para verificar páginas completas, islas y transiciones, se usa una herramienta de navegador real como Playwright. Queda fuera del alcance de este tema, pero es el complemento natural de las pruebas unitarias.

## Buenas prácticas

- Lógica en módulos `.ts`, presentación en `.astro`.
- Validar toda entrada externa: formularios, endpoints y APIs de terceros.
- Ejecutar `npm run test:run` antes de integrar cambios y en el flujo de despliegue.
