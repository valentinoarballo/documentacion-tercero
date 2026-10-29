# Tema 18. Configuración de Render y Vercel para deploy fullstack

Hasta ahora la tienda usaba una base de datos propia accedida desde Next.js. Este tema publica una arquitectura **separada**: un frontend en Vercel que consume una API Django alojada en Render. Es el esquema habitual en equipos donde frontend y backend se desarrollan por separado. Prepara el tema 19, donde se agrega autenticación a esa API.

## Arquitectura

```text
Navegador ──► Vercel (Next.js o Astro) ──► Render (API Django + DRF) ──► PostgreSQL
```

| Servicio | Aloja | Variable clave |
|----------|-------|----------------|
| Vercel | Frontend | `NEXT_PUBLIC_API_URL` / `PUBLIC_API_URL` |
| Render | API Django | `DATABASE_URL`, `SECRET_KEY`, `CORS_ALLOWED_ORIGINS` |
| Render (o Supabase) | PostgreSQL | `DATABASE_URL` |

## Preparar Django para producción

Los ajustes de desarrollo no sirven en producción. Se leen los valores sensibles desde variables de entorno.

```bash
pip install gunicorn whitenoise dj-database-url psycopg[binary] django-cors-headers
pip freeze > requirements.txt
```

```python
# config/settings.py
import os
import dj_database_url
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ["SECRET_KEY"]
DEBUG = os.environ.get("DEBUG", "False") == "True"
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")

INSTALLED_APPS = [
    # ...apps de Django
    "corsheaders",
    "rest_framework",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",   # archivos estáticos
    "corsheaders.middleware.CorsMiddleware",         # antes de CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    # ...resto
]

DATABASES = {
    "default": dj_database_url.config(
        default=f"sqlite:///{BASE_DIR / 'db.sqlite3'}",
        conn_max_age=600,
    )
}

STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
STORAGES = {
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}

CORS_ALLOWED_ORIGINS = os.environ.get("CORS_ALLOWED_ORIGINS", "http://localhost:3000").split(",")
CSRF_TRUSTED_ORIGINS = os.environ.get("CSRF_TRUSTED_ORIGINS", "").split(",") if os.environ.get("CSRF_TRUSTED_ORIGINS") else []
```

Ajustar `config` al nombre real de la carpeta del proyecto.

## Script de build

```bash
#!/usr/bin/env bash
# build.sh
set -o errexit

pip install -r requirements.txt
python manage.py collectstatic --no-input
python manage.py migrate
```

```bash
chmod +x build.sh
```

## Crear el servicio en Render

1. En `render.com`: **New → Web Service** y conectar el repositorio.
2. Configurar:

| Campo | Valor |
|-------|-------|
| Runtime | Python 3 |
| Build Command | `./build.sh` |
| Start Command | `gunicorn config.wsgi:application` |
| Root Directory | Carpeta donde está `manage.py` (si no es la raíz) |

3. En **Environment**, definir las variables:

| Variable | Valor |
|----------|-------|
| `SECRET_KEY` | Cadena larga aleatoria |
| `DEBUG` | `False` |
| `ALLOWED_HOSTS` | `tu-api.onrender.com` |
| `DATABASE_URL` | URL de PostgreSQL (se obtiene al crear la base en Render, o de Supabase) |
| `CORS_ALLOWED_ORIGINS` | `https://tu-tienda.vercel.app` |
| `PYTHON_VERSION` | Versión de Python del proyecto |

Generar una `SECRET_KEY`:

```bash
python -c "from django.core.management.utils import get_random_secret_key as k; print(k())"
```

## Base de datos en Render

**New → PostgreSQL**, y copiar la *Internal Database URL* (si ambos servicios están en la misma región de Render) a `DATABASE_URL` del servicio web. Alternativamente, usar la URL de Supabase del tema 12.

## Configuración como código: `render.yaml`

```yaml
# render.yaml (en la raíz del repositorio)
services:
  - type: web
    name: tienda-api
    runtime: python
    buildCommand: ./build.sh
    startCommand: gunicorn config.wsgi:application
    envVars:
      - key: SECRET_KEY
        generateValue: true
      - key: DEBUG
        value: "False"
      - key: DATABASE_URL
        fromDatabase:
          name: tienda-db
          property: connectionString

databases:
  - name: tienda-db
    plan: free
```

## Conectar el frontend en Vercel

Se guarda la URL de la API en una variable de entorno y se usa en un único módulo.

```bash
# Vercel → Settings → Environment Variables
NEXT_PUBLIC_API_URL=https://tu-api.onrender.com
```

```ts
// src/lib/api.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL;

export async function api<T>(ruta: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${API_URL}${ruta}`, init);
  if (!res.ok) throw new Error(`Error ${res.status} en ${ruta}`);
  return res.json();
}
```

En Astro, las variables públicas se llaman `PUBLIC_API_URL` y se leen con `import.meta.env.PUBLIC_API_URL`.

## Orden de despliegue

1. Desplegar la **API** en Render y comprobar que responde.
2. Copiar su URL a Vercel como variable de entorno.
3. Desplegar el **frontend**.
4. Copiar la URL de Vercel a `CORS_ALLOWED_ORIGINS` en Render y redesplegar la API.

## Verificación

```bash
curl https://tu-api.onrender.com/api/productos/
```

## Problemas frecuentes

| Síntoma | Causa probable |
|---------|----------------|
| `DisallowedHost` | Falta el dominio en `ALLOWED_HOSTS` |
| Error de CORS en el navegador | `CORS_ALLOWED_ORIGINS` no incluye el dominio de Vercel (sin barra final) |
| Estilos del admin de Django rotos | Falta WhiteNoise o `collectstatic` |
| Error 500 sin detalle | Revisar los logs del servicio en Render |
| Primera petición muy lenta | El plan gratuito suspende el servicio por inactividad |
| `relation does not exist` | No se ejecutó `migrate` en el build |

El plan gratuito de Render **suspende el servicio tras un período sin tráfico**: la primera petición puede tardar decenas de segundos. Conviene mostrar un estado de carga en el frontend.
