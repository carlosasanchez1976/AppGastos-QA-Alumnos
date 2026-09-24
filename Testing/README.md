# Plan de Pruebas QA — AppGastos (Centro Médico Lincoln)

**Proyecto:** AppGastos - Sistema de Gestión Administrativa
**Audiencia:** Alumnos de 1º año (rol Tester en Prácticas Profesionalizantes)
**Ambiente:** QA
**Fecha:** 2026-09-23

---

## 🎯 Objetivo de este material

Este es un plan de pruebas manual sobre 3 funcionalidades ya desarrolladas en AppGastos, pensado para
que un alumno sin experiencia previa en testing pueda **ejecutar cada caso sin supervisión constante**,
entendiendo primero qué problema de negocio resuelve la funcionalidad, y después verificando que se
comporta como se espera.

No se espera que los alumnos:
- Escriban o corran código.
- Consulten la base de datos directamente (todo se prueba desde la aplicación web o desde Postman).
- Conozcan de antemano el negocio de una clínica médica (por eso cada documento empieza con una
  introducción funcional).

## 🌐 Ambiente de pruebas

| Dato | Valor |
|---|---|
| URL de la aplicación (frontend) | https://gestur-qa.neosisweb.ar |
| URL base de la API (Postman) | https://api-gestur-qa.neosisweb.ar |
| Usuario | `test@i134.edu` |
| Clave | `Lincoln!2026` |
| Tenant | Centro Médico Lincoln |

> ⚠️ Este tenant debe existir previamente en QA con `tenant_tipo = 'M'` (Medicina) para que aparezcan los
> menús "Servicios" y "Médico", y para que el Dashboard muestre las cards de Prácticas/Tratamientos
> (creación de tenant a cargo de Charly/equipo, no es parte de las pruebas de los alumnos).

## 🔑 Login desde Postman (usado en todos los documentos)

Antes de poder llamar cualquier otro endpoint desde Postman hay que loguearse y guardar el `token` y el
`tenant_id` que devuelve la respuesta.

```
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
  "email": "test@i134.edu",
  "password": "Lincoln!2026"
}
```

**Respuesta esperada (200):**
```json
{
  "message": "Login exitoso",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "usuario": {
    "id": 1,
    "tenant_id": 9,
    "tenant_name": "Centro Médico Lincoln",
    "tenant_tipo": "M",
    "role": "ADMIN",
    "...": "..."
  }
}
```

Guardar en variables de Postman (Environment o Collection variables):
- `baseUrl` = `https://api-gestur-qa.neosisweb.ar`
- `token` = el valor de `token` de la respuesta
- `tenant_id` = el valor de `usuario.tenant_id` de la respuesta

En cada request posterior que requiera autenticación, agregar el header:
```
Authorization: Bearer {{token}}
```

## 🗂️ Documentos de este plan

| Documento | Cubre | Funcionalidad de negocio |
|---|---|---|
| [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md) | SPEC011, SPEC012, SPEC016 | Venta de servicios profesionales (Tratamientos/Prácticas) con convenios de obras sociales |
| [TESTING-Clientes.md](./TESTING-Clientes.md) | SPEC015 | Refactor de la entidad Clientes (grilla, selectores, performance) |
| [TESTING-Dashboard.md](./TESTING-Dashboard.md) | SPEC014 | Dashboard con indicadores reales del negocio, por rol de usuario |
| [TESTING-E2E-FlujoCompleto.md](./TESTING-E2E-FlujoCompleto.md) | Las 4 specs juntas | Casos de punta a punta: un paciente real pasa por todas las pantallas y los números "cierran" en Clientes, Cobranzas y Dashboard |

**Orden recomendado de ejecución:** Venta de Servicios → Clientes → Dashboard → End-to-End. Los primeros 3
documentos generan la masa de datos que se usa en el documento End-to-End; no hace falta cargar datos de
nuevo ahí.

## 📋 Formato de cada caso de prueba

Todos los documentos usan la misma estructura por caso:

| Campo | Significado |
|---|---|
| **ID** | Identificador único del caso (ej. `VDS-014`), para reportar bugs referenciando el caso exacto |
| **Ambiente** | `App` (desde la aplicación web) o `Postman` (llamando directo a la API) |
| **Funcionalidad a probar** | Qué comportamiento del sistema se está verificando |
| **Precondición** | Qué tiene que existir/estar cargado antes de ejecutar el caso |
| **Pasos** | Secuencia numerada de acciones concretas |
| **Resultado esperado** | Qué debe pasar si la funcionalidad está bien implementada |

## 🐞 Cómo reportar un bug encontrado

Si un caso no da el resultado esperado, documentarlo en `00-Documentación/01-Tareas/` con este formato mínimo:
- ID del caso de prueba que falló (ej. `VDS-014`).
- Pasos para reproducir (copiar los del caso, marcando en qué paso empieza a fallar).
- Resultado obtenido vs. resultado esperado.
- Captura de pantalla o respuesta de Postman (sin datos sensibles).

No se debe modificar código para "arreglar" un bug encontrado durante el testing — eso lo hace el equipo
de desarrollo a partir del reporte.
