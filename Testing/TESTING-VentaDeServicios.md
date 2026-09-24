# Plan de Pruebas — Venta de Servicios (SPEC011, SPEC012, SPEC016)

**Ver primero:** [README.md](./README.md) (ambiente, credenciales, formato de caso de prueba)

---

## 1. Introducción funcional — ¿qué problema resuelve esta funcionalidad?

AppGastos nació para vender **artículos físicos con stock** (un kiosco/shop: se vende algo, baja el stock,
se cobra). Un Centro Médico como "Centro Médico Lincoln" necesita algo distinto:

- Vender **servicios/prestaciones** (una consulta, una práctica, un tratamiento estético) que **no tienen
  stock** — no se "descuentan" de ningún depósito.
- Muchos pacientes tienen una **obra social o prepaga**, que cubre parte del costo de la prestación. En
  ese caso hay que cobrarle una parte al paciente (el "copago") y dejar registrada por separado la parte
  que le corresponde cobrar a la obra social (que se cobra después, en otro proceso).
- No todos los servicios se cubren igual: algunos servicios (ej. una limpieza facial) tienen **un único
  precio de lista**, sin importar si el paciente tiene obra social o no. Otros servicios (ej. una práctica
  dermatológica) **dependen de un convenio** firmado entre la clínica y cada obra social/plan, con un
  copago distinto según el plan del paciente.
- Puede pasar que se atienda a un paciente sin tener todavía el precio o el convenio cargado en el
  sistema. Antes, el sistema **no dejaba facturar** en ese caso (bloqueaba la operación). Ahora permite
  facturar igual, pero deja la prestación marcada para que un supervisor la corrija después.

## 2. Explicación funcional de la solución implementada

- **Jerarquía de 3 niveles:** `Rubro de Artículo` → `Tipo de Artículo` → `Artículo/Servicio`. El campo
  clave es `Administra Stock` (Sí/No), que se define **una sola vez, a nivel de Rubro**, y se hereda a
  todos los Tipos y Artículos que cuelgan de él. Por eso, en toda la aplicación, un mismo registro físico
  (tabla `articulos`) puede verse en la pantalla **"Artículos"** (si su rubro administra stock) o en la
  pantalla **"Servicios"** (si su rubro no administra stock) — nunca en ambas a la vez.
- **Dentro de los servicios, otro campo del Rubro los separa en 2 grupos con reglas de precio distintas:**
  - **Tratamientos** (`Admite Convenios = No`): tienen **un único precio de lista** (igual para todos los
    pacientes, tengan o no obra social), cargado en el propio ABM de Servicios con el botón 💰.
  - **Prácticas** (`Admite Convenios = Sí`): el precio **depende siempre de un Convenio** entre esa
    práctica y una Obra Social + Plan puntual. Si el paciente no tiene una obra social real cubriéndolo,
    el sistema busca un convenio cargado contra una obra social "comodín" de nombre/sigla **`PART`**
    (Particular) — así, hasta el precio "de mostrador" de una Práctica sale siempre de un Convenio, nunca
    de un precio de lista suelto.
- **Facturación simple vs. dual:** al facturar una Práctica con convenio real, el sistema genera **2
  movimientos**: uno a nombre del paciente (por el copago del paciente) y otro a nombre de la obra social
  (por el copago que le corresponde pagar a ella, que queda pendiente para cobrarse después). Si no hay
  obra social o es un Tratamiento, genera **1 solo movimiento**, al paciente.
- **Guardado sin precio (SPEC016):** si al momento de facturar no hay precio de Tratamiento ni convenio de
  Práctica disponible, el sistema pregunta si se quiere continuar igual. Si se confirma, graba el
  movimiento con precio $0 y lo marca con un motivo de error, visible en la nueva pantalla **"Movimientos
  sin Precio"**, desde donde luego se puede corregir el precio en forma masiva (solo para Tratamientos).
- **Menús involucrados:** todo lo nuevo vive bajo el menú **Servicios** (Venta, Servicios, Obras Sociales,
  Planes de Obras Sociales, Convenios de Servicios, Movimientos sin Precio, Cobranzas), y una entrada
  nueva **Rubros de Artículos** dentro de Configuraciones. La pantalla de Artículos con stock (menú
  Caja/Shop) no cambia.

---

## 3. Bloque 1 — Alta de datos maestros (masa de datos de prueba)

Estos casos **son en sí mismos casos de prueba** de los ABMs (alta, validaciones, unicidad), y además
generan los datos que se usan en el resto del documento. Ejecutarlos **en este orden**.

> Los nombres/códigos propuestos son sugeridos; se pueden adaptar, pero manteniendo la relación
> Rubro→Tipo→Servicio y respetando que la Obra Social "comodín" se llame exactamente **`PART`** en el
> campo **Nombre/Sigla** (no en Razón Social), porque el sistema la busca por ese valor exacto.

