# Plan de Pruebas — Dashboard con Datos Reales (SPEC014)

**Ver primero:** [README.md](./README.md) (ambiente, credenciales, formato de caso de prueba)

> 💡 Recomendación: ejecutar primero
> [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md) y
> [TESTING-Clientes.md](./TESTING-Clientes.md) — este documento muestra indicadores que se calculan a
> partir de los datos que se generan ahí (facturaciones, cobranzas, clientes con saldo). Sin esos datos,
> las cards igual se prueban, pero se van a ver todas en $0.

---

## 1. Introducción funcional — ¿qué problema resuelve esta funcionalidad?

Al entrar a AppGastos, la primera pantalla (el "Dashboard") mostraba tarjetas de ejemplo, con datos
**inventados** y sin relación con el negocio real (ej. "Turnos Hoy", "Jugadores Activos"), heredadas de
otro rubro de negocio. Un usuario real de una clínica no encontraba ahí ningún dato útil para saber cómo
viene el día o el mes: cuánto se facturó, cuánto se cobró, quién le debe plata a la clínica, qué servicio
se vendió más.

Además, no toda la información es para cualquier usuario: los montos de facturación/cobranza/deuda son
información sensible que solo debería ver un rol de mayor jerarquía (Administrador), mientras que otros
indicadores más operativos (cuántos artículos se vendieron, cuál fue el servicio más solicitado) los puede
ver cualquier usuario logueado.

## 2. Explicación funcional de la solución implementada

- El Dashboard ahora muestra **cards con datos reales**, agrupadas en 2 secciones:
  - **"Resumen Financiero"** (Total Facturado, Total Cobrado, Deuda de Clientes, Deuda de Obras Sociales,
    Caja Neto del Día, Cobranzas del Día, Pagos del Día): **solo visible para el rol `ADMIN`**.
  - **"Actividad"** (Artículos/Servicios Vendidos, Artículo/Servicio Más Vendido, y — solo en tenants de
    tipo Medicina, como Centro Médico Lincoln — Pacientes atendidos por Prácticas/Tratamientos): visible
    para **cualquier** usuario logueado.
- Hay filtros globales de **fecha** (por defecto, del día 1 del mes actual a hoy), **Tipo de Artículo** y
  **Rubro**, que afectan a las cards de facturación/ventas/ranking (no a las de "del día" ni a las de
  deuda, que no dependen del rango elegido).
- Cada card es **clickeable**: abre un modal con el detalle en grilla (la lista de movimientos, recibos o
  clientes que componen ese número).
- ⚠️ **Deuda técnica aceptada, no una falla a reportar:** el backend hoy **no** rechaza (403) si un
  usuario no-ADMIN llama directamente al endpoint de resumen financiero por fuera de la aplicación (ej.
  desde Postman); solo se oculta del lado visual. Queda documentado como riesgo conocido, no reportar como
  bug (ver caso DSH-014).

---

## 3. Preparación — usuario de prueba con otro rol

Para poder comparar lo que ve un `ADMIN` contra lo que ve otro rol, hace falta un segundo usuario.

### DSH-001 — Alta de un usuario con rol COLAB
**Ambiente:** App
**Precondición:** Login como `test@i134.edu` (se asume rol `ADMIN`). Menú **Configuraciones → Usuarios**.
**Pasos:**
1. Click en "➕ Nuevo Usuario". Usuario: `colab.lincoln@i134.edu`. Contraseña: definir una (ej.
   `Colab!2026`) y repetirla en "Confirmar Contraseña".
2. Rol: `COLAB`.
3. Guardar.
**Resultado esperado:** El usuario se crea correctamente y aparece en la grilla de Usuarios con Rol =
`COLAB`.

> Para el resto de los casos de este documento que requieren "loguearse como COLAB", usar una ventana de
> navegación privada/incógnito (para no cerrar la sesión de `test@i134.edu`), y loguearse en
> https://gestur-qa.neosisweb.ar con `colab.lincoln@i134.edu` / la contraseña definida arriba.

---

## 4. Casos de prueba — Visibilidad por rol

