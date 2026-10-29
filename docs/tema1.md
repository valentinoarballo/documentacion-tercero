# Tema 1. Introducción a Programación III

Este tema da el panorama del año: qué problema resuelve Next.js frente a una aplicación React tradicional, cuál es el stack que se usa y cómo se organiza el trabajo. Parte de lo aprendido en la materia anterior (React, Router, contexto, despliegue) y prepara el tema 2, donde se crea el proyecto.

## De React a Next.js

React es una librería para construir interfaces. Next.js es un **framework** que agrega lo que una aplicación real necesita alrededor.

| Necesidad | React + Vite | Next.js |
|-----------|--------------|---------|
| Rutas | Se instala y configura React Router | Por carpetas, incluido |
| Renderizado | Cliente (el navegador arma la página) | Servidor, cliente o ambos |
| Datos | `useEffect` + `fetch` en el cliente | `async/await` directo en el componente |
| Backend | Proyecto aparte | Route Handlers y Server Actions en el mismo proyecto |
| SEO | Limitado (HTML vacío al inicio) | HTML completo desde el servidor |
| Optimización | Manual | Imágenes, fuentes y *code splitting* automáticos |

## Stack del año

| Capa | Herramienta | Tema |
|------|-------------|------|
| Framework | Next.js (App Router) | 2 a 7 |
| Base de datos | PostgreSQL + Prisma | 8, 12 |
| Autenticación | Clerk / Django + JWT | 9, 19 |
| Formularios | React Hook Form + Zod | 10 |
| Testing | Vitest | 11, 17 |
| Sitios de contenido | Astro | 14 a 17 |
| Despliegue | Vercel, Supabase, Render | 12, 13, 18 |

## Arquitectura de la tienda

```text
Navegador
   │
   ▼
Next.js (Vercel) ──► Clerk (autenticación)
   │
   ├──► PostgreSQL en Supabase (vía Prisma)
   ├──► API de IA (solo desde el servidor)
   └──► API Django en Render (temas 18 y 19)
```

## Preparar el entorno

```bash
node --version   # v20.9 o superior
npm --version
git --version
```

Crear las cuentas gratuitas: GitHub, Vercel, Supabase, Clerk y Render. No hace falta configurarlas todavía; cada tema indica cuándo se usan.

## Flujo de trabajo

El repositorio de la tienda sigue Gitflow (visto en *Desarrollo Frontend*):

- `main`: versión desplegada.
- `develop`: integración.
- `feature/*`: una rama por tema o funcionalidad.

Convención de commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`.

## Variables de entorno

Los secretos (claves de API, URLs de bases de datos) nunca se suben al repositorio. Se guardan en `.env.local`, que Next.js carga automáticamente y que ya está incluido en `.gitignore`.

```bash
# .env.local
DATABASE_URL="postgresql://..."
```

En el servidor se leen con `process.env.DATABASE_URL`. Solo las variables que empiezan con `NEXT_PUBLIC_` llegan al navegador.
