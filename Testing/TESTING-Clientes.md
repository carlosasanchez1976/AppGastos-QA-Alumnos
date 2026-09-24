# Plan de Pruebas — Refactorización de Clientes (SPEC015)

**Ver primero:** [README.md](./README.md) (ambiente, credenciales, formato de caso de prueba)

---

## 1. Introducción funcional — ¿qué problema resuelve esta funcionalidad?

Antes de esta mejora, la grilla de **Clientes** tenía dos problemas:

1. **Bug visible:** mostraba mezcladas, en la misma lista, tanto a los pacientes/clientes comunes como a
   las **Obras Sociales** (que técnicamente también son un registro de "cliente", solo que de un tipo
   especial). Un usuario que entraba a "Clientes" buscando un paciente se encontraba con "OSDE", "IOMA",
   etc. mezcladas en el medio.
2. **Problema de rendimiento/diseño (no visible para el usuario, pero sí para el sistema):** cada vez que
   se abría un simple selector de cliente (por ejemplo, para elegir a quién facturarle en "Nuevo
   Movimiento"), el sistema calculaba por detrás el **saldo total adeudado** de *todos* los clientes —un
   cálculo pesado (recorre todos los movimientos) que solo hace falta en la grilla de Clientes, no en un
   selector que ni siquiera muestra esa columna.

## 2. Explicación funcional de la solución implementada

- Se separó la consulta de clientes en **2 variantes**: una **liviana** (sin calcular saldo, usada por
  cualquier selector de cliente en Movimientos, Cobranzas, Facturación de Servicios, etc.) y otra **con
  saldo** (más pesada, usada **únicamente** por la grilla de Clientes, que sí necesita mostrar esa
  columna).
- Ambas variantes aceptan los mismos 2 filtros opcionales por tipo de cliente (usando el código, ej.
  `'OBR'` para Obra Social): **filtro directo** (traer solo ese tipo) y **filtro inverso** (traer todos
  *menos* ese tipo). Con el filtro inverso se resuelve el bug: la grilla de Clientes ahora excluye
  explícitamente a las Obras Sociales.
- En el frontend, se unificaron 2 mecanismos de caché distintos que hacían lo mismo con lógica diferente
  en un solo hook reutilizable, que además agrega caché a la grilla de Clientes (antes no tenía) y expone
  una función para invalidar ese caché después de crear/editar/eliminar un cliente, para que la grilla
  siempre se vea actualizada sin tener que recargar la página.

---

## 3. Casos de prueba

### CLI-001 — Alta de un Cliente común
**Ambiente:** App
**Precondición:** Login en Centro Médico Lincoln. Menú **Caja/Shop → Clientes**.
**Pasos:**
1. Click en "➕ Nuevo Cliente". Apellido: `Fernández`. Nombre: `Roberto`. Tipo de Cliente: cualquiera que
   no sea "Obra Social" (ese tipo no debería ni aparecer en este desplegable, ver CLI-002).
2. Guardar.
**Resultado esperado:** El cliente se crea y aparece **inmediatamente** en la grilla, sin necesidad de
recargar la página, con columna "Saldo" en $0.

### CLI-002 — El tipo de cliente "Obra Social" no se ofrece al dar de alta un Cliente común
**Ambiente:** App
**Pasos:**
1. En "Nuevo Cliente", abrir el desplegable "Tipo de Cliente".
**Resultado esperado:** No aparece ninguna opción de tipo "Obra Social" — esa alta tiene su propia
pantalla dedicada (ver [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md), Bloque 1), para no
duplicar el alta de un mismo concepto desde 2 lugares distintos.

### CLI-003 — La grilla de Clientes NO muestra Obras Sociales (fix del bug principal)
**Ambiente:** App
**Precondición:** Al menos una Obra Social creada (ej. "OSDE", ver
[TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md) caso VDS-010).
**Pasos:**
1. Ir a **Caja/Shop → Clientes**.
2. Recorrer toda la grilla (o buscar "OSDE").
**Resultado esperado:** "OSDE" no aparece en ningún lado de esta grilla, aunque sí exista como registro
(verificable en **Servicios → Obras Sociales**).

### CLI-004 — La grilla de Clientes SÍ muestra el saldo adeudado
**Ambiente:** App
**Precondición:** Un cliente con al menos un movimiento con saldo pendiente (ej. facturar una Práctica sin
cobrarla del todo, ver [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md) caso VDS-042/VDS-043).
**Pasos:**
1. Ir a **Caja/Shop → Clientes** y ubicar a ese cliente.
**Resultado esperado:** La columna "Saldo" muestra un importe mayor a $0, coincidente con lo pendiente de
cobro.

### CLI-005 — El selector de cliente en "Nuevo Movimiento" no muestra Obras Sociales
**Ambiente:** App
**Precondición:** Al menos una Obra Social creada.
**Pasos:**
1. Ir a **Caja/Shop → (ítem de "Ventas"/Nuevo Ingreso, según el parámetro configurado)**.
2. Abrir el buscador/selector de cliente y buscar "OSDE".
**Resultado esperado:** No aparece — este selector se corrigió específicamente para no mezclar Obras
Sociales con clientes/pacientes comunes (las Obras Sociales se facturan desde la pantalla dedicada de
Facturación de Servicios, no desde acá).

### CLI-006 — Editar un cliente refresca la grilla sin recargar la página
**Ambiente:** App
**Precondición:** CLI-001.
**Pasos:**
1. En la grilla de Clientes, editar a "Roberto Fernández": cambiar el teléfono.
2. Guardar.
**Resultado esperado:** La grilla muestra el teléfono actualizado inmediatamente, sin necesidad de recargar
manualmente el navegador.

### CLI-007 — Buscar por nombre en la grilla de Clientes
**Ambiente:** App
**Precondición:** CLI-001 y al menos otro cliente con apellido distinto.
**Pasos:**
1. Escribir "Fernández" en el buscador de la grilla de Clientes y ejecutar la búsqueda.
**Resultado esperado:** Solo se listan los clientes cuyo nombre/apellido coincide con "Fernández" (la
búsqueda es client-side, sobre los datos ya cacheados).

---

## 4. Pruebas desde Postman

### CLI-008 — Endpoint liviano: todos los clientes (sin filtro)
**Ambiente:** Postman
**Precondición:** Token válido (ver [README.md](./README.md)).
**Pasos:**
1. `GET {{baseUrl}}/clientes/listar/{{tenant_id}}` con header `Authorization: Bearer {{token}}`.
**Resultado esperado:** 200 OK. El array incluye clientes comunes **y** Obras Sociales (sin filtrar). Cada
elemento **no** trae el campo `saldo_total` (es la variante liviana, sin cálculo de saldo).

### CLI-009 — Endpoint liviano: excluir Obras Sociales
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/clientes/listar/{{tenant_id}}?excluir_tipo_cliente_codi=OBR`
**Resultado esperado:** 200 OK. El array no incluye ningún registro con `tipo_cliente_codigo = "OBR"`.

### CLI-010 — Endpoint liviano: solo Obras Sociales
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/clientes/listar/{{tenant_id}}?tipo_cliente_codi=OBR`
**Resultado esperado:** 200 OK. El array incluye **únicamente** registros con `tipo_cliente_codigo = "OBR"`
(ej. "OSDE").

### CLI-011 — Endpoint "con saldo": usado por la grilla de Clientes
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/clientes/listar-con-saldo/{{tenant_id}}?excluir_tipo_cliente_codi=OBR`
**Resultado esperado:** 200 OK. Cada elemento del array incluye el campo `saldo_total`, y ninguno tiene
`tipo_cliente_codigo = "OBR"`.

### CLI-012 — Endpoint "con saldo": filtrado a Obras Sociales
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/clientes/listar-con-saldo/{{tenant_id}}?tipo_cliente_codi=OBR`
**Resultado esperado:** 200 OK. Solo trae Obras Sociales, cada una con su `saldo_total` calculado (por
ejemplo, reflejando el copago pendiente de una obra social facturada dual, si se ejecutó
[TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md)).

### CLI-013 — El endpoint viejo de filtro por tipo fue dado de baja
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/clientes/listarXTipoCliente/{{tenant_id}}/OBR`
**Resultado esperado:** 404 Not Found — esta ruta se eliminó del backend; el filtro por tipo ahora se hace
con el query param `tipo_cliente_codi` sobre `/clientes/listar/:id` (CLI-010).

### CLI-014 — Ambas variantes conviven sin pisarse (caché por combinación de filtros)
**Ambiente:** App
**Precondición:** Al menos un cliente común y una Obra Social cargados.
**Pasos:**
1. Abrir **Caja/Shop → Clientes** (usa "con saldo", excluye OBR) y verificar que se ve bien.
2. Sin recargar la página, ir a **Caja/Shop → Nuevo Movimiento** y abrir el selector de cliente (usa
   liviano, excluye OBR).
3. Volver a **Servicios → Obras Sociales** (usa "con saldo", solo OBR).
4. Volver a **Caja/Shop → Clientes**.
**Resultado esperado:** En cada pantalla se ve el conjunto de datos correcto para esa pantalla (ningún
listado "hereda" por error los datos de la pantalla anterior), y la última vuelta a Clientes sigue sin
mostrar Obras Sociales.
