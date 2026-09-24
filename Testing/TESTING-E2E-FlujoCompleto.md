# Plan de Pruebas End-to-End — Ciclo completo de un paciente (SPEC011+012+016, SPEC015, SPEC014)

**Ver primero:** [README.md](./README.md) (ambiente, credenciales, formato de caso de prueba)

---

## Objetivo de este documento

Los 3 documentos anteriores prueban cada funcionalidad **por separado**. Acá se arma **un solo recorrido**
que atraviesa las 3 (Venta de Servicios, Clientes, Dashboard), simulando la atención real de 2 pacientes
distintos, y verificando que **los mismos números aparezcan de forma consistente** en todas las pantallas
que los muestran (la grilla de Clientes, el módulo de Cobranzas, y las cards del Dashboard).

Esto es lo más parecido a lo que un usuario real de la clínica hace en su día a día, y sirve para detectar
bugs de integración que un caso aislado, por pantalla, no detecta (ej.: se factura bien, pero el número no
aparece después en el Dashboard).

> Ejecutar este documento **después** de haber corrido, al menos parcialmente,
> [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md) (para tener cargados Rubros, Tipos,
> Servicios, Obras Sociales, Planes y Convenios). Se recomienda usar pacientes **nuevos** (no los mismos
> ya facturados en ese documento) para poder seguir números "limpios" de punta a punta.

---

## E2E-001 — Paciente con Obra Social: de la facturación dual al Dashboard

**Ambiente:** App (con 2 verificaciones cruzadas por Postman, marcadas explícitamente)

**Precondición:**
- Rubro `Prácticas Dermatológicas` (`PRAC`, Admite Convenios = Sí) con el Tipo `DERM` y el Servicio
  `Consulta Dermatológica` ya cargados (VDS-002/005/008 de
  [TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md)).
- Obra Social `OSDE` con Plan `210` y un Convenio vigente para `Consulta Dermatológica` con Copago
  Paciente `$25.000` / Copago Obra Social `$20.000` (VDS-010/013/018).

**Pasos:**

1. **Alta del paciente (pantalla Clientes).** Ir a **Caja/Shop → Clientes**. Crear un cliente nuevo:
   Apellido `Ramírez`, Nombre `Sofía`.
   **Checkpoint (SPEC015):** el nuevo paciente aparece de inmediato en la grilla de Clientes, con Saldo
   `$0`.

2. **Asignar la Obra Social al paciente.** En la misma grilla, click en el ícono 🏥 de "Ramírez, Sofía".
   Asignar Obra Social `OSDE`, Plan `OSDE Plan 210`, Número de Afiliado `55667788`. Guardar.
   **Checkpoint (SPEC011/012):** el modal confirma la asignación sin error.

3. **Facturar la prestación (pantalla Venta).** Ir a **Servicios → Venta**. Cliente: `Ramírez, Sofía`.
   Verificar que se detecta el badge "🏥 OSDE — OSDE Plan 210". Rubro: `Prácticas Dermatológicas` → Tipo:
   `DERM` → Servicio: `Consulta Dermatológica`.
   **Checkpoint (SPEC016):** se muestra "✅ Convenio vigente encontrado" con Copago paciente `$25.000`,
   Copago obra social `$20.000`, Total prestación `$45.000`.

4. Click en "Facturar".
   **Checkpoint:** el resultado informa **2 movimientos** generados (uno de "Ramírez, Sofía" y otro de
   "OSDE").

5. **Cobrar solo una parte del copago del paciente.** En la grilla de cobro que aparece, cargar `$15.000`
   en efectivo (de los `$25.000` totales) y confirmar el cobro.
   **Checkpoint:** "Total pagado" = `$15.000`, "Saldo pendiente" = `$10.000`.

6. **Verificar el saldo en la grilla de Clientes.** Ir a **Caja/Shop → Clientes** y ubicar a "Ramírez,
   Sofía".
   **Checkpoint (SPEC015):** el Saldo de "Ramírez, Sofía" muestra `$10.000` (el copago pendiente del
   paciente; el copago de la obra social **no** afecta este saldo, porque es un movimiento a nombre de
   OSDE, no de la paciente).

