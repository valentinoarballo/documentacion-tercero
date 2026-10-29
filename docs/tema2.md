# Tema 2. Configuración inicial de Next.js

Se crea el proyecto de la tienda y se recorre su estructura. Todo el resto de la materia se apoya en este proyecto. El tema 3 usa la carpeta `app/` que se explica aquí para definir las rutas.

## Requisitos

Node.js 20.9 o superior. Next.js 16 usa Turbopack por defecto y React 19.

## Crear el proyecto

```bash
npx create-next-app@latest tienda
```

Respuestas recomendadas del asistente:

| Pregunta | Respuesta |
|----------|-----------|
| TypeScript | Yes |
| ESLint | Yes |
| Tailwind CSS | Yes |
| `src/` directory | Yes |
| App Router | Yes |
| Import alias | `@/*` |

Equivalente sin preguntas:

```bash
npx create-next-app@latest tienda --typescript --eslint --tailwind --app --src-dir --import-alias "@/*" --use-npm
```

## Ejecutar

```bash
cd tienda
npm run dev
```

Abrir `http://localhost:3000`. Los cambios se reflejan sin recargar (*Fast Refresh*).

| Script | Función |
|--------|---------|
| `npm run dev` | Servidor de desarrollo |
| `npm run build` | Compilación de producción |
| `npm run start` | Sirve el build de producción |
| `npm run lint` | Análisis con ESLint |

## Estructura

```text
tienda/
├── public/              # archivos estáticos (/logo.png)
├── src/
│   └── app/
│       ├── layout.tsx   # estructura común de todas las páginas
│       ├── page.tsx     # ruta "/"
│       └── globals.css
├── next.config.ts       # configuración del framework
├── tsconfig.json
└── package.json
```

El alias `@/` apunta a `src/`:

```tsx
import { formatearPrecio } from "@/lib/formato";
```

## Primera página de la tienda

Reemplazar el contenido de `src/app/page.tsx`:

```tsx
export default function Inicio() {
  return (
    <main className="p-8">
      <h1 className="text-3xl font-bold">TechShop</h1>
      <p>Tecnología al mejor precio.</p>
    </main>
  );
}
```

## TypeScript: lo mínimo necesario

Se anotan los tipos de las props y de los datos. El resto lo infiere el compilador.

```tsx
type Producto = {
  id: number;
  nombre: string;
  precio: number;
  stock: number;
};

function formatearPrecio(valor: number): string {
  return new Intl.NumberFormat("es-AR", { style: "currency", currency: "USD" }).format(valor);
}
```

Crear `src/lib/formato.ts` con la función anterior (exportándola con `export`) y `src/types/index.ts` con el tipo `Producto`.

## Configuración

`next.config.ts` centraliza la configuración. Se amplía en temas posteriores (imágenes remotas, por ejemplo).

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
```

## Versionar

```bash
git add .
git commit -m "chore: proyecto inicial de la tienda con Next.js"
```