### VDS-001 — Alta de Rubro de Artículo "Tratamientos"
**Ambiente:** App
**Precondición:** Login como `test@i134.edu` en Centro Médico Lincoln. Menú **Configuraciones → Rubros de
Artículos**.
**Pasos:**
1. Click en "➕ Nuevo Rubro de Artículo".
2. Código: `TRAT`. Descripción: `Tratamientos Estéticos`.
3. Administra Stock: **No (servicios)**. Admite Convenios: **No**.
4. Guardar.
**Resultado esperado:** El rubro se crea y aparece en la grilla con "Administra Stock: ❌ No" y "Admite
Convenios: ❌ No".

### VDS-002 — Alta de Rubro de Artículo "Prácticas"
**Ambiente:** App
**Pasos:**
1. Nuevo Rubro: Código `PRAC`, Descripción `Prácticas Dermatológicas`.
2. Administra Stock: **No**. Admite Convenios: **Sí**.
3. Guardar.
**Resultado esperado:** El rubro se crea con "Administra Stock: ❌ No" y "Admite Convenios: ✅ Sí".

### VDS-003 — No permitir código de Rubro duplicado
**Ambiente:** App
**Precondición:** VDS-001 ejecutado.
**Pasos:**
1. Intentar crear otro Rubro con código `TRAT` (mismo código, cualquier descripción).
2. Guardar.
**Resultado esperado:** El sistema rechaza el alta y muestra un mensaje de error indicando que el código ya
existe. No se crea un segundo registro.

### VDS-004 — Alta de Tipo de Artículo bajo el Rubro "Tratamientos"
**Ambiente:** App
**Precondición:** VDS-001. Menú **Configuraciones → Tipos de Artículos**.
**Pasos:**
1. Nuevo Tipo de Artículo. Rubro: `TRAT - Tratamientos Estéticos`. Código: `FACIAL`. Descripción:
   `Tratamientos Faciales`.
2. Guardar.
**Resultado esperado:** Se crea el Tipo y la grilla muestra la columna "Rubro" con el valor
`Tratamientos Estéticos`.

### VDS-005 — Alta de Tipo de Artículo bajo el Rubro "Prácticas"
**Ambiente:** App
**Precondición:** VDS-002.
**Pasos:**
1. Nuevo Tipo de Artículo. Rubro: `PRAC - Prácticas Dermatológicas`. Código: `DERM`. Descripción:
   `Prácticas Dermatológicas`.
2. Guardar.
**Resultado esperado:** Se crea el Tipo, asociado al rubro `PRAC`.

### VDS-006 — El formulario de Tipo de Artículo exige elegir un Rubro
**Ambiente:** App
**Pasos:**
1. Nuevo Tipo de Artículo. Dejar el campo "Rubro" sin seleccionar. Completar Código y Descripción.
2. Intentar guardar.
**Resultado esperado:** El formulario no permite guardar y marca "Rubro" como campo obligatorio.

### VDS-007 — Alta de un Servicio de tipo Tratamiento
**Ambiente:** App
**Precondición:** VDS-004. Menú **Servicios → Servicios**.
**Pasos:**
1. Nuevo Servicio. Código: `LIMPFAC`. Descripción: `Limpieza Facial`.
2. Tipo de Servicio: `FACIAL - Tratamientos Faciales` (notar que el desplegable **solo** ofrece tipos de
   rubros sin stock).
3. Guardar.
**Resultado esperado:** El servicio se crea y aparece en la grilla de "Servicios" con Tipo = `Tratamientos
Faciales`.

### VDS-008 — Alta de un Servicio de tipo Práctica
**Ambiente:** App
**Precondición:** VDS-005.
**Pasos:**
1. Nuevo Servicio. Código: `CONSDER`. Descripción: `Consulta Dermatológica`.
2. Tipo de Servicio: `DERM - Prácticas Dermatológicas`.
3. Guardar.
**Resultado esperado:** El servicio se crea correctamente.

### VDS-009 — Cargar el precio de lista del Tratamiento
**Ambiente:** App
**Precondición:** VDS-007. En la grilla de "Servicios", ubicar la fila "Limpieza Facial".
**Pasos:**
1. Click en el ícono 💰 ("Cambiar precio") de la fila.
2. Precio de Venta: `50000`. Fecha de Vigencia: hoy.
3. Guardar.
**Resultado esperado:** El modal se cierra sin error y, si la grilla muestra columna de precio, refleja
el valor cargado (o al menos no da error al recargar).

### VDS-010 — Alta de Obra Social real (OSDE)
**Ambiente:** App
**Precondición:** Menú **Servicios → Obras Sociales**.
**Pasos:**
1. Nueva Obra Social. Razón Social: `OSDE`. Nombre/Sigla: `OSDE`.
2. Guardar.
**Resultado esperado:** Se crea y aparece en la grilla de Obras Sociales, con columna "Saldo" en $0 (o
vacío, sin movimientos todavía).

