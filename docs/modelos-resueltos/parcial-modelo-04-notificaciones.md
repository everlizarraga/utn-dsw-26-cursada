# 📝 Parcial Modelo 04 — Notificaciones del Usuario

**Materia:** Desarrollo de Software — UTN FRBA · **Dominio:** turnos médicos (tu TP) · **Tiempo sugerido:** 1h 30min

---

## 🎯 Cómo usar

Leé la consigna → tapá lo que está bajo `⬇️ RESPUESTA` → respondé al hueso → destapá y compará. Mirá **🔎 qué busca el corrector**, **🟡 divague típico**, y el **💡 profundizar** opcional.

> **Core fijo:** endpoints (verbo + url + body/params + **auth**), servicios, errores, **test de integración**, **validación/guarda**, **persistencia (embebido vs colección)**, **accesibilidad**, **criterio**. Rota la funcionalidad.
>
> ⚠️ **Este parcial tiene una trampa conceptual a propósito:** no toda operación repetida es un 409. Prestá atención al punto 2.5 y al 4.1.

---

## 📋 Enunciado

> Los usuarios (pacientes y médicos) reciben **notificaciones** dentro de la plataforma (ej. "se reservó un turno", "se canceló un turno"). El equipo de producto pide que el usuario pueda: **ver sus notificaciones no leídas**, **ver las leídas**, y **marcar una notificación como leída**. La campana de la navbar debe mostrar un **contador de no leídas**.
>
> Resolvé los siguientes puntos teniendo en cuenta la solución del TP.

---
---

# 1 · UI / FRONTEND

## 1.1 — Croquis y apariencia de las pantallas.

<br><br>

### ⬇️ RESPUESTA 1.1

En la **navbar** (modificada), un ícono de **campana con un badge** que muestra la cantidad de no leídas. Al tocarlo, se abre un **panel/lista de notificaciones**: cada ítem muestra el mensaje, la fecha y un indicador de leída/no leída (ej. no leídas resaltadas con un punto). Se contemplan estados de **carga, vacío** ("No tenés notificaciones") **y error**. Opcionalmente, separación o filtro entre leídas y no leídas.

🔎 **Qué busca el corrector:** la campana con **contador**, el listado con distinción visual leída/no leída, y los estados de carga/vacío/error.

---

## 1.2 — Componentes nuevos y modificados.

<br><br>

### ⬇️ RESPUESTA 1.2

**Nuevos:** `NotificacionItem`, el panel/lista `Notificaciones`, un `BadgeContador`, y un `notificacionService` (axios). **Modificados:** la `Navbar` (suma la campana + badge). El estado de notificaciones puede vivir en un Context para que el contador se actualice en cualquier pantalla.

🔎 **Qué busca el corrector:** distinguir nuevos vs modificados; un componente reutilizable para el ítem; manejo de estado compartido para el contador.

---

## 1.3 — Interacción al (a) ver notificaciones y (b) marcar una como leída.

<br><br>

### ⬇️ RESPUESTA 1.3

**(a) Ver:** click en la campana → `GET` de no leídas → loader → render de la lista (o vacío). **(b) Marcar leída:** click en una notificación (o en su botón) → `PATCH` → al confirmar, el ítem pasa a estilo "leída" y el **badge baja en uno**. Si falla, se revierte el cambio visual.

🔎 **Qué busca el corrector:** que el contador refleje el cambio, conexión real con el back, y manejo de error (revertir).

---

## 1.4 — Aspectos de accesibilidad.

<br><br>

### ⬇️ RESPUESTA 1.4

El badge no puede comunicar **solo con un número de color**: lleva `aria-label` ("3 notificaciones sin leer"). Las notificaciones nuevas que aparezcan en vivo conviene anunciarlas con una **región `aria-live`** (lector de pantalla las lee sin que el usuario tenga que buscarlas). Lista semántica (`<ul>/<li>`), navegable por teclado, y el estado leída/no leída no depende **solo del color**. Primera regla de ARIA: HTML semántico nativo primero.

🔎 **Qué busca el corrector:** `aria-label` en el badge, idealmente **`aria-live`** para contenido dinámico (punto fino de esta feature), y no-solo-color + teclado.

---
---

# 2 · BACKEND

## 2.1 — Rutas REST y autenticación.

<br><br>

### ⬇️ RESPUESTA 2.1

> 🔴 Tip de oro: aclarar siempre la autenticación. Acá pega fuerte: las notificaciones son **de un usuario**, nadie debe ver las de otro.

Con `Authorization: Bearer <JWT>` (valida que el usuario sea el dueño):

| Verbo | URL | Para qué |
|---|---|---|
| `GET` | `/usuarios/:id/notificaciones?leida=false` | No leídas del usuario. Query `leida` (true/false) + paginación. → 200 |
| `GET` | `/usuarios/:id/notificaciones?leida=true` | Leídas del usuario. → 200 |
| `PATCH` | `/notificaciones/:id` | Marcar como leída. Body: `{ "leida": true }`. → 200 |