### DSH-002 — El usuario ADMIN ve ambas secciones
**Ambiente:** App
**Precondición:** Login como `test@i134.edu`.
**Pasos:**
1. Ir al Dashboard (pantalla de inicio tras el login).
**Resultado esperado:** Se ven 2 títulos de sección: "Resumen Financiero" (con 7 cards: Total Facturado,
Total Cobrado, Deuda de Clientes, Deuda de Obras Sociales, Caja Neto del Día, Cobranzas del Día, Pagos del
Día) y "Actividad" (con Artículos Vendidos, Servicios Vendidos, Artículo Más Vendido, Servicio Más
Vendido, Pacientes - Prácticas, Pacientes - Tratamientos).

### DSH-003 — Un usuario COLAB solo ve la sección "Actividad"
**Ambiente:** App
**Precondición:** DSH-001. Login como `colab.lincoln@i134.edu` (ventana privada).
**Pasos:**
1. Ir al Dashboard.
**Resultado esperado:** **No** se ve el título "Resumen Financiero" ni ninguna de sus 7 cards. Sí se ve
"Actividad" con sus cards, igual que para el ADMIN.

### DSH-004 — Las cards de Prácticas/Tratamientos aparecen porque el tenant es de tipo Medicina
**Ambiente:** App
**Precondición:** Cualquiera de los 2 logins anteriores.
**Pasos:**
1. Revisar la sección "Actividad".
**Resultado esperado:** Aparecen las cards "Pacientes - Prácticas" y "Pacientes - Tratamientos" (son
específicas de tenants `tenant_tipo = 'M'`, como Centro Médico Lincoln; en un tenant de otro rubro no
aparecerían).

---

## 5. Casos de prueba — Filtros y datos

### DSH-005 — El filtro de fechas viene con el mes actual por defecto
**Ambiente:** App
**Pasos:**
1. Entrar al Dashboard sin tocar nada.
2. Observar los campos "Fecha desde" y "Fecha hasta".
**Resultado esperado:** "Fecha desde" = día 1 del mes actual. "Fecha hasta" = fecha de hoy.

### DSH-006 — Cambiar el rango de fechas actualiza los valores
**Ambiente:** App
**Precondición:** Al menos una facturación de un mes anterior (o, si no hay datos históricos, simplemente
verificar que el valor cambia al acotar el rango a un solo día sin movimientos).
**Pasos:**
1. Cambiar "Fecha desde" y "Fecha hasta" a un rango sin ninguna venta cargada (ej. una fecha futura).
2. Click en "🔄 Actualizar".
**Resultado esperado:** Las cards de período (Total Facturado, Artículos/Servicios Vendidos, Artículo/
Servicio Más Vendido) pasan a mostrar $0 / "Sin datos". Las cards "del día" (Caja Neto, Cobranzas,
Pagos) **no** cambian, porque siempre usan la fecha de hoy sin importar el filtro.

### DSH-007 — El filtro de Tipo de Artículo / Rubro acota las cards de ventas
**Ambiente:** App
**Precondición:** Ventas registradas tanto de Artículos con stock como de Servicios (ver
[TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md)).
**Pasos:**
1. En el filtro "Rubro", elegir `Prácticas Dermatológicas`.
2. Click en "🔄 Actualizar".
**Resultado esperado:** "Total Facturado" y las cards de vendidos/ranking recalculan considerando
únicamente movimientos de ese rubro (el valor debería bajar respecto de "Todos", salvo que sea el único
rubro con ventas).

### DSH-008 — Click en una card abre el detalle en grilla
**Ambiente:** App
**Pasos:**
1. Click en la card "Total Facturado".
**Resultado esperado:** Se abre un modal con una grilla ("Detalle — Total Facturado") listando los
movimientos de venta del período filtrado (Fecha, Cliente, Artículo/Servicio, Cantidad, Total).

### DSH-009 — "Deuda de Clientes" excluye a las Obras Sociales
**Ambiente:** App
**Precondición:** Al menos un cliente común con saldo pendiente y una Obra Social con saldo pendiente (ver
[TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md), facturación dual).
**Pasos:**
1. Click en la card "Deuda de Clientes".
**Resultado esperado:** La grilla muestra clientes comunes con saldo pendiente, pero **no** incluye
ninguna Obra Social.

### DSH-010 — "Deuda de Obras Sociales" muestra solo Obras Sociales
**Ambiente:** App
**Pasos:**
1. Click en la card "Deuda de Obras Sociales".
**Resultado esperado:** La grilla muestra únicamente Obras Sociales con saldo pendiente (ej. "OSDE", con
el copago pendiente de cobro de una facturación dual).

