# Tema 19. Autenticación con Django

En el tema 9 la autenticación la resolvía Clerk, un servicio externo. Aquí se implementa en la **propia API Django**, con tokens **JWT**, y se consume desde el frontend. Se apoya en la API desplegada del tema 18. Cierra el recorrido de la materia: una tienda con frontend y backend separados, protegida y publicada.

## Cómo funciona

```text
1. POST /api/auth/login/   {email, password}  →  { access, refresh }
2. GET  /api/auth/me/      Authorization: Bearer <access>  →  datos del usuario
3. POST /api/auth/refresh/ {refresh}          →  { access }  (el access expira rápido)
```

| Token | Duración | Uso |
|-------|----------|-----|
| `access` | Corta (minutos) | Se envía en cada petición |
| `refresh` | Larga (días) | Solo para obtener nuevos `access` |

## Backend: instalación

```bash
pip install djangorestframework djangorestframework-simplejwt
pip freeze > requirements.txt
```

```python
# config/settings.py
from datetime import timedelta

INSTALLED_APPS += ["rest_framework", "rest_framework_simplejwt"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ),
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
}
```

`IsAuthenticatedOrReadOnly` deja el catálogo público y exige sesión para modificar.

## Registro de usuarios

```python
# accounts/serializers.py
from django.contrib.auth import get_user_model
from rest_framework import serializers

User = get_user_model()

class RegistroSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)

    class Meta:
        model = User
        fields = ("id", "username", "email", "password")

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)  # hashea la contraseña
```

```python
# accounts/views.py
from django.contrib.auth import get_user_model
from rest_framework import generics, permissions
from rest_framework.response import Response
from rest_framework.views import APIView
from .serializers import RegistroSerializer

class RegistroView(generics.CreateAPIView):
    queryset = get_user_model().objects.all()
    serializer_class = RegistroSerializer
    permission_classes = [permissions.AllowAny]

class MeView(APIView):
    permission_classes = [permissions.IsAuthenticated]

    def get(self, request):
        u = request.user
        return Response({"id": u.id, "username": u.username, "email": u.email, "is_staff": u.is_staff})
```

## Rutas

```python
# accounts/urls.py
from django.urls import path
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
from .views import RegistroView, MeView

urlpatterns = [
    path("registro/", RegistroView.as_view()),
    path("login/", TokenObtainPairView.as_view()),
    path("refresh/", TokenRefreshView.as_view()),
    path("me/", MeView.as_view()),
]
```

```python
# config/urls.py
from django.urls import include, path

urlpatterns = [
    path("api/auth/", include("accounts.urls")),
    # ...
]
```

`TokenObtainPairView` autentica con el campo `username` del modelo de usuario por defecto.

## Proteger vistas y roles

```python
from rest_framework import permissions

class ProductoViewSet(viewsets.ModelViewSet):
    queryset = Producto.objects.all()
    serializer_class = ProductoSerializer

    def get_permissions(self):
        if self.action in ("list", "retrieve"):
            return [permissions.AllowAny()]
        return [permissions.IsAdminUser()]  # crear, editar y borrar: solo staff
```

## Probar con curl

```bash
curl -X POST http://127.0.0.1:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"username":"ana","password":"clave-segura-123"}'

curl http://127.0.0.1:8000/api/auth/me/ -H "Authorization: Bearer <access>"
```

## Frontend: dónde guardar el token

| Opción | Riesgo |
|--------|--------|
| `localStorage` | Cualquier script (XSS) puede leerlo |
| Cookie `httpOnly` | JavaScript del navegador **no puede** leerla |

Se recomienda la cookie `httpOnly`. En Next.js, un Route Handler actúa de intermediario: recibe las credenciales, llama a Django y guarda el token en una cookie.

### Login

```ts
// src/app/api/auth/login/route.ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";

export async function POST(request: Request) {
  const credenciales = await request.json();

  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/auth/login/`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(credenciales),
  });

  if (!res.ok) {
    return NextResponse.json({ error: "Credenciales inválidas" }, { status: 401 });
  }

  const { access, refresh } = await res.json();
  const jar = await cookies();
  const base = { httpOnly: true, secure: process.env.NODE_ENV === "production", sameSite: "lax" as const, path: "/" };

  jar.set("access", access, { ...base, maxAge: 60 * 15 });
  jar.set("refresh", refresh, { ...base, maxAge: 60 * 60 * 24 * 7 });

  return NextResponse.json({ ok: true });
}
```

### Logout

```ts
// src/app/api/auth/logout/route.ts
import { cookies } from "next/headers";
import { NextResponse } from "next/server";

export async function POST() {
  const jar = await cookies();
  jar.delete("access");
  jar.delete("refresh");
  return NextResponse.json({ ok: true });
}
```

### Leer el usuario en un componente de servidor

```ts
// src/lib/sesion.ts
import "server-only";
import { cookies } from "next/headers";

export async function obtenerUsuario() {
  const token = (await cookies()).get("access")?.value;
  if (!token) return null;

  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/auth/me/`, {
    headers: { Authorization: `Bearer ${token}` },
    cache: "no-store",
  });

  return res.ok ? res.json() : null;
}
```

```tsx
// src/app/(tienda)/perfil/page.tsx
import { redirect } from "next/navigation";
import { obtenerUsuario } from "@/lib/sesion";

export default async function Perfil() {
  const usuario = await obtenerUsuario();
  if (!usuario) redirect("/login");

  return <h1>Hola, {usuario.username}</h1>;
}
```

### Redirección temprana en `proxy.ts`

```ts
// src/proxy.ts (en Next.js 15 o anterior: middleware.ts)
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const hayToken = request.cookies.has("access") || request.cookies.has("refresh");

  if (!hayToken) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}

export const config = { matcher: ["/perfil/:path*", "/pedidos/:path*"] };
```

Esta comprobación solo verifica que **exista** una cookie. La verificación real del token la hace siempre Django: nunca confiar únicamente en el `proxy` para proteger datos.

## Renovar el token

Cuando Django responde 401, se pide un `access` nuevo con el `refresh`:

```ts
async function renovar(refresh: string) {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/auth/refresh/`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ refresh }),
  });
  return res.ok ? (await res.json()).access : null;
}
```

## Clerk o Django: cuándo usar cada uno

| | Clerk | Django + JWT |
|---|-------|--------------|
| Tiempo de implementación | Muy corto | Mayor |
| Control sobre usuarios y datos | Parcial (servicio externo) | Total |
| Funciones listas (OAuth, MFA, emails) | Incluidas | Hay que construirlas |
| Costo | Plan gratuito con límites | Solo infraestructura |
| Ideal para | Prototipos y productos propios | Sistemas con backend existente |

## Seguridad

- Contraseñas: nunca guardarlas en texto plano; `create_user` las hashea.
- Usar siempre HTTPS en producción (Render y Vercel lo proveen).
- Tiempos de vida cortos para `access` y `refresh` con rotación si el riesgo lo justifica.
- Limitar los orígenes permitidos en `CORS_ALLOWED_ORIGINS` (tema 18).
- Mensajes de error genéricos en el login: no revelar si el usuario existe.
