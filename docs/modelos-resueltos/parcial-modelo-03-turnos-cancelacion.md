# 📝 Parcial Modelo 03 — Cancelación y Reprogramación de Turnos

**Materia:** Desarrollo de Software — UTN FRBA · **Dominio:** turnos médicos (tu TP) · **Tiempo sugerido:** 1h 30min

---

## 🎯 Cómo usar

Leé la consigna → tapá lo que está bajo `⬇️ RESPUESTA` → respondé al hueso → destapá y compará. Mirá **🔎 qué busca el corrector** y, donde esté, **🟡 divague típico**. El **💡 profundizar** es opcional.

> **Core fijo:** endpoints (verbo + url + body/params + **auth**), servicios, errores, **test de integración**, **validación/guarda**, **persistencia (embebido vs colección)**, **accesibilidad**, **criterio**. Rota la funcionalidad.

---

## 📋 Enunciado

> El equipo de producto necesita que el **paciente** pueda **cancelar** un turno reservado y **reprogramarlo** a otra disponibilidad del mismo médico, desde "Mis turnos".
>
> Reglas: solo se puede cancelar o reprogramar con cierta **antelación** (ej. 60 minutos antes del turno). Cancelar **libera la disponibilidad** para otros pacientes. Un turno cancelado **no se borra**: queda en el historial del paciente. Cada cambio de estado debe quedar **registrado**, y se debe **notificar a la contraparte**.
>
> Resolvé los siguientes puntos teniendo en cuenta la solución del TP.

---
---

# 1 · UI / FRONTEND

## 1.1 — Croquis y apariencia de las pantallas.

<br><br>

### ⬇️ RESPUESTA 1.1

Se trabaja sobre **Mis turnos** (modificada). En cada turno **activo** (RESERVADO/CONFIRMADO) aparecen "Cancelar" y "Reprogramar". **Cancelar** abre un **modal de confirmación** (acción destructiva). **Reprogramar** abre un selector con las **disponibilidades libres** del médico para elegir la nueva. Los turnos **cancelados** siguen visibles en el historial, con etiqueta de estado (no se borran de la vista).

🔎 **Qué busca el corrector:** confirmación para la acción destructiva (cancelar), un flujo para elegir la nueva disponibilidad al reprogramar, y que el cancelado **persista visible** en el historial.

---

## 1.2 — Componentes nuevos y modificados.

<br><br>

### ⬇️ RESPUESTA 1.2

**Reutilizados/modificados:** el `ConfirmacionModal` que ya existe (para confirmar la cancelación) y la card de turno en "Mis turnos" (suma los botones y la etiqueta de estado). **Nuevos:** un `ReprogramarModal` (o selector de disponibilidad) y, si hace falta, un `EstadoTurnoBadge`.

🔎 **Qué busca el corrector:** **reutilizar** el modal de confirmación existente en vez de inventar uno (consistencia, requerimiento no funcional); distinguir nuevo vs modificado.

---

## 1.3 — Interacción al (a) cancelar y (b) reprogramar.

<br><br>

### ⬇️ RESPUESTA 1.3

**(a) Cancelar:** click en "Cancelar" → modal "¿Seguro?" → al confirmar, `PATCH`/`DELETE` → loader → si OK, el turno pasa a CANCELADO en la vista; si falla (ej. ya pasó el tiempo), muestra el error. **(b) Reprogramar:** click en "Reprogramar" → elige nueva disponibilidad → `PUT`/`PATCH` → loader → si OK, actualiza fecha/hora; si la disponibilidad se ocupó, muestra el conflicto.

🔎 **Qué busca el corrector:** confirmación antes de cancelar, conexión real con el back, y manejo de los **casos de error** (fuera de tiempo, disponibilidad ocupada), no solo el happy path.

---

## 1.4 — Aspectos de accesibilidad.

<br><br>

### ⬇️ RESPUESTA 1.4

Foco en el **modal accesible**: `role="dialog"` + `aria-modal="true"`, el foco se mueve al modal al abrirlo y **vuelve al botón** al cerrarlo (focus trap), se cierra con **Esc**, y el título del modal está asociado con `aria-labelledby`. Botones con label claro. El estado del turno no se comunica **solo por color** (etiqueta de texto además del color). Navegable por teclado.

🔎 **Qué busca el corrector:** que el **modal** sea accesible (rol, foco, Esc) — es el punto fino de esta feature — más HTML semántico (primera regla de ARIA) y no-solo-color.

---
---

# 2 · BACKEND

## 2.1 — Rutas REST y autenticación.

<br><br>

### ⬇️ RESPUESTA 2.1

> 🔴 Tip de oro: aclarar siempre la autenticación.

Con `Authorization: Bearer <JWT>` (valida que el turno sea del paciente del token):

| Verbo | URL | Para qué |
|---|---|---|
| `DELETE` | `/turnos/:id` | Cancelar (soft): pasa a CANCELADO y libera la disponibilidad. → 200 |
| `PUT` | `/turnos/:id` | Reprogramar: reemplaza disponibilidad/fecha. Body: `{ disponibilidadId, fechaHora }`. → 200 |