### VDS-011 — Alta de la Obra Social "comodín" Particular
**Ambiente:** App
**Pasos:**
1. Nueva Obra Social. Razón Social: `Atención Particular`. Nombre/Sigla: **`PART`** (exacto, en mayúsculas).
2. Guardar.
**Resultado esperado:** Se crea correctamente. Esta obra social es la que el sistema usa como respaldo de
precio de lista para Prácticas sin cobertura real (ver Bloque 7).

### VDS-012 — Una Obra Social creada acá también aparece en Clientes
**Ambiente:** App
**Precondición:** VDS-010. Menú **Caja/Shop → Clientes**.
**Pasos:**
1. Abrir la grilla de Clientes y buscar "OSDE".
**Resultado esperado:** "OSDE" **no aparece** en la grilla de Clientes (el bug de SPEC015 hizo que se
excluyan las Obras Sociales de esa grilla — ver [TESTING-Clientes.md](./TESTING-Clientes.md)), aunque es
el mismo registro por debajo. Este caso confirma que Clientes y Obras Sociales son la misma tabla con
vistas distintas.

### VDS-013 — Alta de Plan para OSDE
**Ambiente:** App
**Precondición:** VDS-010. Menú **Servicios → Planes de Obras Sociales**.
**Pasos:**
1. Nuevo Plan. Obra Social: `OSDE`. Código: `210`. Descripción: `OSDE Plan 210`.
2. Guardar.
**Resultado esperado:** Se crea el plan asociado a OSDE.

### VDS-014 — Alta de Plan para la Obra Social Particular
**Ambiente:** App
**Precondición:** VDS-011.
**Pasos:**
1. Nuevo Plan. Obra Social: `Atención Particular`. Código: `PART`. Descripción: `Precio Particular`.
2. Guardar.
**Resultado esperado:** Se crea el plan.

### VDS-015 — No permitir código de Plan duplicado para la misma Obra Social
**Ambiente:** App
**Precondición:** VDS-013.
**Pasos:**
1. Nuevo Plan. Obra Social: `OSDE`. Código: `210` (repetido). Descripción: cualquiera.
2. Guardar.
**Resultado esperado:** Rechazado con mensaje de error de código duplicado para esa Obra Social.

### VDS-016 — Alta de Cliente/paciente CON obra social
**Ambiente:** App
**Precondición:** Menú **Caja/Shop → Clientes**.
**Pasos:**
1. Nuevo Cliente. Apellido: `Pérez`. Nombre: `Juan`. Tipo de Cliente: cualquiera que no sea Obra Social.
2. Guardar.
3. En la fila de "Juan Pérez", click en el ícono 🏥 ("Asignar Obra Social").
4. Obra Social: `OSDE`. Plan: `OSDE Plan 210` (el desplegable de Plan debe habilitarse recién al elegir
   Obra Social). Número de Afiliado: `12345678`.
5. Guardar.
**Resultado esperado:** El cliente "Juan Pérez" queda con la Obra Social OSDE Plan 210 asignada, visible
al reabrir el mismo modal.

### VDS-017 — Alta de Cliente/paciente SIN obra social
**Ambiente:** App
**Pasos:**
1. Nuevo Cliente. Apellido: `López`. Nombre: `María`.
2. Guardar. No asignarle ninguna obra social.
**Resultado esperado:** El cliente se crea correctamente, sin obra social asociada.

### VDS-018 — Alta de Convenio para la Práctica con OSDE
**Ambiente:** App
**Precondición:** VDS-008, VDS-013. Menú **Servicios → Convenios de Servicios**.
**Pasos:**
1. Nuevo Convenio. Artículo/Servicio: `CONSDER - Consulta Dermatológica` (el desplegable solo debe listar
   servicios de rubros con "Admite Convenios = Sí").
2. Obra Social: `OSDE`. Plan: `OSDE Plan 210` (se habilita recién tras elegir Obra Social).
3. Copago Paciente: `25000`. Copago Obra Social: `20000`.
4. Vigencia Desde: hoy. Vigencia Hasta: en blanco (sin vencimiento).
5. Guardar.
**Resultado esperado:** El convenio se crea y aparece en la grilla con Artículo, Obra Social, Plan y ambos
copagos.

### VDS-019 — Alta de Convenio "Particular" para la misma Práctica (precio de mostrador)
**Ambiente:** App
**Precondición:** VDS-011, VDS-014.
**Pasos:**
1. Nuevo Convenio. Artículo/Servicio: `CONSDER - Consulta Dermatológica`.
2. Obra Social: `Atención Particular`. Plan: `Precio Particular`.
3. Copago Paciente: `40000`. Copago Obra Social: `0`.
4. Vigencia Desde: hoy.
5. Guardar.
**Resultado esperado:** El convenio se crea. Este es el precio que va a pagar un paciente **sin** obra
social real cuando se le factura esta Práctica (ver VDS-039).