🔎 **Qué busca el corrector:** notificaciones como **sub-recurso del usuario** para el listado, **query param** `leida` para discriminar, `PATCH` (modificación parcial) para marcar, paginación, y la **auth** validando dueño.

🟡 **Divague típico:** crear `GET /noLeidas` y `GET /leidas` como rutas separadas con verbo/estado en la URL. **Respuesta justa:** un GET con `?leida=` filtra; `PATCH /notificaciones/:id` marca.

---

## 2.2 — Lógica de la capa de servicios.

<br><br>

### ⬇️ RESPUESTA 2.2

`NotificacionService.listar(usuarioId, { leida, page })`: valida usuario, trae del repo filtrando por `leida` y pagina. `NotificacionService.marcarLeida(notifId, usuarioId)`: valida que la notificación **exista** (404) y sea **del usuario** (403); pone `leida = true` y guarda. Para el contador, una consulta de **conteo** de no leídas.

🔎 **Qué busca el corrector:** filtrado por estado en el listado, validación de **pertenencia** (dueño) al marcar, lógica en el service.

---

## 2.3 — Gestión de errores.

<br><br>

### ⬇️ RESPUESTA 2.3

Notificación inexistente → **404**. Notificación de otro usuario → **403**. Token ausente/inválido → **401**. Body inválido → **400**. El service lanza tipados; el middleware mapea. **Nota:** marcar como leída una notificación **que ya estaba leída NO es un error** (ver 2.5).

🔎 **Qué busca el corrector:** **403** por pertenencia (no son tus notificaciones), y reconocer que "ya leída" **no** es 409.

---

## 2.4 — Ejemplo de test de integración.  🔴 *(tu tema flojo)*

<br><br>

### ⬇️ RESPUESTA 2.4

Con **Supertest** y el **repo mockeado**: `PATCH /notificaciones/n1` (del usuario autenticado) → **200** y `leida: true` en la respuesta. Y `GET /usuarios/u1/notificaciones?leida=false` → **200** con la lista filtrada. Verificás status + body; sin base real.

🔎 **Qué busca el corrector:** integración por HTTP, repo mockeado, un caso de cada operación (marcar + listar), AAA.

💡 **Profundizar (pirámide de tests, repaso rápido):** **unitario** (una clase aislada, mock de dependencias) → **integración** (la cadena HTTP con Supertest, repo mockeado) → **E2E** (un navegador real haciendo clicks, ej. Cypress, con todo levantado). La base es ancha (unitarios baratos/rápidos), la punta angosta (E2E caros/lentos). Acá estamos en el medio.

---

## 2.5 — Pseudocódigo: ¿cómo manejás marcar como leída para que repetirlo no rompa nada?

<br><br>

### ⬇️ RESPUESTA 2.5

> ⚠️ Acá está la trampa: marcar como leída es **idempotente**. Repetir la operación da el **mismo resultado** y **no es un conflicto**. No va un 409.

```js
async marcarLeida(notifId, usuarioId) {
  const notif = await this.notificacionRepository.obtenerPorId(notifId);
  if (!notif) throw new NotFoundError("La notificación no existe");   // 404
  if (notif.usuarioId !== usuarioId) throw new ForbiddenError();      // 403

  // Idempotente: si ya estaba leída, devolvemos igual (NO error)
  if (notif.leida) return notif;                                      // 200, sin cambios

  notif.leida = true;
  return await this.notificacionRepository.guardar(notif);            // 200
}
```

🔎 **Qué busca el corrector:** entender la **idempotencia**: "poner leída = true" repetido no genera conflicto; se devuelve 200 igual. Esto **contrasta** con la unicidad de los parciales 01-03 (donde repetir SÍ era 409).

🟡 **Divague típico:** tirar un `ConflictError` (409) porque "ya está leída". Es el error conceptual que este parcial busca cazar: **409 es para evitar duplicados/conflictos de creación; un set idempotente no entra ahí.**

---

## 2.6 — *(Extra de teoría que puede caer)* ¿Qué tenés que tener en cuenta para que el frontend consuma estos endpoints desde otro origen?

<br><br>

### ⬇️ RESPUESTA 2.6

**CORS** (Cross-Origin Resource Sharing). Como el frontend (ej. `localhost:5173` o el dominio de Netlify) y el backend corren en **orígenes distintos**, el navegador bloquea las requests por seguridad salvo que el backend lo **habilite explícitamente** con cabeceras `Access-Control-Allow-Origin` (y métodos/headers permitidos). En Express se resuelve con el middleware `cors`, configurando qué orígenes, métodos y headers se aceptan.

🔎 **Qué busca el corrector:** nombrar **CORS**, entender que es una protección del **navegador** ante orígenes distintos, y que se habilita en el **backend** (no es un bug, es una política a configurar).

---
---

# 3 · PERSISTENCIA  🔴 *(tu tema flojo — acá: colección + populate)*

## 3.1 — Cambios al modelo. ¿Colección propia o embebido? ¿Cómo mostrás el turno relacionado?

<br><br>

### ⬇️ RESPUESTA 3.1

**Colección propia `Notificacion`:** `{ _id, usuario: ObjectId, tipo, mensaje, leida: false, turno: ObjectId?, createdAt }`. Comparación:

- **Colección aparte (elegida):** ✅ las notificaciones **crecen sin límite** por usuario y se consultan/paginan **por su cuenta** (no querés inflar el documento del usuario con cientos de notificaciones). ✅ se referencian a `usuario` y opcionalmente a `turno`. ❌ una colección más.
- **Embebido en el usuario:** ❌ el documento del usuario crecería sin techo y se cargaría entero cada vez que lo leés, aunque no quieras las notificaciones. No conviene.

**Mostrar el turno relacionado → `populate`.** La notificación guarda solo el `ObjectId` del turno; para mostrar datos del turno (fecha, médico) en la UI, se **popula** la referencia al consultar (`.populate("turno")`), que trae el documento completo en lugar del id.

🔎 **Qué busca el corrector:** **colección propia** con el criterio (crecen sin techo / se consultan aparte), referencias por id, y **populate** para "rellenar" la referencia al mostrar.

💡 **Profundizar (populate):** `ref` en el schema apunta al **nombre del modelo** (`mongoose.model('Turno', ...)`), no al nombre de la colección. Sin populate, el campo `turno` viene como un `ObjectId` crudo; con populate, viene el documento entero. Es el equivalente conceptual a un JOIN, pero lo pedís explícitamente cuando lo necesitás.

🟡 **Divague típico:** decir "embebo las notificaciones en el usuario" sin notar que crecen sin límite. **Respuesta justa:** "colección propia (crecen y se consultan aparte), referencio al turno por id y lo populo para mostrarlo".

---
---

# 4 · SOBRE LA FUNCIONALIDAD (CRITERIO)

## 4.1 — Si un usuario marca como leída una notificación que ya estaba leída, ¿debería dar error? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.1

No. La operación es **idempotente**: el estado objetivo ("leída") ya se cumple, así que volver a aplicarla da el mismo resultado sin efectos secundarios. Se devuelve **200** con la notificación, no un error. Distinto de crear un favorito o una valoración duplicada, donde repetir **sí** generaría un registro de más (por eso ahí va 409).

🔎 **Qué busca el corrector:** **no**, por **idempotencia**, y saber distinguirlo de los casos de unicidad de creación (409). Es el punto que separa al que entiende los códigos del que los memoriza.

---

## 4.2 — Para mostrar el contador de no leídas, ¿lo calculás en cada request o lo guardás precalculado? Compará.

<br><br>

### ⬇️ RESPUESTA 4.2

**Calcularlo en cada request** (un `countDocuments({ usuario, leida: false })`) es lo más simple y siempre **consistente**: nunca queda desfasado. **Guardarlo precalculado** (un campo `noLeidas` en el usuario) ahorra esa consulta si el contador se pide muchísimo, pero introduce el riesgo de **desincronización** (hay que actualizarlo en cada alta/lectura, y si algo falla queda mal). Para el volumen del TP, **calcular en cada request alcanza y es más seguro**; el contador denormalizado se justifica solo a gran escala.

🔎 **Qué busca el corrector:** el trade-off **simplicidad/consistencia vs performance/denormalización**, y elegir lo simple para esta escala justificándolo.

---

## 4.3 — ¿Qué funcionalidades ya hechas del TP generan o se relacionan con notificaciones?

<br><br>

### ⬇️ RESPUESTA 4.3

**Reserva de turno** (notifica al médico), **cancelación** (notifica a la contraparte) y **reprogramación** — todas las acciones sobre turnos **disparan** la creación de notificaciones. O sea, esta feature **consume** los eventos que ya generan las funcionalidades existentes; el `TurnoService` llama al `NotificacionService` cuando ocurre cada evento.

🔎 **Qué busca el corrector:** ver que las notificaciones se **alimentan** de los eventos de turnos ya implementados (reserva, cancelación), no que vivan aisladas.

---
---

## 🏁 Cierre

Contá los **🔎** que tocaste de verdad. Foco:
- **El punto del parcial:** si en **2.5 / 4.1** dijiste 409 para "ya leída" → ese es exactamente el error que este modelo quería cazar. Grabate: **idempotente ≠ conflicto**.
- Si en **3.1** no mencionaste **populate** → es el tema nuevo de persistencia acá.
- Si en **2.6** no supiste **CORS** → repasalo, los profes avisaron que puede caer.

**Cobertura acumulada (01-04):** REST + auth · capas · errores (404/409/422/400/401-403) · test de integración + **pirámide de tests** · guardas: unicidad (01-02), transición (03), **idempotencia/no-conflicto (04)** · persistencia: **embebido refs (01)**, **colección (02)**, **sub-docs + soft delete (03)**, **colección + populate (04)** · accesibilidad: toggle, rating, modal, **aria-live/badge** · **CORS** · denormalización/trade-offs · criterio.

**Sin tocar (para próximos):** ORM vs ODM · CSR vs SSR (renderizado) · DTOs / capa de mapeo en detalle · TDD/BDD · filtros + ordenamiento + paginación combinados en búsqueda · arquitecturas (MVC/MVVM/monolito vs microservicios).

**FIN DEL PARCIAL MODELO 04**
