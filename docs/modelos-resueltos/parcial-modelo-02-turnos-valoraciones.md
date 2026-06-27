# 📝 Parcial Modelo 02 — Valoraciones de Médicos

**Materia:** Desarrollo de Software — UTN FRBA · **Dominio:** turnos médicos (tu TP) · **Tiempo sugerido:** 1h 30min

---

## 🎯 Cómo usar

Leé la **consigna** → tapá lo que está bajo `⬇️ RESPUESTA` → respondé al hueso, sin divagar → destapá y compará. Mirá el bloque **🔎 qué busca el corrector** (tu medida de precisión) y, donde esté, el **🟡 divague típico** (lo que NO conviene). El **💡 profundizar** es opcional: salteálo si vas rápido, abrilo si el tema te tiembla.

> **Core fijo de todos los parciales:** endpoints REST (verbo + url + body/params + **autenticación**), capa de servicios, manejo de errores, **test de integración**, **validación anti-duplicado**, **persistencia (embebido vs colección)**, **accesibilidad**, y **criterio**. Rota la funcionalidad.

---

## 📋 Enunciado

> El equipo de producto quiere que, una vez que un turno pasó a estado **REALIZADO**, el **paciente** pueda dejar una **valoración** del médico que lo atendió: un **puntaje del 1 al 5** y un **comentario opcional**. La acción está disponible desde el turno realizado (en "Mis turnos").
>
> Además, cualquier usuario debe poder **ver las valoraciones de un médico** y su **promedio** desde la vista de detalle del médico.
>
> Reglas del negocio: solo se puede valorar a un médico con quien se tuvo un turno **REALIZADO**, y **cada turno se puede valorar una sola vez**.
>
> Teniendo en cuenta las tecnologías y la solución del TP, resolvé los siguientes puntos.

---
---

# 1 · UI / FRONTEND

## 1.1 — Croquis y apariencia de las pantallas.

<br><br>

### ⬇️ RESPUESTA 1.1

Dos pantallas. **(1) Mis turnos / detalle del turno realizado (modificada):** en cada turno con estado REALIZADO aparece un botón "Valorar". Al tocarlo se abre un **formulario** con selector de estrellas (1-5) y un campo de comentario opcional. **(2) Detalle del médico (modificada):** muestra el **promedio** (ej. ★ 4,2) y una **lista de valoraciones** (puntaje + comentario + fecha), con estados de carga / vacío / error.

🔎 **Qué busca el corrector:** identificar las dos vistas, que la valoración nazca desde un turno **REALIZADO** (no desde cualquier lado), y que el médico muestre **promedio + lista**.

---

## 1.2 — Componentes nuevos y modificados.

<br><br>

### ⬇️ RESPUESTA 1.2

**Nuevos:** `FormValoracion` (input de estrellas + comentario), `ListaValoraciones`, `PromedioEstrellas`, y un `valoracionService` (axios). **Modificados:** la card/detalle del turno en "Mis turnos" (suma el botón "Valorar") y el detalle del médico (suma promedio + lista).

🔎 **Qué busca el corrector:** distinguir nuevos vs modificados; un componente reutilizable para las estrellas (se usa tanto para *cargar* como para *mostrar*).

---

## 1.3 — Interacción al (a) valorar y (b) consultar valoraciones.

<br><br>

### ⬇️ RESPUESTA 1.3

**(a) Valorar:** el paciente toca "Valorar" en un turno realizado → completa estrellas y comentario → al enviar se dispara un `POST` → loader mientras responde → si OK, confirma y el botón pasa a "Ya valoraste"; si falla, muestra el error. **(b) Consultar:** entra al detalle del médico → `GET` de las valoraciones → loader → render del promedio y la lista (o estado vacío "Aún no tiene valoraciones").

🔎 **Qué busca el corrector:** que el flujo conecte con el back y contemple **loading / éxito / error**; que valorar solo sea posible sobre un turno realizado y una sola vez.

---

## 1.4 — Aspectos de accesibilidad.

<br><br>

### ⬇️ RESPUESTA 1.4

