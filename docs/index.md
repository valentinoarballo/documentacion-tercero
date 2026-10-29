# Programación III

## Presentación

Esta materia lleva al estudiante del frontend con React al desarrollo **fullstack moderno**. Se trabaja con Next.js para construir aplicaciones con renderizado en servidor, base de datos, autenticación y pruebas; luego se explora Astro como alternativa orientada a contenido, y se cierra con el despliegue de una arquitectura completa (frontend, API y base de datos) en servicios reales.

## Objetivos

Al finalizar la cursada, el estudiante podrá:

- Construir aplicaciones con el App Router de Next.js: routing, layouts, componentes de servidor y de cliente.
- Exponer y consumir APIs propias e integrar un servicio de IA de forma segura.
- Modelar y consultar una base de datos PostgreSQL con Prisma.
- Autenticar usuarios con Clerk y sincronizarlos con la base de datos local.
- Validar formularios con React Hook Form y Zod, compartiendo esquemas entre cliente y servidor.
- Escribir pruebas automatizadas con Vitest.
- Desarrollar sitios con Astro usando rutas dinámicas, View Transitions e islas.
- Desplegar con Supabase, Vercel y Render, y autenticar contra una API Django.

## Proyecto transversal: la tienda online

Se continúa con el ecommerce de la materia anterior, esta vez como una aplicación fullstack real.

| Bloque | Temas | Qué se construye |
|--------|-------|------------------|
| Next.js | [1](tema1.md) a [7](tema7.md) | Catálogo renderizado en servidor, carrito, rutas, API propia y asistente de compras con IA |
| Datos y calidad | [8](tema8.md) a [13](tema13.md) | Base de datos, panel de administración, autenticación, formularios validados, pruebas y publicación |
| Astro | [14](tema14.md) a [17](tema17.md) | Vitrina de la tienda con Astro: rutas, transiciones, islas y validaciones |
| Arquitectura completa | [18](tema18.md) y [19](tema19.md) | Despliegue de frontend y API Django, con autenticación por tokens |

## Requisitos previos

- Aprobar o dominar los contenidos de *Desarrollo Frontend*: JavaScript moderno, DOM, React (componentes, hooks, Router, contexto) y Git.
- Nociones básicas de bases de datos relacionales y SQL.
- Manejo de la terminal.

Los ejemplos están escritos en **TypeScript** con anotaciones mínimas. Los tipos se explican cuando aparecen.

## Herramientas y cuentas

- Node.js 20.9 o superior y npm
- Git y una cuenta de GitHub
- Visual Studio Code
- Docker (opcional, para PostgreSQL local)
- Cuentas gratuitas en Vercel, Supabase, Clerk y Render
- Python 3.12 o superior (para el tema de Django)

## Cómo está organizada la documentación

Cada tema es una página independiente:

1. **Fundamentación:** un párrafo que indica qué se verá y cómo se relaciona con el tema anterior y el siguiente.
2. **Desarrollo:** secciones breves, cada una con explicación y ejemplo aplicado a la tienda.

Las herramientas de este curso cambian rápido (Next.js, Prisma, Clerk y Astro tienen versiones mayores recientes). Ante una diferencia entre un ejemplo y tu versión instalada, la documentación oficial manda.

## Evaluación

- **Entregas parciales** por bloque, sobre el repositorio de la tienda.
- **Examen final integrador:** aplicación fullstack completa, publicada y funcionando.
