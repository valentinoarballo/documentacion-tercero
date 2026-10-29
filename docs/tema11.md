# Tema 11. Testing con Vitest

Con lógica de negocio, validaciones y componentes interactivos, los cambios empiezan a romper cosas sin que se note. Este tema incorpora **Vitest** para escribir pruebas automatizadas sobre las piezas de la tienda: funciones, esquemas de Zod, componentes y Server Actions. Se apoya en los esquemas del tema 10 y los componentes del tema 6.

## Qué probar

| Elemento | Tipo de prueba | Herramienta |
|----------|----------------|-------------|
| Funciones puras (totales, descuentos) | Unitaria | Vitest |
| Esquemas de Zod | Unitaria | Vitest |
| Componentes de cliente | Componente | Vitest + Testing Library |
| Server Actions | Unitaria, con la base simulada | Vitest + mocks |
| Flujos completos (comprar) | End-to-end | Playwright (fuera de alcance) |

Vitest **no soporta Server Components asíncronos**. Para esos se prueban las funciones de datos que usan y se cubre la página completa con pruebas end-to-end.

## Instalación

```bash
npm install -D vitest @vitejs/plugin-react jsdom vite-tsconfig-paths \
  @testing-library/react @testing-library/dom @testing-library/user-event @testing-library/jest-dom
```

```ts
// vitest.config.mts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./vitest.setup.ts"],
  },
});
```

```ts
// vitest.setup.ts
import "@testing-library/jest-dom/vitest";
```

En `tsconfig.json`, agregar `"types": ["vitest/globals"]` dentro de `compilerOptions`. En `package.json`:

```json
{ "scripts": { "test": "vitest", "test:run": "vitest run" } }
```

## Prueba de una función

```ts
// src/lib/carrito.ts
export type Item = { precio: number; cantidad: number };

export function calcularTotal(items: Item[], envioGratisDesde = 200) {
  const subtotal = items.reduce((acc, i) => acc + i.precio * i.cantidad, 0);
  const envio = subtotal === 0 || subtotal >= envioGratisDesde ? 0 : 15;
  return { subtotal, envio, total: subtotal + envio };
}
```

```ts
// src/lib/carrito.test.ts
import { describe, it, expect } from "vitest";
import { calcularTotal } from "./carrito";

describe("calcularTotal", () => {
  it("suma el subtotal de los productos", () => {
    const { subtotal } = calcularTotal([{ precio: 50, cantidad: 2 }]);
    expect(subtotal).toBe(100);
  });

  it("cobra envío por debajo del mínimo", () => {
    expect(calcularTotal([{ precio: 50, cantidad: 1 }]).envio).toBe(15);
  });

  it("no cobra envío desde 200", () => {
    expect(calcularTotal([{ precio: 200, cantidad: 1 }]).envio).toBe(0);
  });

  it("devuelve cero con el carrito vacío", () => {
    expect(calcularTotal([])).toEqual({ subtotal: 0, envio: 0, total: 0 });
  });
});
```

```bash
npm test
```

## Prueba de un esquema de Zod

```ts
// src/lib/schemas/producto.test.ts
import { describe, it, expect } from "vitest";
import { productoSchema } from "./producto";

const valido = { name: "Teclado", price: 89.9, stock: 5, categoryId: 1 };

describe("productoSchema", () => {
  it("acepta datos válidos", () => {
    expect(productoSchema.safeParse(valido).success).toBe(true);
  });

  it("rechaza precio negativo", () => {
    const r = productoSchema.safeParse({ ...valido, price: -1 });
    expect(r.success).toBe(false);
  });

  it("rechaza nombre corto", () => {
    expect(productoSchema.safeParse({ ...valido, name: "ab" }).success).toBe(false);
  });
});
```

## Prueba de un componente

```tsx
// src/components/Contador.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { describe, it, expect } from "vitest";
import Contador from "./Contador";

describe("Contador", () => {
  it("incrementa hasta el stock disponible", async () => {
    const user = userEvent.setup();
    render(<Contador stock={2} />);

    await user.click(screen.getByRole("button", { name: "+" }));
    await user.click(screen.getByRole("button", { name: "+" }));

    expect(screen.getByText("2")).toBeInTheDocument(); // no supera el stock
  });
});
```

Principio: se busca por **rol y texto accesible** (`getByRole`, `getByText`), como lo percibe el usuario, no por clases o estructura interna.

### Componentes que usan contexto

```tsx
import { CartProvider } from "@/context/CartContext";

render(
  <CartProvider>
    <AddToCartButton producto={{ id: 1, title: "Teclado", price: 50 }} />
  </CartProvider>
);
```

### Componentes que usan el router

```tsx
import { vi } from "vitest";

vi.mock("next/navigation", () => ({
  useRouter: () => ({ push: vi.fn() }),
  usePathname: () => "/productos",
  useSearchParams: () => new URLSearchParams(),
}));
```

## Prueba de una Server Action con Prisma simulado

Se reemplaza el cliente de la base por un doble, para probar la lógica sin una base real.

```ts
// src/app/(admin)/admin/productos/actions.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("server-only", () => ({}));
vi.mock("next/cache", () => ({ revalidatePath: vi.fn() }));
vi.mock("@/lib/usuario", () => ({ requerirAdmin: vi.fn() }));
vi.mock("@/lib/prisma", () => ({
  prisma: { product: { create: vi.fn() } },
}));

import { prisma } from "@/lib/prisma";
import { crearProducto } from "./actions";

describe("crearProducto", () => {
  beforeEach(() => vi.clearAllMocks());

  it("guarda un producto válido", async () => {
    const r = await crearProducto({ name: "Teclado", price: 50, stock: 3, categoryId: 1 });

    expect(r.ok).toBe(true);
    expect(prisma.product.create).toHaveBeenCalledOnce();
  });

  it("no escribe en la base si los datos son inválidos", async () => {
    const r = await crearProducto({ name: "ab", price: -5, stock: 3, categoryId: 1 });

    expect(r.ok).toBe(false);
    expect(prisma.product.create).not.toHaveBeenCalled();
  });
});
```

## Cobertura y organización

```bash
npm install -D @vitest/coverage-v8
npx vitest run --coverage
```

- Los archivos de prueba conviven con el código: `archivo.test.ts` junto a `archivo.ts`.
- Cada prueba verifica **un comportamiento** y tiene un nombre que lo describe.
- Ejecutar `npm run test:run` antes de cada Pull Request (tema 14 de la materia anterior).