El selector de estrellas no puede ser un puñado de íconos clickeables sueltos: se modela como un **grupo de radios** (`role="radiogroup"`), cada estrella con su `aria-label` ("1 estrella", …, "5 estrellas"), **navegable por teclado** (flechas/Tab + Enter). HTML semántico nativo primero (**primera regla de ARIA**). La lista de valoraciones como `<ul>/<li>`. No comunicar el puntaje **solo por color**; contraste y tamaños adecuados para mobile.

🔎 **Qué busca el corrector:** la **primera regla de ARIA** + el caso fino del **rating accesible** (no solo estrellas pintadas: labels + teclado) + 3-4 puntos concretos. Se evalúa, no lo saltees.

---
---

# 2 · BACKEND

## 2.1 — Rutas REST (verbo, url, body, path vars, query params) y autenticación.

<br><br>

### ⬇️ RESPUESTA 2.1

> 🔴 **Tip de oro:** aclarar SIEMPRE cómo se autentica el usuario. No alcanza verbo + url.

Todos con `Authorization: Bearer <JWT>` (el back deriva qué paciente es y valida que sea el dueño):

| Verbo | URL | Para qué |
|---|---|---|
| `POST` | `/medicos/:medicoId/valoraciones` | Crear valoración. Body: `{ "turnoId", "puntaje", "comentario?" }`. → 201 |
| `GET` | `/medicos/:medicoId/valoraciones` | Listar valoraciones del médico + promedio. Query: `?page=&pageSize=`. → 200 |
| `DELETE` | `/medicos/:medicoId/valoraciones/:id` | El paciente borra su propia valoración. → 200/204 |

🔎 **Qué busca el corrector:** valoraciones como **sub-recurso del médico**, verbos correctos, body/params bien puestos, **paginación** en el GET, y la **autenticación aclarada** (sin eso el punto queda flojo).

🟡 **Divague típico:** `GET /traerValoracionesDelMedico?id=...` (verbo en la URL, no-REST) y olvidarse la auth. **Respuesta justa:** `GET /medicos/:id/valoraciones` + "Bearer JWT, valida dueño". Una línea por endpoint.

---

## 2.2 — Lógica de la capa de servicios.

<br><br>

### ⬇️ RESPUESTA 2.2

El `ValoracionService.crear(pacienteId, medicoId, { turnoId, puntaje, comentario })` orquesta: valida que existan **paciente**, **médico** y **turno** (404 c/u); valida que el turno **sea de ese paciente, sea de ese médico y esté REALIZADO** (si no → 422); valida que ese **turno no haya sido valorado antes** (si sí → 409); persiste. Intervienen `PacienteService`, `MedicoService` y `TurnoService` (para chequear estado y pertenencia del turno).

🔎 **Qué busca el corrector:** la lógica vive en el **service**; aparecen las dos validaciones clave — **regla de negocio** (turno realizado y propio → 422) y **unicidad** (un turno, una valoración → 409); se apoya en otros services (inyección de dependencias).

🟡 **Divague típico:** explicar cómo Mongoose hace el `save`, o calcular el promedio paso a paso acá. **Respuesta justa:** "valido entidades, valido que el turno esté realizado y sea del paciente, valido que no esté ya valorado, guardo".

---

## 2.3 — Gestión de errores.

<br><br>

### ⬇️ RESPUESTA 2.3

Paciente / médico / turno inexistente → **404**. Turno no realizado o que no es del paciente → **422** (regla de negocio violada). Turno ya valorado → **409**. Puntaje fuera de 1-5 o body inválido → **400** (Zod en el borde). Token ausente/ajeno → **401 / 403**. El service lanza errores tipados; el **middleware único** los mapea a su status; el controller solo hace `next(err)`. El dominio no conoce HTTP.

🔎 **Qué busca el corrector:** distinguir **422 (regla de negocio)** de **409 (duplicado)** de **400 (validación de formato)**; el manejo centralizado en un middleware.

---

## 2.4 — Ejemplo de un test de integración.  🔴 *(tu tema flojo)*

<br><br>

### ⬇️ RESPUESTA 2.4

Con **Supertest**, levantando la app completa (router + controller + service + middlewares) y **solo el repositorio mockeado** (sin base real). Ejemplo: `POST /medicos/m1/valoraciones` con un `turnoId` de un turno realizado y propio → **201**; un segundo POST sobre el mismo turno → **409**. Verificás **status + body**, no persistencia real (no hay base en el test).

