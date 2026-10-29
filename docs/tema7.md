# Tema 7. Fetch, API Routes y Chat con IA

Next.js permite crear endpoints HTTP dentro del mismo proyecto mediante **Route Handlers**. Este tema crea la API propia de la tienda y la usa para un asistente de compras con IA, manteniendo la clave del servicio **solo en el servidor**. Retoma `fetch` del tema 5 y los componentes de cliente del tema 6; en el tema 8 los datos pasan a una base de datos real.

## Route Handlers

Un archivo `route.ts` dentro de `app/` define un endpoint. Exporta una función por método HTTP.

```ts
// src/app/api/productos/route.ts
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const categoria = searchParams.get("categoria");

  const url = categoria
    ? `https://dummyjson.com/products/category/${categoria}`
    : "https://dummyjson.com/products";

  const res = await fetch(url);
  const data = await res.json();

  return NextResponse.json(data.products);
}

export async function POST(request: Request) {
  const body = await request.json();

  if (!body.nombre || typeof body.precio !== "number") {
    return NextResponse.json({ error: "Datos inválidos" }, { status: 400 });
  }

  return NextResponse.json({ id: Date.now(), ...body }, { status: 201 });
}
```

Probar desde el navegador en `http://localhost:3000/api/productos` o con `curl`:

```bash
curl -X POST http://localhost:3000/api/productos \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Teclado","precio":89.9}'
```

## Parámetros dinámicos

```ts
// src/app/api/productos/[id]/route.ts
import { NextResponse } from "next/server";

export async function GET(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const res = await fetch(`https://dummyjson.com/products/${id}`);

  if (!res.ok) return NextResponse.json({ error: "No encontrado" }, { status: 404 });
  return NextResponse.json(await res.json());
}
```

## Cuándo usar un Route Handler

| Caso | Alternativa |
|------|-------------|
| Leer datos para mostrar en una página | Componente de servidor (directo, sin API) |
| Cliente externo que consume tu API (app móvil, otro sitio) | Route Handler |
| Webhooks de terceros (tema 9, parte 2) | Route Handler |
| Llamar a un servicio con una clave secreta desde el navegador | Route Handler como intermediario |

Un componente de servidor **no debería llamar a su propia API**: accede directamente a los datos.

## Asistente de compras con IA

El navegador nunca debe conocer la clave del servicio de IA. El cliente llama a un Route Handler propio, y este llama al proveedor.

```bash
# .env.local
ANTHROPIC_API_KEY=tu_clave
AI_MODEL=claude-sonnet-5-5
```

Consultar en la documentación del proveedor los nombres de modelo vigentes. El patrón es el mismo para cualquier proveedor de IA.

### Endpoint

```ts
// src/app/api/chat/route.ts
import { NextResponse } from "next/server";

type Mensaje = { role: "user" | "assistant"; content: string };

export async function POST(request: Request) {
  const { mensajes } = (await request.json()) as { mensajes: Mensaje[] };

  // Validar entrada: nunca confiar en lo que envía el cliente
  if (!Array.isArray(mensajes) || mensajes.length === 0 || mensajes.length > 20) {
    return NextResponse.json({ error: "Conversación inválida" }, { status: 400 });
  }

  const catalogo = await fetch("https://dummyjson.com/products?limit=10&select=title,price,stock")
    .then((r) => r.json())
    .then((d) => JSON.stringify(d.products));

  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY!,
      "anthropic-version": "2023-06-01",
    },
    body: JSON.stringify({
      model: process.env.AI_MODEL,
      max_tokens: 500,
      system: `Sos el asistente de compras de TechShop. Respondé en español, breve y solo con productos de este catálogo: ${catalogo}`,
      messages: mensajes.map((m) => ({ role: m.role, content: String(m.content).slice(0, 1000) })),
    }),
  });

  if (!res.ok) {
    return NextResponse.json({ error: "El asistente no está disponible" }, { status: 502 });
  }

  const data = await res.json();
  return NextResponse.json({ respuesta: data.content?.[0]?.text ?? "" });
}
```

### Componente de chat

```tsx
// src/components/ChatAsistente.tsx
"use client";

import { useState } from "react";

type Mensaje = { role: "user" | "assistant"; content: string };

export default function ChatAsistente() {
  const [mensajes, setMensajes] = useState<Mensaje[]>([]);
  const [texto, setTexto] = useState("");
  const [cargando, setCargando] = useState(false);

  async function enviar(e: React.FormEvent) {
    e.preventDefault();
    if (!texto.trim()) return;

    const nuevos: Mensaje[] = [...mensajes, { role: "user", content: texto }];
    setMensajes(nuevos);
    setTexto("");
    setCargando(true);

    try {
      const res = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ mensajes: nuevos }),
      });
      const data = await res.json();
      setMensajes([...nuevos, { role: "assistant", content: data.respuesta ?? data.error }]);
    } finally {
      setCargando(false);
    }
  }

  return (
    <section>
      {mensajes.map((m, i) => (
        <p key={i} className={m.role === "user" ? "text-right" : ""}>{m.content}</p>
      ))}
      {cargando && <p>Escribiendo...</p>}
      <form onSubmit={enviar}>
        <input value={texto} onChange={(e) => setTexto(e.target.value)} placeholder="¿Qué estás buscando?" />
        <button type="submit" disabled={cargando}>Enviar</button>
      </form>
    </section>
  );
}
```

## Seguridad

- La clave vive en `.env.local` y se lee solo en el servidor; **nunca** usar el prefijo `NEXT_PUBLIC_`.
- Limitar el tamaño de la conversación y de cada mensaje, como se hizo arriba.
- En producción, agregar límite de peticiones por usuario (se resuelve con autenticación en el tema 9) para controlar costos.
- Para respuestas progresivas, el proveedor ofrece *streaming*; el Route Handler puede devolver un `ReadableStream`. Se omite aquí para mantener el ejemplo simple.