7. **Verificar la deuda de la Obra Social.** Ir a **Servicios → Obras Sociales** y ubicar a "OSDE".
   **Checkpoint:** el Saldo de "OSDE" incluye los `$20.000` del copago de la obra social de este
   movimiento (más cualquier otro pendiente anterior).

8. **Verificar en "Servicios → Cobranzas" (Obras Sociales).** Abrir esa pantalla y buscar sin elegir un
   cliente puntual.
   **Checkpoint (SPEC015):** aparece el movimiento de `$20.000` a nombre de "OSDE" pendiente de cobro; no
   aparece en cambio ningún movimiento de "Ramírez, Sofía" (los pacientes no se ven desde esta pantalla).

9. **Verificar en "Caja/Shop → Cobranzas" (clientes comunes).** Abrir esa pantalla y buscar el cliente
   "Ramírez, Sofía".
   **Checkpoint:** aparece el cobro parcial de `$15.000` ya registrado, con `$10.000` pendiente; "OSDE" no
   aparece en el selector de esta pantalla.

10. **Verificar el impacto en el Dashboard.** Ir al Dashboard (login como `test@i134.edu`/ADMIN), con el
    filtro de fechas incluyendo hoy.
    **Checkpoint (SPEC014):**
    - "Total Facturado" incluye los `$45.000` de esta prestación (paciente + obra social).
    - "Deuda de Clientes" incluye los `$10.000` pendientes de "Ramírez, Sofía".
    - "Deuda de Obras Sociales" incluye los `$20.000` pendientes de "OSDE".
    - "Cobranzas del Día" incluye los `$15.000` cobrados hoy.
    - "Pacientes - Prácticas" incluye a "Ramírez, Sofía" en el conteo de pacientes atendidos hoy por
      Prácticas.
    - Click en la card "Deuda de Clientes" → la grilla de detalle incluye a "Ramírez, Sofía" con `$10.000`.
    - Click en la card "Deuda de Obras Sociales" → la grilla de detalle incluye a "OSDE".

11. **(Postman) Verificación cruzada del listado de clientes con saldo.**
    `GET {{baseUrl}}/clientes/listar-con-saldo/{{tenant_id}}?excluir_tipo_cliente_codi=OBR`
    **Checkpoint:** el registro de "Ramírez, Sofía" trae `saldo_total = 10000` (o el valor acumulado
    correspondiente), coincidiendo con lo visto en la grilla de Clientes en el paso 6.

12. **(Postman) Verificación cruzada del resumen del Dashboard.**
    `GET {{baseUrl}}/dashboard/resumen?tenant_id={{tenant_id}}&fecha_desde=<hoy>&fecha_hasta=<hoy>`
    **Checkpoint:** `cobranzas_dia` incluye los `$15.000` cobrados en el paso 5, coincidiendo con lo visto
    en el paso 10.

**Resultado esperado global:** los mismos `$45.000` facturados, `$15.000` cobrados y `$30.000` pendientes
(`$10.000` del paciente + `$20.000` de la obra social) se reflejan de forma consistente en las 3
funcionalidades (Venta de Servicios, Clientes/Cobranzas y Dashboard), sin ningún número "perdido" ni
duplicado entre pantallas.

---

## E2E-002 — Paciente sin Obra Social: Tratamiento sin precio, corrección posterior y su efecto en el Dashboard

**Ambiente:** App (con una verificación por Postman)

**Precondición:** Rubro `Tratamientos Estéticos` (`TRAT`) con el Tipo `FACIAL` cargado (VDS-001/004 de
[TESTING-VentaDeServicios.md](./TESTING-VentaDeServicios.md)).

**Pasos:**

1. **Alta de un Servicio de Tratamiento sin precio.** Ir a **Servicios → Servicios**. Crear un nuevo
   Servicio: Código `DEPILA`, Descripción `Depilación Definitiva`, Tipo `FACIAL`. **No** cargarle precio.

2. **Alta del paciente.** En **Caja/Shop → Clientes**, crear: Apellido `Torres`, Nombre `Ana`. No
   asignarle obra social.
   **Checkpoint (SPEC015):** aparece en la grilla con Saldo `$0`.