### VDS-020 — No permitir superposición de vigencias del mismo Convenio
**Ambiente:** App
**Precondición:** VDS-018.
**Pasos:**
1. Nuevo Convenio con el mismo Artículo + Obra Social + Plan que VDS-018, con Vigencia Desde de hoy
   también (o una fecha dentro del rango ya vigente).
2. Guardar.
**Resultado esperado:** Rechazado con un mensaje de error de superposición de vigencias.

---

## 4. Bloque 2 — Contraste: ¿dónde se ven los Artículos y dónde los Servicios?

> Objetivo: demostrar que **son el mismo tipo de registro** (tabla `articulos`) mostrado en dos pantallas
> distintas según el Rubro, y que **no se mezclan nunca** entre sí.

### VDS-021 — Alta de un Artículo con stock (contraste)
**Ambiente:** App
**Precondición:** Verificar en **Configuraciones → Rubros de Artículos** que existe el rubro `ART`
(seedeado por defecto, Administra Stock = Sí). Si no existe, crearlo como en VDS-001 pero con
Administra Stock = **Sí**. Luego crear un Tipo de Artículo bajo ese rubro (ej. `INSUM - Insumos`), menú
**Configuraciones → Tipos de Artículos**.
**Pasos:**
1. Menú **Caja/Shop → Artículos**. Nuevo Artículo. Código: `GUANTE100`. Descripción: `Guantes descartables
   x100`. Tipo de Artículo: `INSUM - Insumos`.
2. Guardar.
**Resultado esperado:** El artículo se crea y aparece en la grilla de **"Artículos"**, con columnas Precio
Compra / Precio Venta.

### VDS-022 — El Artículo con stock NO aparece en la pantalla de Servicios
**Ambiente:** App
**Precondición:** VDS-021.
**Pasos:**
1. Ir a **Servicios → Servicios**.
2. Buscar "Guantes descartables" en la grilla.
**Resultado esperado:** No aparece. La grilla de Servicios solo muestra artículos cuyo Rubro tiene
"Administra Stock = No".

### VDS-023 — Los Servicios NO aparecen en la pantalla de Artículos
**Ambiente:** App
**Precondición:** VDS-007, VDS-008.
**Pasos:**
1. Ir a **Caja/Shop → Artículos**.
2. Buscar "Limpieza Facial" y "Consulta Dermatológica" en la grilla.
**Resultado esperado:** Ninguno de los dos aparece. La grilla de Artículos solo muestra artículos de
rubros con "Administra Stock = Sí".

### VDS-024 — El desplegable de "Tipo de Artículo" en el alta de un Artículo con stock solo ofrece
tipos que administran stock
**Ambiente:** App
**Pasos:**
1. En **Caja/Shop → Artículos**, click en "Nuevo Artículo".
2. Abrir el desplegable "Tipo de Artículo".
**Resultado esperado:** Solo aparecen tipos de rubros con Administra Stock = Sí (ej. `Insumos`); no
aparecen `Tratamientos Faciales` ni `Prácticas Dermatológicas`.

### VDS-025 — El desplegable de "Tipo de Servicio" en el alta de un Servicio solo ofrece tipos que NO
administran stock
**Ambiente:** App
**Pasos:**
1. En **Servicios → Servicios**, click en "Nuevo Servicio".
2. Abrir el desplegable "Tipo de Servicio".
**Resultado esperado:** Solo aparecen `Tratamientos Faciales` y `Prácticas Dermatológicas`; no aparece
`Insumos`.

### VDS-026 (Postman) — El endpoint de Artículos con stock nunca devuelve servicios
**Ambiente:** Postman
**Precondición:** Token válido (ver README).
**Pasos:**
1. `GET {{baseUrl}}/articulos/listar/{{tenant_id}}` con header `Authorization: Bearer {{token}}`.
**Resultado esperado:** 200 OK. El array de respuesta incluye `GUANTE100` pero **no** incluye
`LIMPFAC` ni `CONSDER`.

### VDS-027 (Postman) — El endpoint de Servicios nunca devuelve artículos con stock
**Ambiente:** Postman
**Pasos:**
1. `GET {{baseUrl}}/articulos/servicios/{{tenant_id}}` con header `Authorization: Bearer {{token}}`.
**Resultado esperado:** 200 OK. El array incluye `LIMPFAC` y `CONSDER`, pero **no** incluye `GUANTE100`.
Cada elemento trae `administra_stock` y `admite_convenios` (como valor tipo Buffer/BIT, ej.
`{"type":"Buffer","data":[0]}`).

---

## 5. Bloque 3 — Contraste: Tratamientos vs Prácticas

> Objetivo: mostrar que, dentro de "Servicios", hay 2 sub-grupos con reglas de precio distintas, aunque
> ambos se vean en la misma grilla de "Servicios".