🔎 **Qué busca el corrector:** que sea **integración** (cruza capas vía HTTP), que **solo se mockee el repo**, y un caso happy (201) + uno de error (409). Patrón **AAA**.

🟡 **Divague típico:** decir "después hago un GET para ver si quedó guardado". Como el repo está mockeado, **no hay persistencia real que verificar** — verificás la respuesta y, si querés, que el mock se haya llamado.

💡 **Profundizar (tests):**
- **Unitario vs integración:** el unitario prueba el service aislado (lo instanciás con un repo mock y llamás al método directo). El de integración manda un **request HTTP** y prueba toda la cadena, incluidos validaciones, middlewares de error y el JSON de salida. No se reemplazan, se complementan.
- Esqueleto:
```js
import request from "supertest";
import { describe, test, expect, jest, beforeEach } from "@jest/globals";
import { buildTestApp } from "./utils/buildApp.js";

describe("Valoraciones API - Integración", () => {
  let app, valoracionRepository;
  beforeEach(() => {
    valoracionRepository = { existePorTurno: jest.fn(), guardar: jest.fn() };
    app = buildTestApp({ valoracionRepository });
  });

  test("POST crea valoración → 201", async () => {
    valoracionRepository.existePorTurno.mockResolvedValue(false);       // Arrange
    valoracionRepository.guardar.mockResolvedValue({ id: "v1", puntaje: 5 });
    const res = await request(app)                                       // Act
      .post("/medicos/m1/valoraciones")
      .send({ turnoId: "t1", puntaje: 5 });
    expect(res.status).toBe(201);                                       // Assert
  });

  test("POST sobre un turno ya valorado → 409", async () => {
    valoracionRepository.existePorTurno.mockResolvedValue(true);
    const res = await request(app)
      .post("/medicos/m1/valoraciones")
      .send({ turnoId: "t1", puntaje: 4 });
    expect(res.status).toBe(409);
  });
});
```

---

## 2.5 — Pseudocódigo de la validación anti-duplicado en el service.

<br><br>

### ⬇️ RESPUESTA 2.5

Antes de guardar, preguntar al repo si ese turno ya tiene valoración; si sí, `ConflictError` (409).

```js
async crearValoracion(pacienteId, medicoId, { turnoId, puntaje, comentario }) {
  await this.medicoService.obtenerPorId(medicoId);          // 404 si no existe
  const turno = await this.turnoService.obtenerPorId(turnoId); // 404 si no existe

  // Regla de negocio: el turno tiene que ser de este paciente, de este médico, y REALIZADO
  if (turno.pacienteId !== pacienteId || turno.medicoId !== medicoId || turno.estado !== "REALIZADO") {
    throw new UnprocessableEntityError("Solo se puede valorar un turno realizado propio"); // 422
  }

  // Unicidad: un turno se valora una sola vez
  if (await this.valoracionRepository.existePorTurno(turnoId)) {
    throw new ConflictError("Este turno ya fue valorado"); // 409
  }

  return await this.valoracionRepository.guardar({ pacienteId, medicoId, turnoId, puntaje, comentario });
}
```

🔎 **Qué busca el corrector:** la **guarda explícita** antes de insertar; la unicidad se resuelve **en el servidor**; el 422 separado del 409.

💡 **Profundizar:** la unicidad por turno se puede reforzar con un **índice único** sobre `turnoId` en la base. Código + índice = defensa en profundidad (si entran dos requests casi simultáneas, la base rechaza la segunda).

---
---

# 3 · PERSISTENCIA  🔴 *(tu tema flojo — y acá la decisión se da vuelta vs favoritos)*

## 3.1 — Cambios al modelo. ¿Nueva colección o embebido? Elegí y compará.

<br><br>

### ⬇️ RESPUESTA 3.1

**Elijo colección aparte.** Una colección `Valoracion` con `{ _id, pacienteId, medicoId, turnoId, puntaje, comentario, createdAt }`. Comparación:

- **Embebido (array de valoraciones dentro del Médico):** ✅ una lectura trae al médico con sus valoraciones. ❌ el documento del médico **crece sin límite** (un médico puede juntar miles), no podés consultar/paginar valoraciones por separado con comodidad, y mezclás dos entidades con vida propia.
- **Colección aparte (elegida):** ✅ **escala** sin inflar al médico, **paginás** las valoraciones, calculás el **promedio** con una agregación, sumás metadata (fecha), y un **índice único en `turnoId`** garantiza el no-duplicado. ❌ una colección más y, si querés mostrar datos del paciente, un `populate`.

> **El contraste con favoritos (parcial 01):** ahí ganaba *embebido* porque la lista era **acotada** y se consultaba **junto al paciente**. Acá gana *colección* porque las valoraciones son **muchas, crecen sin techo y se consultan por médico**. Misma decisión, resultado opuesto: lo que manda es **cómo se consulta y cuánto crece**, no una regla fija.

🔎 **Qué busca el corrector:** que **elijas y justifiques**, que **compares las dos**, y que el criterio sea **uno-a-pocos/consulta junta** (→ embebido) vs **uno-a-muchos grande/consulta separada** (→ colección). Bonus: el índice único.

🟡 **Divague típico:** describir las dos sin comprometerte, o re-explicar todo Mongoose. **Respuesta justa:** "colección aparte porque crecen sin límite y se consultan por médico; embebido serviría si fueran pocas y se leyeran con el médico".

---
---

# 4 · SOBRE LA FUNCIONALIDAD (CRITERIO)

## 4.1 — ¿Se debería poder valorar dos veces el mismo turno? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.1

No. La valoración representa la experiencia de **ese** turno; dos valoraciones del mismo turno se contradicen y distorsionan el promedio. Se garantiza con la unicidad por `turnoId` en el service (e índice único en la base).

🔎 **Qué busca el corrector:** un **no** + el porqué en una oración, conectado con la validación 2.5.

---

## 4.2 — ¿Tiene sentido valorar a un médico con quien nunca tuviste un turno realizado? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.2

No. La valoración tiene que estar **respaldada por una atención real** (turno REALIZADO y propio); si no, abrís la puerta a reseñas falsas y se pierde la confiabilidad del promedio. Por eso es una **regla de negocio** que el service valida (→ 422), no algo que dependa del front.

🔎 **Qué busca el corrector:** atar la valoración a un **turno realizado** (anti-fraude) y reconocer que es regla de **negocio en el servidor** (422), no validación de formato.

---

## 4.3 — ¿Qué funcionalidades ya hechas del TP se ven afectadas?

<br><br>

### ⬇️ RESPUESTA 4.3

**Búsqueda de turnos:** poder ordenar/destacar médicos por su promedio de valoraciones. **Detalle del médico:** mostrar promedio + reseñas. **Marcar turno como realizado:** es el evento que *habilita* poder valorar, así que el flujo de estados del turno queda atado a esta feature.

🔎 **Qué busca el corrector:** conectar la feature nueva con el sistema existente (búsqueda, detalle del médico) y notar la dependencia con el **estado REALIZADO** del turno.

---
---

## 🏁 Cierre

Contá cuántos **🔎** tocaste, sin "me sonaba". Foco según cómo fue:
- Si en **3.1** no supiste *por qué acá gana colección y en favoritos ganaba embebido* → ese es el corazón del tema persistencia. Releé el contraste: manda **cómo se consulta y cuánto crece**.
- Si en **2.4** dijiste "verifico con un GET que quedó guardado" → recordá que el repo está **mockeado**: no hay base real.
- Si en **2.2/2.3** mezclaste **422 (regla de negocio)** con **409 (duplicado)** o **400 (formato)** → tenelos separados, es un clásico de corrección.

**Cobertura acumulada (01 + 02):** REST + auth · capas y validaciones · errores (404/409/**422**/400/401-403) · test de integración · anti-duplicado en service + índice único · **embebido (01)** y **colección (02)** argumentados en espejo · accesibilidad (toggle en 01, **rating/radiogroup en 02**) · criterio de negocio · paginación.

**Todavía sin tocar (para próximos):** populate · soft delete · estados de turno y transiciones · DTOs / capa de mapeo · CORS · pirámide de tests (unitario/integración/E2E, TDD/BDD) · ORM vs ODM · idempotencia (PUT vs POST) · CSR vs SSR · filtros/ordenamiento avanzados en búsqueda.

**FIN DEL PARCIAL MODELO 02**