3. **Intentar facturar sin precio.** Ir a **Servicios → Venta**. Cliente: `Torres, Ana`. Servicio:
   `Depilación Definitiva`.
   **Checkpoint (SPEC016):** se muestra "⚠️ Este Tratamiento no tiene precio cargado" con el botón "+
   Cargar precio del Tratamiento" (no hay bloqueo duro).

4. Sin cargar el precio, click directo en "Facturar" y confirmar "Facturar sin precio" en el diálogo.
   **Checkpoint:** el resultado muestra "⚠️ Guardado sin precio — pendiente de corrección por un
   supervisor". Se generó 1 movimiento con precio $0.

5. **Verificar que no afecta la deuda del paciente incorrectamente.** Ir a **Caja/Shop → Clientes** y
   verificar el Saldo de "Torres, Ana".
   **Checkpoint:** el saldo es `$0` (el movimiento se grabó con precio $0, no con un importe fantasma).

6. **Localizar el movimiento en el ABM de corrección.** Ir a **Servicios → Movimientos sin Precio**.
   **Checkpoint (SPEC016):** aparece una fila con Artículo `Depilación Definitiva`, Motivo `Sin precio de
   convenio/tratamiento` (o similar), sin acciones de "Editar"/"Eliminar".

7. **Cargar el precio real del Tratamiento.** Ir a **Servicios → Servicios**, ícono 💰 en la fila de
   "Depilación Definitiva". Cargar Precio de Venta `$60.000`. Guardar.

8. **Corregir en lote el movimiento pendiente.** Volver a **Servicios → Movimientos sin Precio**. Click en
   "💰 Actualizar precio de movimiento" en la fila de "Depilación Definitiva".
   **Checkpoint:** la fila desaparece de la grilla de pendientes.

9. **Verificar el saldo corregido del paciente.** Ir a **Caja/Shop → Clientes** y revisar el Saldo de
   "Torres, Ana".
   **Checkpoint (SPEC015):** ahora muestra `$60.000` pendiente (el precio corregido pasó a formar parte de
   la deuda real del paciente).

10. **Verificar el impacto en el Dashboard.** Ir al Dashboard, con el filtro de fecha incluyendo hoy.
    **Checkpoint (SPEC014):**
    - "Deuda de Clientes" incluye ahora los `$60.000` de "Torres, Ana" (antes de la corrección, este
      importe no estaba reflejado en ningún lado, porque el movimiento tenía precio $0).
    - "Total Facturado" refleja el precio corregido, no $0.

11. **(Postman) Confirmar que ya no queda ningún pendiente para ese artículo.**
    `GET {{baseUrl}}/movimientos-deta/tenant/{{tenant_id}}?error_codi=SIN_PRECIO`
    **Checkpoint:** la respuesta no incluye ningún registro de `arti_codi = "DEPILA"`.

**Resultado esperado global:** un movimiento guardado deliberadamente sin precio no genera una deuda
incorrecta ni un número fantasma en el Dashboard mientras está pendiente de corrección ($0 en todos
lados), y una vez corregido, el precio real se refleja de forma consistente en la deuda del paciente
(Clientes) y en los totales de facturación (Dashboard).

---

## 📊 Tabla resumen de puntos de control cruzados

| Dato | Dónde se genera | Dónde se debe volver a ver igual |
|---|---|---|
| Total facturado de una prestación | Venta de Servicios (facturar) | Dashboard → "Total Facturado" |
| Copago pendiente del paciente | Venta de Servicios (cobro parcial/pendiente) | Clientes (columna Saldo) y Caja/Shop → Cobranzas |
| Copago pendiente de la obra social | Venta de Servicios (facturación dual) | Obras Sociales (columna Saldo) y Servicios → Cobranzas |
| Corrección de un movimiento sin precio | Movimientos sin Precio (corrección masiva) | Clientes (columna Saldo) y Dashboard → "Deuda de Clientes" / "Total Facturado" |
| Cobro registrado hoy | Venta de Servicios / Cobranzas | Dashboard → "Cobranzas del Día" |