### VDS-028 — El Rubro es el único lugar donde se define Tratamiento vs Práctica
**Ambiente:** App
**Precondición:** VDS-001, VDS-002.
**Pasos:**
1. Ir a **Configuraciones → Rubros de Artículos**.
2. Comparar la fila `TRAT` (Tratamientos Estéticos) contra la fila `PRAC` (Prácticas Dermatológicas).
**Resultado esperado:** Ambas tienen "Administra Stock: ❌ No" (por eso ambas caen en "Servicios"), pero
difieren únicamente en "Admite Convenios": `TRAT` = No (Tratamiento), `PRAC` = Sí (Práctica).

### VDS-029 — Un Tratamiento sin convenio igual tiene precio (precio de lista)
**Ambiente:** App
**Precondición:** VDS-007, VDS-009. Menú **Servicios → Venta**.
**Pasos:**
1. Seleccionar Cliente: `María López` (sin obra social).
2. Rubro de Artículos: `Tratamientos Estéticos`. Tipo: `Tratamientos Faciales`. Servicio: `Limpieza
   Facial`.
**Resultado esperado:** Se muestra el bloque "💊 Precio de Tratamiento" con el precio de lista cargado en
VDS-009 (`$50.000`), **sin** preguntar nada sobre obra social ni convenio.

### VDS-030 — Una Práctica, en cambio, siempre pregunta por la cobertura del paciente
**Ambiente:** App
**Precondición:** VDS-008. En la misma pantalla de Venta.
**Pasos:**
1. Seleccionar Cliente: `Juan Pérez` (con OSDE).
2. Rubro: `Prácticas Dermatológicas`. Tipo: `Prácticas Dermatológicas`. Servicio: `Consulta
   Dermatológica`.
**Resultado esperado:** El sistema busca automáticamente un convenio vigente para la obra social/plan del
paciente y muestra el desglose de copagos (ver VDS-035), a diferencia del Tratamiento que solo mostró un
precio fijo.

### VDS-031 — Un Tratamiento nunca genera facturación dual, aunque el paciente tenga obra social
**Ambiente:** App
**Precondición:** VDS-007, VDS-009.
**Pasos:**
1. Seleccionar Cliente: `Juan Pérez` (con OSDE).
2. Facturar el Servicio "Limpieza Facial" (Tratamiento).
3. Completar la facturación (ver Bloque 4).
**Resultado esperado:** Se genera **un solo movimiento** (al paciente) por el precio de lista, sin importar
que el cliente tenga OSDE asignada. Los Tratamientos nunca generan un segundo movimiento a la obra social.

---

## 6. Bloque 4 — Facturación de Servicios: Tratamientos

### VDS-032 — Facturar un Tratamiento con precio cargado
**Ambiente:** App
**Precondición:** VDS-007, VDS-009. Menú **Servicios → Venta**.
**Pasos:**
1. Cliente: `María López`. Rubro: `Tratamientos Estéticos` → Tipo: `Tratamientos Faciales` → Servicio:
   `Limpieza Facial`.
2. Verificar que se muestra "Precio: $50.000" en modo solo lectura (no hay ningún campo para tipear el
   precio a mano).
3. Click en "Facturar".
**Resultado esperado:** El sistema muestra el resultado de la facturación con un número de movimiento
generado, sin ningún aviso de error/pendiente.

### VDS-033 — Tratamiento sin precio cargado: alta de precio "al vuelo"
**Ambiente:** App
**Precondición:** Crear un nuevo Servicio de tipo Tratamiento SIN cargarle precio (repetir VDS-007 con
otro código, ej. `PEELING` / `Peeling Químico`, sin ejecutar VDS-009 para este).
**Pasos:**
1. En Venta, elegir Cliente y Servicio: `Peeling Químico`.
2. Verificar que aparece el aviso "⚠️ Este Tratamiento no tiene precio cargado" con el botón "+ Cargar
   precio del Tratamiento".
3. Click en el botón, cargar un precio de venta (ej. `35000`) y confirmar.
**Resultado esperado:** El modal se cierra y la pantalla de Venta pasa a mostrar el precio recién cargado
en modo lectura, sin recargar la página. Ahora se puede facturar con ese precio.

### VDS-034 — Tratamiento sin precio: declinar el alta y facturar igual con error
**Ambiente:** App
**Precondición:** Otro Servicio de tipo Tratamiento sin precio cargado (ej. `MASAJE` / `Masaje
Descontracturante`).
**Pasos:**
1. En Venta, elegir Cliente y Servicio `Masaje Descontracturante`.
2. **No** cargar precio: click directamente en "Facturar".
3. Confirmar en el diálogo "¿Desea cargar la prestación sin precio de todas formas?".
**Resultado esperado:** La facturación se completa igual, y el resultado muestra el aviso "⚠️ Guardado sin
precio — pendiente de corrección por un supervisor". El movimiento queda con precio $0 (ver Bloque 8 para
verificarlo en el ABM de corrección).