(Variante válida: modelar cancelar como `PATCH /turnos/:id` con `{ estado: "CANCELADO" }`. Lo importante es justificar el verbo.)

🔎 **Qué busca el corrector:** verbos coherentes y **justificados**, body del reprogramar, auth aclarada. El cancelar es **soft** (no un borrado físico).

🟡 **Divague típico:** `POST /cancelarTurno?id=...`. **Respuesta justa:** `DELETE /turnos/:id` (soft cancel) + auth Bearer.

💡 **Profundizar (idempotencia · PUT vs PATCH):** **PUT** reemplaza el recurso completo y es **idempotente** (mandarlo dos veces con la misma fecha deja el mismo resultado). **PATCH** modifica parcialmente. Reprogramar a una fecha concreta encaja bien con PUT por eso. Cancelar **no** es idempotente en sentido estricto (la segunda vez ya está cancelado), por eso se protege con una guarda (ver 2.5).

---

## 2.2 — Lógica de la capa de servicios.

<br><br>

### ⬇️ RESPUESTA 2.2

`TurnoService.cancelar(turnoId, pacienteId)`: valida que el turno **exista** (404) y sea del paciente; que **no esté ya cancelado** (409); que esté **dentro de la ventana de tiempo** permitida (si no → 422); cambia el estado a CANCELADO, **registra el cambio en el historial**, **libera la disponibilidad** (vía `MedicoService`/`DisponibilidadService`) y **notifica a la contraparte** (vía `NotificacionService`). Reprogramar es análogo: valida que la nueva disponibilidad esté **libre** (si no → 409), reasigna y registra.

🔎 **Qué busca el corrector:** las validaciones de **transición de estado** (no cancelar lo ya cancelado) y de **negocio** (ventana de tiempo), más los efectos colaterales: **liberar disponibilidad**, **registrar historial**, **notificar**.

🟡 **Divague típico:** describir cómo Mongoose actualiza el documento. **Respuesta justa:** "valido existencia y dueño, valido que no esté cancelado y esté en tiempo, cambio estado, libero disponibilidad, registro historial, notifico".

---

## 2.3 — Gestión de errores.

<br><br>

### ⬇️ RESPUESTA 2.3

Turno inexistente → **404**. Turno ya cancelado / transición inválida → **409**. Fuera de la ventana de tiempo → **422**. Reprogramar a una disponibilidad ocupada → **409**. Body inválido → **400**. Token ajeno → **403**. El service lanza errores tipados; el middleware central los mapea.

🔎 **Qué busca el corrector:** separar **409** (estado/transición inválida o recurso ocupado) de **422** (regla de negocio: fuera de tiempo). El típico error es mandar todo a 400.

---

## 2.4 — Ejemplo de test de integración.  🔴 *(tu tema flojo)*

<br><br>

### ⬇️ RESPUESTA 2.4

Con **Supertest** y el **repo mockeado**: `DELETE /turnos/t1` sobre un turno RESERVADO dentro de tiempo → **200** y estado CANCELADO en la respuesta; un segundo `DELETE` sobre el mismo turno (ya cancelado) → **409**. Verificás status + body; no persistencia real (no hay base en el test).

🔎 **Qué busca el corrector:** que sea integración (cruza capas vía HTTP), repo mockeado, caso happy (200) + caso de error (409). Patrón AAA.

🟡 **Divague típico:** "consulto la base después para ver que quedó cancelado". El repo está mockeado: verificás la **respuesta** y, si querés, que el mock de `guardar` se haya llamado.

---

## 2.5 — Pseudocódigo de la guarda que impide cancelar un turno ya cancelado.

<br><br>

### ⬇️ RESPUESTA 2.5

```js
async cancelarTurno(turnoId, pacienteId) {
  const turno = await this.turnoRepository.obtenerPorId(turnoId);
  if (!turno) throw new NotFoundError("El turno no existe");           // 404
  if (turno.pacienteId !== pacienteId) throw new ForbiddenError();     // 403

  // Guarda de transición: no cancelar lo ya cancelado
  if (turno.estado === "CANCELADO") {
    throw new ConflictError("El turno ya estaba cancelado");           // 409
  }

  // Regla de negocio: ventana de tiempo
  if (!this.estaDentroDeVentana(turno.fechaHora)) {
    throw new UnprocessableEntityError("Pasó el tiempo para cancelar"); // 422
  }

  turno.actualizarEstado("CANCELADO", pacienteId, "Cancelado por paciente"); // registra historial
  await this.medicoService.liberarDisponibilidad(turno.medicoId, turno.disponibilidadId);
  return await this.turnoRepository.guardar(turno);
}
```

🔎 **Qué busca el corrector:** la **guarda de estado** explícita (no cancelar dos veces → 409), separada de la regla de tiempo (422), y que el cambio **quede registrado** en el historial.

