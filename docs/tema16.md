# Tema 16. Server Islands e islas dinámicas en Astro

Las páginas estáticas del tema 15 son rápidas, pero algunas partes necesitan datos en tiempo real: el stock, el saludo al usuario o el contador del carrito. Las **server islands** permiten mantener la página estática y cargar solo esas piezas desde el servidor. Se complementan con las **islas de cliente** del tema 14.

## Dos tipos de isla

| | Isla de cliente | Server island |
|---|-----------------|---------------|
| Se ejecuta en | El navegador | El servidor, bajo demanda |
| Directiva | `client:*` | `server:defer` |
| Sirve para | Interactividad | Contenido dinámico o personalizado |
| Ejemplo | Selector de cantidad | Stock actual, saludo al usuario |

## Requisitos

Las server islands necesitan un **adaptador** de servidor (Vercel, Node, etc.):

```bash
npx astro add vercel
```

## Primera server island

Se marca un componente `.astro` con `server:defer`. La página se sirve estática (y cacheable) y ese componente se pide por separado.

```astro
---
// src/components/StockBadge.astro
interface Props { productoId: number }
const { productoId } = Astro.props;

const res = await fetch(`https://dummyjson.com/products/${productoId}?select=stock`);
const { stock } = await res.json();
---

<span class={stock > 0 ? "ok" : "agotado"}>
  {stock > 0 ? `${stock} disponibles` : "Agotado"}
</span>
```

```astro
---
// src/pages/productos/[id].astro
import StockBadge from "../../components/StockBadge.astro";
---

<h1>{producto.title}</h1>
<StockBadge productoId={producto.id} server:defer>
  <span slot="fallback">Consultando stock...</span>
</StockBadge>
```

Flujo:

1. El navegador recibe el HTML estático con el contenido de `fallback`.
2. Una petición separada pide el componente al servidor.
3. El resultado reemplaza al `fallback`.

La ficha del producto se entrega al instante desde la caché y solo el stock tiene latencia.

## Personalización con cookies y cabeceras

Una server island tiene acceso a la petición, por lo que puede leer la sesión sin volver dinámica toda la página.

```astro
---
// src/components/SaludoUsuario.astro
const nombre = Astro.cookies.get("nombre")?.value;
---

{nombre ? <span>Hola, {nombre}</span> : <a href="/login">Ingresar</a>}
```

```astro
<SaludoUsuario server:defer>
  <a slot="fallback" href="/login">Ingresar</a>
</SaludoUsuario>
```

## Reglas para las props

Las props de una server island se serializan y se envían en la petición, por lo que deben ser **datos simples**: números, strings, arrays y objetos. No pueden ser funciones ni elementos con contenido hijo (salvo el `slot` de `fallback`).

## Clave de cifrado en producción

Astro cifra las props de las server islands. Si cada despliegue genera una clave distinta, las páginas cacheadas con la clave anterior dejan de funcionar. Se fija una clave estable:

```bash
npx astro create-key
```

```bash
# Variable de entorno en Vercel
ASTRO_KEY=la_clave_generada
```

## Elegir entre estrategias

| Necesidad | Solución |
|-----------|----------|
| Página completa distinta por usuario | SSR (`prerender = false`) |
| Página común con una pieza dinámica | Server island |
| Pieza que necesita eventos del usuario | Isla de cliente |
| Contenido que cambia solo en cada build | Página estática |

## Combinar ambas islas

```astro
<h1>{producto.title}</h1>

<!-- Dinámico en el servidor -->
<StockBadge productoId={producto.id} server:defer>
  <span slot="fallback">...</span>
</StockBadge>

<!-- Interactivo en el cliente, cargado al hacerse visible -->
<AddToCart producto={producto} client:visible />
```

## Verificar el comportamiento

En las herramientas del navegador, pestaña **Network**, la página de producto debe mostrar:

- Una respuesta HTML inicial rápida y cacheable.
- Una petición adicional hacia la ruta interna de la server island.

Si la página tarda lo mismo que antes de usar `server:defer`, revisar que el adaptador esté instalado y que la página no se esté renderizando por completo en el servidor.