### DSH-011 — "Artículo Más Vendido" vs "Servicio Más Vendido" nunca se mezclan
**Ambiente:** App
**Precondición:** Ventas de al menos un Artículo con stock y un Servicio en el período filtrado.
**Pasos:**
1. Click en "Artículo Más Vendido" y anotar el resultado.
2. Cerrar el modal y click en "Servicio Más Vendido".
**Resultado esperado:** El ranking de "Artículo Más Vendido" solo contiene productos con stock; el de
"Servicio Más Vendido" solo contiene servicios (Tratamientos/Prácticas). Ningún artículo aparece en ambos
rankings.

### DSH-012 — "Caja Neto del Día" es Cobranzas del día menos Pagos del día
**Ambiente:** App
**Precondición:** Al menos una cobranza y un pago registrados hoy.
**Pasos:**
1. Anotar los valores de "Cobranzas del Día" y "Pagos del Día".
2. Click en "Caja Neto del Día".
**Resultado esperado:** El modal muestra el desglose (Cobranzas del día / Pagos del día / Neto), y
`Neto = Cobranzas del día − Pagos del día`.

---

## 6. Regresión

### DSH-013 — El listado de "Nuevo Movimiento" / grilla de Movimientos sigue funcionando igual
**Ambiente:** App
**Precondición:** Al menos un movimiento cargado.
**Pasos:**
1. Ir a **Caja/Shop → (ítem de Ventas)** y abrir la grilla de movimientos existente.
**Resultado esperado:** La grilla muestra los mismos movimientos que antes de esta funcionalidad (esta
pantalla comparte el mismo endpoint que las cards del Dashboard, que fue ampliado con filtros opcionales
sin romper el comportamiento previo).

---

## 7. Pruebas desde Postman

### DSH-014 — Resumen del Dashboard
**Ambiente:** Postman
**Precondición:** Token válido (ver [README.md](./README.md)).
**Pasos:**
1. `GET {{baseUrl}}/dashboard/resumen?tenant_id={{tenant_id}}` (opcionalmente agregar
   `&fecha_desde=20260901&fecha_hasta=20260923`), con header `Authorization: Bearer {{token}}`.
**Resultado esperado:** 200 OK. Una única fila/objeto con todos los campos documentados (`total_facturado`,
`total_cobrado`, `deuda_clientes`, `deuda_obras_sociales`, `articulo_mas_vendido`, `servicio_mas_vendido`,
`pacientes_atendidos_practicas`, `pacientes_atendidos_tratamientos`, `caja_neto_dia`, `cobranzas_dia`,
`pagos_dia`). Nota: esta llamada responde 200 sin importar el rol del token usado (deuda técnica aceptada
de la Decisión 10 de la spec — no reportar como bug).

### DSH-015 — Detalle de deuda de clientes
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/dashboard/deuda-clientes?tenant_id={{tenant_id}}`
**Resultado esperado:** 200 OK, array de clientes con saldo pendiente.

### DSH-016 — Ranking de artículos/servicios más vendidos
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/dashboard/ranking-articulos?tenant_id={{tenant_id}}&fecha_desde=20260901&fecha_hasta=20260923&es_servicio=1`
**Resultado esperado:** 200 OK, array ordenado por cantidad vendida, solo de servicios (`es_servicio=1`).

### DSH-017 — Pacientes atendidos por Prácticas
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/dashboard/pacientes-atendidos?tenant_id={{tenant_id}}&fecha_desde=20260901&fecha_hasta=20260923&admite_convenios=1`
**Resultado esperado:** 200 OK, array de pacientes distintos atendidos en Prácticas en ese rango,
excluyendo Obras Sociales.

### DSH-018 — Regresión del endpoint de movimientos (usado también por el Dashboard)
**Ambiente:** Postman
**Pasos:**
1. `POST {{baseUrl}}/movimientos-enca/movs` con el mismo body que se usaba antes de esta spec (sin los
   parámetros nuevos `tipo_arti_id`/`rubro_arti_id`/`tipo_movim_grupo`).
**Resultado esperado:** 200 OK, con la misma respuesta que antes de ampliar el SP (sin regresión).