---
---

# 3 · PERSISTENCIA  🔴 *(tu tema flojo — acá: historial embebido + soft delete)*

## 3.1 — Cambios al modelo: ¿cómo guardás el historial de estados? ¿Y conviene borrar el turno al cancelar o no?

<br><br>

### ⬇️ RESPUESTA 3.1

**Historial → embebido.** Un array de sub-documentos `historialEstados: [{ estado, fecha, quien, motivo }]` **dentro del turno**. Comparación:

- **Embebido (elegido):** ✅ el historial **siempre se lee junto al turno**, no tiene **vida propia** (un cambio de estado no existe sin su turno), y la cantidad está **acotada** (un turno tiene pocos cambios). ❌ no se consulta el historial por separado fácilmente (no es el caso de uso).
- **Colección de auditoría aparte:** ✅ útil si necesitaras analizar **todos** los cambios del sistema (auditoría global, métricas). ❌ exige join/populate y más queries para algo que casi siempre querés con el turno.

**Borrado → soft delete.** Cancelar **no borra** el documento: cambia el estado a CANCELADO (y/o un flag `eliminado`). Así el turno **sigue en el historial del paciente**, se conserva la **trazabilidad** y no se rompen referencias. El borrado físico (`findByIdAndDelete`) perdería esa información para siempre.

🔎 **Qué busca el corrector:** **embebido** para el historial con el criterio correcto (uno-a-pocos / sin vida propia / se lee junto), y **soft delete** justificado (auditoría + historial + el paciente lo ve). Comparar contra la alternativa en ambos casos.

🟡 **Divague típico:** proponer una colección de historial sin justificar por qué, o decir "borro el turno" sin notar que se pide que **quede en el historial**. **Respuesta justa:** "historial embebido (se lee con el turno, no tiene vida propia); cancelar es soft delete porque el turno tiene que sobrevivir en el historial".

---
---

# 4 · SOBRE LA FUNCIONALIDAD (CRITERIO)

## 4.1 — ¿Se debería poder cancelar un turno ya cancelado? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.1

No. Es una **transición de estado inválida**: el turno ya está en estado final CANCELADO, volver a cancelarlo no tiene efecto ni sentido. Se rechaza con 409. Modelar el ciclo de vida del turno (qué transiciones son válidas) evita estados incoherentes.

🔎 **Qué busca el corrector:** un **no** + "transición inválida / estado final" + conexión con la guarda 2.5.

---

## 4.2 — ¿Conviene borrar físicamente el turno al cancelar, o conservarlo? Compará.

<br><br>

### ⬇️ RESPUESTA 4.2

Conservarlo (**soft delete**). Borrarlo físicamente perdería el **historial** del paciente, la **trazabilidad** (quién canceló y cuándo) y podría **romper referencias** (notificaciones que lo apuntan). El soft delete mantiene la integridad y permite reportes. El único costo es que hay que **filtrar** los cancelados/eliminados en las consultas donde no van — un costo menor frente al beneficio.

🔎 **Qué busca el corrector:** elegir **soft delete** y comparar: borrado físico pierde datos y rompe referencias; soft delete conserva trazabilidad a cambio de filtrar en las queries.

---

## 4.3 — ¿Qué funcionalidades ya hechas del TP se ven afectadas?

<br><br>

### ⬇️ RESPUESTA 4.3

**Disponibilidad / búsqueda de turnos:** al cancelar se **libera** la disponibilidad, que vuelve a aparecer como reservable. **Notificaciones:** se avisa a la contraparte (médico) de la cancelación. **Mis turnos / historial:** el turno cancelado sigue visible con su estado.

🔎 **Qué busca el corrector:** la liberación de la disponibilidad (impacta la búsqueda), las notificaciones, y la visibilidad en el historial.

---
---

## 🏁 Cierre

Contá los **🔎** que tocaste de verdad. Foco:
- Si en **2.3/4.1** mezclaste **409 (transición/ocupado)** con **422 (fuera de tiempo)** → tenelos separados.
- Si en **3.1** no justificaste el **soft delete** por el historial → es el corazón de la consigna ("no se borra, queda en el historial").
- Si dudaste en **PUT vs PATCH/idempotencia** → abrí el 💡 de 2.1.

**Cobertura acumulada (01-03):** REST + auth · capas · errores (404/409/422/400/403) · test de integración · guardas (unicidad / transición de estado) · persistencia: **embebido refs (01)**, **colección (02)**, **embebido sub-docs + soft delete (03)** · accesibilidad: toggle (01), rating/radiogroup (02), **modal/dialog (03)** · **estados y transiciones** · **idempotencia / PUT vs PATCH** · criterio.

**Sin tocar (para próximos):** populate · contadores/denormalización · CORS · pirámide de tests (E2E, TDD/BDD) · ORM vs ODM · CSR vs SSR · DTOs/capa de mapeo.

**FIN DEL PARCIAL MODELO 03**
