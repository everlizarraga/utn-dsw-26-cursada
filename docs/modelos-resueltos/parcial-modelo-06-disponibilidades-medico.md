# 📝 Parcial Modelo 06 — Disponibilidades del Médico

**Materia:** Desarrollo de Software — UTN FRBA · **Dominio:** turnos médicos (tu TP) · **Tiempo sugerido:** 1h 30min

---

## 🎯 Cómo usar

Leé la consigna → tapá lo que está bajo `⬇️ RESPUESTA` → respondé al hueso → destapá y compará. Mirá **🔎 qué busca el corrector**, **🟡 divague típico**, y el **💡 profundizar** opcional.

> **Core fijo:** endpoints (verbo + url + body/params + **auth**), servicios, errores, **test de integración**, **validación/guarda**, **persistencia (embebido vs colección)**, **accesibilidad**, **criterio**. Rota la funcionalidad.
>
> 🆕 **Novedad:** este parcial cambia de actor. Hasta ahora todo era del lado del **paciente**; acá la funcionalidad es del **médico**, lo que abre el tema de **autorización por rol**.

---

## 📋 Enunciado

> El equipo de producto necesita que el **médico** pueda gestionar sus **disponibilidades horarias**: definir en qué **día de la semana** y en qué **rango horario** (hora desde / hora hasta) atiende, en qué **sede**. Debe poder agregar, listar y eliminar sus disponibilidades, desde su perfil.
>
> Regla clave: **dos disponibilidades del mismo médico no se pueden solapar** en horario. Las disponibilidades son la base sobre la que se generan los turnos buscables.
>
> Resolvé los siguientes puntos teniendo en cuenta la solución del TP.

---
---

# 1 · UI / FRONTEND

## 1.1 — Croquis y apariencia de las pantallas.

<br><br>

### ⬇️ RESPUESTA 1.1

En el **perfil del médico** (modificada), una sección **"Mis disponibilidades"**: una **lista** de las disponibilidades actuales (día, rango horario, sede, con botón "Eliminar" cada una) y un **formulario** para agregar una nueva (selector de día, hora desde, hora hasta, sede). Estados de carga / vacío / error, y feedback claro cuando una disponibilidad se solapa.

🔎 **Qué busca el corrector:** la lista + el form de alta, y que se contemple el **mensaje de error de solapamiento** (no solo el happy path).

---

## 1.2 — Componentes nuevos y modificados.

<br><br>

### ⬇️ RESPUESTA 1.2

**Nuevos:** `FormDisponibilidad` (día, horas, sede), `ListaDisponibilidades`, `DisponibilidadItem`, y un `disponibilidadService` (axios). **Modificados:** el perfil del médico (suma la sección).

🔎 **Qué busca el corrector:** distinguir nuevos vs modificados; un form reutilizable para alta/edición.

---

## 1.3 — Interacción al (a) agregar y (b) eliminar una disponibilidad.

<br><br>

### ⬇️ RESPUESTA 1.3

**(a) Agregar:** el médico completa el form → `POST` → loader → si OK, la disponibilidad aparece en la lista; si **solapa** con otra existente, muestra el error sin agregarla. **(b) Eliminar:** click en "Eliminar" → confirmación → `DELETE` → la saca de la lista.

🔎 **Qué busca el corrector:** conexión real, manejo del **error de solapamiento**, confirmación antes de eliminar.

---

## 1.4 — Aspectos de accesibilidad.

<br><br>

### ⬇️ RESPUESTA 1.4

El form lleva `label` asociado a cada control (día, horas, sede) con `for`/`id` o `aria-label`. Los **mensajes de error** del form se asocian al campo con `aria-describedby`, para que el lector de pantalla los lea junto al input que falló. Navegable por teclado, foco visible, errores comunicados con texto (no solo color/borde rojo). HTML semántico nativo primero (primera regla de ARIA).

🔎 **Qué busca el corrector:** **labels** en los campos del form + `aria-describedby` para los errores (el punto fino en formularios) + teclado + no-solo-color.

---
---

# 2 · BACKEND

## 2.1 — Rutas REST y autenticación.

<br><br>

### ⬇️ RESPUESTA 2.1

> 🔴 Tip de oro: auth siempre. Acá con un giro: además de autenticar, hay que **autorizar por rol** — solo el propio médico (no un paciente, no otro médico) edita sus disponibilidades.

Con `Authorization: Bearer <JWT>`:

| Verbo | URL | Para qué |
|---|---|---|
| `POST` | `/medicos/:id/disponibilidades` | Crear. Body: `{ diaSemana, horaDesde, horaHasta, sedeId }`. → 201 |
| `GET` | `/medicos/:id/disponibilidades` | Listar las del médico. → 200 |
| `DELETE` | `/medicos/:id/disponibilidades/:dispId` | Eliminar una. → 200/204 |

El backend valida que el token sea de **ese** médico (rol médico + dueño). Un paciente o un médico distinto → 403.