---

## 7. Bloque 5 — Facturación de Servicios: Prácticas

### VDS-035 — Práctica con convenio vigente: facturación dual
**Ambiente:** App
**Precondición:** VDS-016 (Juan Pérez con OSDE Plan 210), VDS-018 (convenio OSDE para Consulta
Dermatológica).
**Pasos:**
1. En Venta, Cliente: `Juan Pérez`. Verificar que se muestra el badge "🏥 OSDE — OSDE Plan 210" junto al
   selector de cliente.
2. Servicio: `Consulta Dermatológica`.
3. Verificar el bloque "✅ Convenio vigente encontrado" con Copago paciente `$25.000`, Copago obra social
   `$20.000`, Total prestación `$45.000`.
4. Facturar.
**Resultado esperado:** El resultado de la facturación informa **2 movimientos** generados: uno del
paciente (Juan Pérez) y otro de la obra social (OSDE), coincidiendo con los importes del convenio.

### VDS-036 — Práctica sin convenio para la obra social del paciente
**Ambiente:** App
**Precondición:** Un paciente con una obra social **sin** convenio cargado para esta práctica (ej. crear
una tercera Obra Social "Swiss Medical" sin ningún convenio, y asignarla a un cliente nuevo "Gómez,
Carlos").
**Pasos:**
1. En Venta, Cliente: `Gómez, Carlos` (con Swiss Medical, sin convenio). Servicio: `Consulta
   Dermatológica`.
**Resultado esperado:** No se detecta convenio real. Si existe el convenio "Particular" (VDS-019), el
sistema debería mostrar el precio de lista particular (`$40.000`) en vez de bloquear — ver VDS-039. Si no
existe convenio Particular, debe mostrar el bloqueo "⚠️ No hay convenio ni precio de lista... Puede
continuar sin precio".

### VDS-037 — Práctica sin obra social asignada al paciente
**Ambiente:** App
**Precondición:** VDS-017 (María López, sin obra social), VDS-019 (convenio Particular cargado).
**Pasos:**
1. En Venta, Cliente: `María López`. Verificar que se muestra "⚪ Sin obra social".
2. Servicio: `Consulta Dermatológica`.
**Resultado esperado:** El sistema encuentra el convenio de la obra social "Particular" (`PART`) y muestra
"💲 Precio de lista — Paciente sin obra social (Particular)" con el precio `$40.000` cargado en VDS-019 —
demuestra que hasta el precio "de mostrador" de una Práctica sale de un Convenio, nunca de un precio suelto
como los Tratamientos.

### VDS-038 — Facturar una Práctica "Particular" genera un solo movimiento
**Ambiente:** App
**Precondición:** VDS-037.
**Pasos:**
1. Continuar la facturación de VDS-037 y confirmar "Facturar".
**Resultado esperado:** Se genera **1 solo movimiento** (al paciente, por $40.000) — no hay obra social
real a la cual generarle un segundo movimiento, aunque el precio haya salido de un "convenio".

### VDS-039 — Asignar el "+ Asignar Obra Social" desde la propia pantalla de Venta
**Ambiente:** App
**Precondición:** Un cliente sin obra social recién creado (ej. "Torres, Ana").
**Pasos:**
1. En Venta, elegir Cliente: `Torres, Ana`.
2. Click en "+ Asignar Obra Social" (visible junto al badge "⚪ Sin obra social").
3. Asignar OSDE Plan 210, Número de Afiliado `99999999`. Guardar.
**Resultado esperado:** El modal se cierra y, sin recargar la pantalla, el badge de cobertura pasa a
mostrar "🏥 OSDE — OSDE Plan 210".

### VDS-040 — Práctica sin convenio real ni Particular: guardar con error
**Ambiente:** App
**Precondición:** Eliminar (o no crear) el convenio Particular para un servicio de Práctica nuevo (ej.
crear `CRIOTERAP` / `Crioterapia` sin ningún convenio cargado, ni real ni particular).
**Pasos:**
1. En Venta, Cliente: `María López`. Servicio: `Crioterapia`.
2. Verificar el bloqueo "⚠️ No hay convenio ni precio de lista... Puede continuar sin precio".
3. Click en "Facturar" y confirmar "Facturar sin precio".
**Resultado esperado:** Se graba el movimiento con precio $0 y motivo de error, igual que en VDS-034.

---

## 8. Bloque 6 — Cobro del copago del paciente

### VDS-041 — Cobrar el total del copago del paciente en efectivo
**Ambiente:** App
**Precondición:** VDS-035 recién facturada (queda habilitada la grilla de cobro).
**Pasos:**
1. En la grilla de cobro, marcar la forma de pago "Efectivo" con el importe total ($25.000, el copago del
   paciente).
2. Click en "Confirmar Cobro".
**Resultado esperado:** El cobro se registra sin error; "Total pagado" = $25.000, "Saldo pendiente" = $0.

### VDS-042 — Cobro parcial del copago del paciente
**Ambiente:** App
**Precondición:** Facturar de nuevo una Práctica con convenio (repetir VDS-035 con otro paciente/fecha).
**Pasos:**
1. En la grilla de cobro, cargar solo una parte del importe (ej. $10.000 de $25.000) con una forma de
   pago.
2. Confirmar Cobro.
**Resultado esperado:** "Total pagado" = $10.000, "Saldo pendiente" = $15.000. El movimiento del paciente
queda con saldo pendiente por la diferencia.

### VDS-043 — Dejar el cobro pendiente
**Ambiente:** App
**Precondición:** Facturar otra Práctica con convenio.
**Pasos:**
1. En vez de cargar una forma de pago, click en "Dejar pendiente".
**Resultado esperado:** No se registra ningún cobro; el movimiento del paciente queda con el saldo
completo pendiente (verificable luego en la grilla de Clientes o en Cobranzas).

### VDS-044 — El movimiento de la Obra Social nunca se cobra desde esta pantalla
**Ambiente:** App
**Precondición:** VDS-035 (facturación dual).
**Pasos:**
1. Revisar la grilla de cobro que aparece tras facturar.
**Resultado esperado:** La grilla de cobro **solo** ofrece cobrar el movimiento del paciente; no hay forma
de cobrar, desde esta pantalla, el movimiento generado a nombre de la obra social (queda para el módulo de
Cobranzas de Obras Sociales, ver Bloque 9).

---

## 9. Bloque 7 — Movimientos sin Precio (corrección posterior)

### VDS-045 — El movimiento guardado con error aparece en "Movimientos sin Precio"
**Ambiente:** App
**Precondición:** VDS-034 ejecutado (Tratamiento "Masaje Descontracturante" guardado sin precio).
**Pasos:**
1. Ir a **Servicios → Movimientos sin Precio**.
**Resultado esperado:** Aparece una fila con Artículo `Masaje Descontracturante`, Precio `-` (o $0), y
Motivo `Sin precio de convenio/tratamiento` (o similar). No hay botones de "Nuevo"/"Editar"/"Eliminar" en
esta pantalla.

### VDS-046 — La acción de corrección masiva solo aparece para Tratamientos
**Ambiente:** App
**Precondición:** VDS-045 y VDS-040 (una fila de Tratamiento y otra de Práctica, ambas en error).
**Pasos:**
1. Comparar la fila del Tratamiento "Masaje Descontracturante" contra la fila de la Práctica "Crioterapia"
   en la grilla de Movimientos sin Precio.
**Resultado esperado:** Solo la fila del Tratamiento muestra la acción "💰 Actualizar precio de
movimiento"; la fila de la Práctica se ve, pero **sin** esa acción (las Prácticas se corrigen dando de
alta el Convenio correspondiente, no desde acá).

### VDS-047 — Corregir en lote el precio de un Tratamiento en error
**Ambiente:** App
**Precondición:** VDS-046. Cargar primero el precio de lista de "Masaje Descontracturante" (como en
VDS-033, pero desde **Servicios → Servicios**, ícono 💰, en vez de desde la pantalla de Venta).
**Pasos:**
1. Volver a **Servicios → Movimientos sin Precio**.
2. Click en "💰 Actualizar precio de movimiento" en la fila de "Masaje Descontracturante".
**Resultado esperado:** La fila desaparece de la grilla de "Movimientos sin Precio" (quedó corregida con
el precio de lista vigente).

### VDS-048 (Postman) — Corrección masiva vía API
**Ambiente:** Postman
**Precondición:** Un `arti_id` de Tratamiento con movimientos en error y precio de lista ya cargado.
**Pasos:**
1. `GET {{baseUrl}}/movimientos-deta/tenant/{{tenant_id}}?error_codi=SIN_PRECIO` — anotar el `arti_id` de
   un Tratamiento en la respuesta.
2. `POST {{baseUrl}}/movimientos-deta/CorreccionXArtiID` con body:
   ```json
   { "arti_id": <el arti_id anotado>, "user_id": 1 }
   ```
**Resultado esperado:** 200 OK. Al repetir el `GET` del paso 1, ese `arti_id` ya no aparece en el
resultado (o aparece con `error_codi = null` si se listara sin filtro).

### VDS-049 (Postman) — Corrección masiva rechazada si el artículo no tiene precio
**Ambiente:** Postman
**Precondición:** Un `arti_id` de Tratamiento en error y **sin** precio de lista cargado (ej. "Crioterapia"
sin convenio ni precio — nota: este caso aplica solo si "Crioterapia" fuera un Tratamiento; si es una
Práctica, usar cualquier otro Tratamiento sin precio en `articulos_precios`).
**Pasos:**
1. `POST {{baseUrl}}/movimientos-deta/CorreccionXArtiID` con `arti_id` de un Tratamiento sin precio
   cargado.