🔎 **Qué busca el corrector:** sub-recurso del médico, body completo, y la **autorización por rol/dueño** (no solo "está logueado", sino "es el médico correcto").

🟡 **Divague típico:** `POST /crearDisponibilidad` con todo en el body y sin mencionar quién puede hacerlo. **Respuesta justa:** `POST /medicos/:id/disponibilidades` + "auth Bearer, valida que sea el propio médico (403 si no)".

---

## 2.2 — Lógica de la capa de servicios.

<br><br>

### ⬇️ RESPUESTA 2.2

`MedicoService.crearDisponibilidad(medicoId, datos)`: valida que el médico **exista** (404); valida que el **rango sea válido** (`horaHasta > horaDesde`, si no → 400); valida que la nueva disponibilidad **no se solape** con ninguna existente del médico (si solapa → 409); persiste. El listar trae las del médico; el eliminar valida existencia y borra.

🔎 **Qué busca el corrector:** las tres validaciones — existencia, **rango válido**, y **no solapamiento** — en la capa de servicio, antes de persistir.

🟡 **Divague típico:** describir cómo se guarda en Mongo. **Respuesta justa:** "valido médico, valido rango, valido que no solape, guardo".

---

## 2.3 — Gestión de errores.

<br><br>

### ⬇️ RESPUESTA 2.3

Médico inexistente → **404**. Rango inválido (`horaHasta <= horaDesde`) → **400** (validación de formato/datos). Solapamiento con otra disponibilidad → **409** (conflicto con el estado actual). Token de otro rol/usuario → **403**. Body incompleto → **400**. Tipados + middleware.

🔎 **Qué busca el corrector:** distinguir **400** (rango mal formado) de **409** (solapamiento, conflicto con lo existente) y el **403** por rol.

---

## 2.4 — Ejemplo de test de integración.  🔴 *(tu tema flojo — y este escenario ya lo tenés en el repo)*

<br><br>

### ⬇️ RESPUESTA 2.4

Con **Supertest** y el **repo mockeado**: `POST /medicos/m1/disponibilidades` con un rango que no choca → **201**; otro `POST` con un rango que **solapa** uno existente → **409**. Verificás status + body; sin base real.

🔎 **Qué busca el corrector:** integración por HTTP, repo mockeado, caso válido (201) + solapamiento (409), AAA.

💡 **Profundizar (TDD/BDD + tu repo):** este escenario ya existe en tu repo como test **unitario** (`disponibilidades.test.js`: testea `MedicoService.crearDisponibilidad` con el repo mockeado, cubriendo médico inexistente → NotFound, rango inválido → BadRequest, solapamiento → Conflict). La versión de **integración** es la misma idea pero entrando por HTTP con Supertest. **TDD** (Test-Driven Development) = escribís el test **antes** del código y dejás que falle, luego implementás hasta que pase (red → green → refactor). **BDD** (Behavior-Driven) = describís el comportamiento esperado en lenguaje cercano al negocio ("debería rechazar si se solapa"), que es justo cómo están redactados los `test("Error si se superponen rangos…")`.

---

## 2.5 — Pseudocódigo de la validación de solapamiento de rangos.

<br><br>

### ⬇️ RESPUESTA 2.5

> Esta guarda es más jugosa que las anteriores: no es "ya existe igual", es **¿se pisan dos intervalos?**.

```js
async crearDisponibilidad(medicoId, { diaSemana, horaDesde, horaHasta, sedeId }) {
  await this.medicoService.obtenerPorId(medicoId);              // 404 si no existe

  if (horaHasta <= horaDesde) {
    throw new BadRequestError("La hora fin debe ser mayor a la de inicio"); // 400
  }

  const existentes = await this.disponibilidadRepository
    .obtenerDelMedicoEnDia(medicoId, diaSemana);

  // Dos intervalos [a,b) y [c,d) se solapan si: a < d && c < b
  const haySolapamiento = existentes.some(e =>
    horaDesde < e.horaHasta && e.horaDesde < horaHasta
  );
  if (haySolapamiento) {
    throw new ConflictError("La disponibilidad se solapa con otra existente"); // 409
  }

  return await this.disponibilidadRepository.guardar(medicoId, { diaSemana, horaDesde, horaHasta, sedeId });
}
```

🔎 **Qué busca el corrector:** la **fórmula de solapamiento de intervalos** (`inicioA < finB && inicioB < finA`), el filtro por **mismo día**, y que la validación esté en el **service**. No busca un algoritmo rebuscado; busca que sepas comparar dos rangos.

---
---

# 3 · PERSISTENCIA  🔴 *(tu tema flojo — caso debatible a propósito)*

## 3.1 — Cambios al modelo. ¿Embebido en el médico o colección aparte? Elegí y compará.

<br><br>

### ⬇️ RESPUESTA 3.1

Caso **genuinamente debatible** — lo importante es que justifiques:

- **Embebido en el Médico** (`disponibilidades: [{ diaSemana, horaDesde, horaHasta, sede }]`): ✅ las disponibilidades **no tienen vida propia** (no existen sin su médico) y se leen casi siempre **con el médico**; la cantidad está **acotada** (un médico tiene una agenda finita). Encaja con el criterio de embebido.
- **Colección aparte** (`Disponibilidad` con `ref` al médico): ✅ la **búsqueda de turnos** consulta disponibilidades **a través de muchos médicos** y las filtra/ordena; tenerlas en su colección facilita **indexar y consultar** sin cargar el médico entero. ❌ requiere join/populate para ver al médico.

**Mi elección:** si el peso de la app está en "el médico edita su agenda", **embebido** es lo simple y suficiente. Si el peso está en "buscar entre todas las disponibilidades de todos los médicos" (que es el caso del TP), una **colección aparte indexada** rinde mejor. Cualquiera se defiende; lo que el corrector mide es que reconozcas el trade-off según el patrón de consulta dominante.

🔎 **Qué busca el corrector:** reconocer que es un caso **debatible**, presentar las dos con el criterio (vida propia/consulta junta → embebido; consulta global/indexar → colección) y **comprometer** una elección justificada. Bonus: mencionar **índices** para la consulta de búsqueda.

🟡 **Divague típico:** elegir sin comparar, o no notar que la **búsqueda de turnos** es lo que tracciona hacia una colección consultable. **Respuesta justa:** "embebido si el foco es la agenda del médico; colección indexada si el foco es buscar entre todos — en el TP la búsqueda tracciona hacia colección".

---
---

# 4 · SOBRE LA FUNCIONALIDAD (CRITERIO)

## 4.1 — ¿Se deberían permitir disponibilidades solapadas? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.1

No. Dos disponibilidades pisadas generarían **turnos ambiguos** en el mismo horario (¿a cuál pertenece el turno de las 10:00?), con riesgo de doble reserva del médico a la misma hora. La regla de no-solapamiento mantiene la agenda coherente. Se valida en el service (2.5).

🔎 **Qué busca el corrector:** **no** + el porqué concreto (turnos ambiguos / doble reserva) + conexión con la validación.

---

## 4.2 — ¿Quién debería poder modificar las disponibilidades de un médico, y cómo lo asegurás?

<br><br>

### ⬇️ RESPUESTA 4.2

Solo el **propio médico** (autorización por rol y por pertenencia). Se asegura en el backend: del JWT se obtiene el usuario y su rol; se valida que sea **rol médico** y que su id coincida con el `:id` del recurso. Un paciente o un médico distinto recibe **403**. No alcanza con autenticar (saber *quién* es); hay que **autorizar** (validar que *puede* hacer esta acción sobre este recurso).

🔎 **Qué busca el corrector:** distinguir **autenticación** (quién sos) de **autorización** (qué podés hacer), y que la validación de dueño/rol viva en el **backend**, no en el front.

---

## 4.3 — ¿Qué funcionalidades del TP dependen de las disponibilidades?

<br><br>

### ⬇️ RESPUESTA 4.3

**Búsqueda de turnos:** las disponibilidades **libres** son la materia prima de la búsqueda (los turnos buscables salen de expandir las disponibilidades). **Reserva:** ocupa una disponibilidad. **Cancelación/reprogramación:** libera/reasigna disponibilidades. Editar disponibilidades impacta directamente en qué turnos existen.

🔎 **Qué busca el corrector:** que las disponibilidades son la **base** de la búsqueda y la reserva — el centro del sistema, no un agregado aislado.

---
---

## 🏁 Cierre

Contá los **🔎** que tocaste. Foco:
- Si en **2.5** no te salió la **fórmula de solapamiento** (`inicioA < finB && inicioB < finA`) → memorizala, es el corazón de esta guarda.
- Si en **4.2** no separaste **autenticación vs autorización** → es el concepto nuevo de este parcial.
- Si en **3.1** elegiste sin comparar → en persistencia siempre se compara y se justifica.

**Cobertura acumulada (01-06):** REST + auth **+ autorización por rol** · capas · errores (404/409/422/400/401-403) · test de integración + pirámide + **TDD/BDD** · guardas: unicidad, transición, idempotencia, **solapamiento de intervalos** · persistencia: embebido-refs, colección, sub-docs+soft delete, colección+populate, sub-docs, **caso debatible + índices** · accesibilidad: toggle, rating, modal, aria-live, form + `aria-describedby` · CORS · DTO/mapeo · ODM vs ORM · CSR vs SSR · criterio.

**Lo que queda (más teórico, para un eventual parcial "de teoría"):** arquitecturas (MVC/MVVM/Flux, monolito vs microservicios/BFF) · ciclo de vida del software / metodologías · Git flows · ACID/CAP en profundidad. Avisame si querés uno cargado a eso.

**FIN DEL PARCIAL MODELO 06**