**Resultado esperado:** Respuesta de error (no 200), con un mensaje similar a "El artículo no posee precio
para actualizar movimientos erróneos".

---

## 10. Bloque 8 — Cobranzas: Caja/Shop vs Servicios (Obras Sociales)

### VDS-050 — "Caja/Shop → Cobranzas" no ofrece Obras Sociales como cliente
**Ambiente:** App
**Precondición:** VDS-010 (OSDE creada).
**Pasos:**
1. Ir a **Caja/Shop → Cobranzas**. Verificar el título "🧾 Reporte de Cobranzas".
2. Abrir el selector de cliente y buscar "OSDE".
**Resultado esperado:** "OSDE" no aparece en el selector de cliente de esta pantalla.

### VDS-051 — "Servicios → Cobranzas" muestra únicamente Obras Sociales
**Ambiente:** App
**Precondición:** VDS-035 facturado (generó un movimiento a nombre de OSDE).
**Pasos:**
1. Ir a **Servicios → Cobranzas**. Verificar el título "🧾 Cobranzas de Obras Sociales".
2. Abrir el selector de cliente.
**Resultado esperado:** El selector **solo** ofrece Obras Sociales (ej. "OSDE"); no ofrece pacientes como
"Juan Pérez" o "María López".

### VDS-052 — El movimiento de la Obra Social aparece pendiente en "Servicios → Cobranzas"
**Ambiente:** App
**Precondición:** VDS-035 y VDS-044 (movimiento de OSDE nunca cobrado desde Venta).
**Pasos:**
1. En **Servicios → Cobranzas**, buscar sin elegir cliente puntual (o eligiendo "OSDE").
**Resultado esperado:** Aparece el movimiento generado por la facturación dual de VDS-035, con el importe
del copago de la obra social ($20.000) pendiente de cobro.

---

## 11. Bloque 9 — Pruebas adicionales desde Postman

### VDS-053 — Buscar convenio vigente por API
**Ambiente:** Postman
**Precondición:** VDS-018, y los `id` de artículo/obra social/plan involucrados (se pueden obtener listando
`/articulos/servicios/:tenant_id`, `/clientes/listar/:tenant_id?tipo_cliente_codi=OBR` y
`/planes-obras-sociales/listar/:obra_social_id`).
**Pasos:**
1. `GET {{baseUrl}}/convenios-articulos/buscar-vigente?arti_id=<id>&obra_social_id=<id>&plan_id=<id>`
**Resultado esperado:** 200 OK con `{"tiene_convenio": true, "convenio": {...}}` y los copagos cargados en
VDS-018.

### VDS-054 — Buscar convenio vigente para una combinación sin convenio
**Ambiente:** Postman
**Pasos:**
1. Repetir la request anterior con un `plan_id` que no tenga convenio cargado para ese artículo.
**Resultado esperado:** 404, con `{"tiene_convenio": false, "mensaje": "No existe convenio vigente"}`.

### VDS-055 — Facturar una Práctica con convenio, directo por API
**Ambiente:** Postman
**Precondición:** Cliente con obra social/plan con convenio vigente para un servicio de Práctica.
**Pasos:**
1. `POST {{baseUrl}}/facturacion-servicios` con body:
   ```json
   {
     "tenant_id": <tenant_id>,
     "tipo_movim_id": <id de un tipo de movimiento de venta>,
     "cliente_id": <id del paciente>,
     "fecha": "2026-09-23",
     "observ": "Prueba Postman",
     "arti_id": <id de la práctica>,
     "cant": 1,
     "precio_unit": null,
     "observ_deta": "Prueba",
     "user_id": 1
   }
   ```
**Resultado esperado:** 200/201 con `tipo_facturacion: "dual"`, y ambos `movim_id_paciente` /
`movim_id_obra_social` con valores numéricos (no `null`).

### VDS-056 — Guardar sin precio, directo por API
**Ambiente:** Postman
**Precondición:** Un `arti_id` de Tratamiento sin precio de lista cargado.
**Pasos:**
1. Repetir la request de VDS-055 apuntando a ese `arti_id`, con `"precio_unit": null`.
**Resultado esperado:** 200/201. La respuesta no debería fallar (no más `SIGNAL SQLSTATE '45000'`); el
movimiento queda grabado (verificable luego con `GET
/movimientos-deta/tenant/:tenant_id?error_codi=SIN_PRECIO`).

### VDS-057 — Intentar crear un Convenio para un artículo con stock (debe fallar)
**Ambiente:** Postman
**Precondición:** El `arti_id` de "Guantes descartables x100" (VDS-021, administra stock).
**Pasos:**
1. `POST {{baseUrl}}/facturacion-servicios` con `arti_id` de ese artículo con stock.
**Resultado esperado:** Error (no 200), ya que `facturacionServiciosGrabar` rechaza explícitamente
artículos que administran stock ("no admite artículos que administran stock").
