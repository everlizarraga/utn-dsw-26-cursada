# 📝 Parcial Modelo 05 — Búsquedas Guardadas

**Materia:** Desarrollo de Software — UTN FRBA · **Dominio:** turnos médicos (tu TP) · **Tiempo sugerido:** 1h 30min

---

## 🎯 Cómo usar

Leé la consigna → tapá lo que está bajo `⬇️ RESPUESTA` → respondé al hueso → destapá y compará. Mirá **🔎 qué busca el corrector**, **🟡 divague típico**, y el **💡 profundizar** opcional.

> **Core fijo:** endpoints (verbo + url + body/params + **auth**), servicios, errores, **test de integración**, **validación/guarda**, **persistencia (embebido vs colección)**, **accesibilidad**, **criterio**. Rota la funcionalidad.

---

## 📋 Enunciado

> El equipo de producto quiere que el **paciente** pueda **guardar una búsqueda**: una combinación de filtros (especialidad, sede, médico, rango de fechas, orden) con un **nombre**, para reutilizarla rápido. El paciente debe poder ver sus búsquedas guardadas, **ejecutar** una (que lo lleva a los resultados ya filtrados) y eliminarla.
>
> Resolvé los siguientes puntos teniendo en cuenta la solución del TP.

---
---

# 1 · UI / FRONTEND

## 1.1 — Croquis y apariencia de las pantallas.

<br><br>

### ⬇️ RESPUESTA 1.1

Dos cosas. En la **pantalla de búsqueda** (modificada), cuando hay filtros aplicados aparece un botón "Guardar búsqueda" que abre un modal pidiendo un **nombre**. Y una pantalla **"Mis búsquedas guardadas"** (nueva): lista donde cada ítem muestra el nombre, un resumen de los filtros, y botones **"Ejecutar"** y **"Eliminar"**. Estados de carga / vacío / error.

🔎 **Qué busca el corrector:** el punto de entrada para guardar (desde la búsqueda con filtros) y la pantalla de listado con ejecutar/eliminar.

---

## 1.2 — Componentes nuevos y modificados.

<br><br>

### ⬇️ RESPUESTA 1.2

**Nuevos:** `GuardarBusquedaModal`, `ListaBusquedasGuardadas`, `BusquedaGuardadaItem`, y un `busquedaGuardadaService` (axios). **Modificados:** la pantalla de búsqueda (botón guardar) y la navbar (acceso a "Mis búsquedas").

🔎 **Qué busca el corrector:** distinguir nuevos vs modificados; reutilizar el modal de confirmación/formulario existente si aplica.

---

## 1.3 — Interacción al (a) guardar y (b) ejecutar una búsqueda guardada.

<br><br>

### ⬇️ RESPUESTA 1.3

**(a) Guardar:** con filtros aplicados, click en "Guardar" → modal pide nombre → `POST` → loader → si OK, confirma; si el nombre ya existe, muestra el error. **(b) Ejecutar:** en "Mis búsquedas", click en "Ejecutar" → se **cargan esos filtros** en la búsqueda y se dispara el `GET` de búsqueda de turnos → muestra los resultados. O sea, ejecutar **reutiliza** la búsqueda existente, no una nueva.

🔎 **Qué busca el corrector:** que ejecutar **reuse el endpoint de búsqueda** ya hecho (replay de los filtros), no que reimplemente la búsqueda.

---

## 1.4 — Accesibilidad + ¿esta pantalla la renderizarías del lado del cliente o del servidor?

<br><br>

### ⬇️ RESPUESTA 1.4

**Accesibilidad:** el modal de guardar es un form con `label` asociado al input del nombre (`label for` / `aria-label`), botones con texto claro, navegable por teclado, errores comunicados con texto (no solo color), `role="dialog"` + foco gestionado.

**CSR vs SSR (extra de teoría):** con Next.js podés renderizar del lado del **servidor (SSR)** o del **cliente (CSR)**. Para una pantalla **privada y dinámica** como "mis búsquedas" (depende del usuario logueado y cambia seguido), **CSR** es razonable: se renderiza el cascarón y se piden los datos con la sesión del usuario. **SSR** conviene para contenido **público y cacheable** o donde importe el SEO y el primer pintado (ej. una landing). Acá, al ser data del usuario tras login, CSR/render en cliente es lo natural.

🔎 **Qué busca el corrector:** accesibilidad del form/modal + saber qué es **CSR vs SSR** y elegir con criterio (privado/dinámico → cliente; público/SEO → servidor).

---
---

# 2 · BACKEND

## 2.1 — Rutas REST y autenticación.

<br><br>

### ⬇️ RESPUESTA 2.1

> 🔴 Tip de oro: auth siempre.

Con `Authorization: Bearer <JWT>` (valida dueño):

| Verbo | URL | Para qué |
|---|---|---|
| `POST` | `/pacientes/:id/busquedas-guardadas` | Crear. Body: `{ nombre, filtros: { especialidadId?, sedeId?, medicoId?, fechaDesde?, fechaHasta?, orden? } }`. → 201 |
| `GET` | `/pacientes/:id/busquedas-guardadas` | Listar las del paciente. → 200 |
| `DELETE` | `/pacientes/:id/busquedas-guardadas/:busquedaId` | Eliminar una. → 200/204 |

La **ejecución** no necesita endpoint nuevo: el front toma los `filtros` guardados y los manda como **query params** al `GET /turnos` que ya existe.

🔎 **Qué busca el corrector:** sub-recurso del paciente, el body con el objeto `filtros`, auth, y notar que **ejecutar reusa** la búsqueda existente (no se duplica lógica).

🟡 **Divague típico:** crear un `POST /ejecutarBusquedaGuardada` que reimplemente la búsqueda. **Respuesta justa:** se guardan los filtros y se replayean en el `GET /turnos` ya hecho.

---

## 2.2 — Lógica de la capa de servicios.

<br><br>

### ⬇️ RESPUESTA 2.2

`BusquedaGuardadaService.crear(pacienteId, { nombre, filtros })`: valida que el paciente exista (404); valida que **no exista ya una búsqueda con ese nombre** para ese paciente (409); persiste. El listar trae las del paciente. La lógica de **búsqueda en sí no se toca** acá: este service solo administra el guardado de criterios.

🔎 **Qué busca el corrector:** la guarda de unicidad por nombre, y la separación de responsabilidades (este service **no** ejecuta la búsqueda, solo guarda/lista criterios).

💡 **Profundizar (DTO / capa de mapeo):** lo que se guarda son los **criterios** (`filtros`). Cuando se ejecuta, el resultado del `GET /turnos` no es el documento crudo de Mongo: es un **DTO** armado por el `BusquedaTurnoService` que combina datos del turno + cobertura del plan + costo calculado. El DTO es la "vista" que la API expone, distinta del documento persistido y de la clase de dominio. Mapear dominio↔persistencia↔DTO es lo que mantiene desacopladas las capas.

---

## 2.3 — Gestión de errores.

<br><br>

### ⬇️ RESPUESTA 2.3

Paciente inexistente → **404**. Nombre de búsqueda ya usado por ese paciente → **409**. `filtros` mal formados / nombre vacío → **400** (Zod). Token ajeno → **403**. Tipados en el service, mapeo en el middleware.

🔎 **Qué busca el corrector:** **409** para el nombre duplicado, **400** para el body inválido, manejo centralizado.

---

## 2.4 — Ejemplo de test de integración.  🔴 *(tu tema flojo)*

<br><br>

### ⬇️ RESPUESTA 2.4

Con **Supertest** y el **repo mockeado**: `POST /pacientes/p1/busquedas-guardadas` con un nombre nuevo → **201**; un segundo POST con el **mismo nombre** para ese paciente → **409**. Verificás status + body; sin base real.

🔎 **Qué busca el corrector:** integración por HTTP, repo mockeado, happy (201) + error (409), AAA.

---

## 2.5 — Pseudocódigo de la guarda de unicidad por nombre.

<br><br>

### ⬇️ RESPUESTA 2.5

```js
async crearBusquedaGuardada(pacienteId, { nombre, filtros }) {
  await this.pacienteService.obtenerPorId(pacienteId);     // 404 si no existe

  // Unicidad: no dos búsquedas con el mismo nombre para este paciente
  if (await this.busquedaRepository.existePorNombre(pacienteId, nombre)) {
    throw new ConflictError("Ya tenés una búsqueda con ese nombre"); // 409
  }

  return await this.busquedaRepository.guardar({ pacienteId, nombre, filtros });
}
```

🔎 **Qué busca el corrector:** la guarda **consultar-antes-de-guardar**, scope correcto (unicidad **por paciente**, no global — dos pacientes pueden tener una búsqueda llamada igual).

---
---

# 3 · PERSISTENCIA  🔴 *(tu tema flojo)*

## 3.1 — Cambios al modelo. ¿Embebido o colección? Elegí y compará. *(Extra: ¿qué es un ODM y por qué Mongoose?)*

<br><br>

### ⬇️ RESPUESTA 3.1

**Embebido** (array dentro del Paciente): `busquedasGuardadas: [{ nombre, filtros: {...}, createdAt }]`. Cada búsqueda guardada es un **sub-documento** (no una referencia: los filtros son datos propios, no apuntan a otra entidad con vida propia). Comparación:

- **Embebido (elegido):** ✅ son **pocas** por paciente, se consultan **junto al paciente** ("mis búsquedas"), no tienen vida propia fuera de su dueño. ✅ una sola lectura. ❌ no se consultan globalmente (no es el caso de uso).
- **Colección aparte:** ✅ si necesitaras analizar búsquedas de **todos** los pacientes (métricas de qué se busca). ❌ overkill para algo que siempre se pide por paciente.

**ODM (extra):** Mongoose es un **ODM** (Object-Document Mapper): mapea tus objetos/clases de dominio a documentos de Mongo, agregando schema, validaciones y métodos sobre una base que es schema-less. Es el equivalente documental de un **ORM** (Object-Relational Mapper, ej. Hibernate) que mapea objetos a **tablas** SQL. La diferencia clave: el ORM pelea con tablas/joins/esquema rígido; el ODM mapea casi directo porque tus objetos ya se parecen a documentos JSON.

🔎 **Qué busca el corrector:** **embebido** con el criterio (pocas / se leen con el paciente / sin vida propia), comparar, y saber qué es un **ODM** y en qué se diferencia de un ORM.

🟡 **Divague típico:** decir "lo guardo como referencia" — no hay otra entidad a la que referenciar; los filtros son datos propios → sub-documento embebido.

---
---

# 4 · SOBRE LA FUNCIONALIDAD (CRITERIO)

## 4.1 — ¿Debería permitirse guardar dos búsquedas con el mismo nombre? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.1

No (dentro del mismo paciente). El nombre es lo que identifica la búsqueda para el usuario; dos iguales lo confunden y rompen la lógica de "ejecutar tal búsqueda". Sí pueden coexistir nombres iguales **entre distintos pacientes** (la unicidad es por dueño, no global).

🔎 **Qué busca el corrector:** **no** dentro del paciente + la sutileza del **scope** de la unicidad (por paciente).

---

## 4.2 — ¿Guardás los resultados de la búsqueda o los criterios? ¿Por qué?

<br><br>

### ⬇️ RESPUESTA 4.2

Los **criterios** (filtros), no los resultados. Los turnos disponibles **cambian todo el tiempo** (se reservan, se cancelan, se abren nuevos): si guardaras los resultados, quedarían **obsoletos** al instante. Guardando los criterios, cada vez que ejecutás la búsqueda obtenés el estado **actual**. Es la misma razón por la que un "filtro guardado" en cualquier app guarda el *qué buscar*, no el *qué encontró aquella vez*.

🔎 **Qué busca el corrector:** **criterios**, justificado por la **volatilidad** de los datos. Es el insight central del parcial.

---

## 4.3 — ¿Qué funcionalidades del TP se relacionan con esta?

<br><br>

### ⬇️ RESPUESTA 4.3

**Búsqueda de turnos:** ejecutar una búsqueda guardada **reutiliza** ese endpoint (replay de filtros). **Notificaciones:** extensión natural a futuro — alertar al paciente cuando aparezca un turno que matchea una búsqueda guardada.

🔎 **Qué busca el corrector:** la reutilización del endpoint de búsqueda; bonus si conectás con notificaciones.

---
---

## 🏁 Cierre

Contá los **🔎** que tocaste. Foco:
- Si en **4.2** dijiste "guardo los resultados" → ese es el error que el parcial busca; los datos son volátiles, se guardan criterios.
- Si en **1.4** no supiste **CSR vs SSR** o en **3.1** no supiste **ODM vs ORM** → son los extras de teoría, repasálos.
- Si reimplementaste la búsqueda en vez de reusarla → ojo con duplicar lógica.

**Cobertura acumulada (01-05):** REST + auth · capas · errores (404/409/422/400/401-403) · test de integración + pirámide · guardas: unicidad, transición, idempotencia · persistencia: embebido-refs, colección, sub-docs+soft delete, colección+populate, **sub-docs (05)** · accesibilidad: toggle, rating, modal, aria-live, **form/SSR-CSR** · CORS · **DTO/mapeo** · **ODM vs ORM** · **CSR vs SSR** · criterio/trade-offs.

**Sin tocar (para próximos):** validación de solapamiento de rangos · TDD/BDD · autorización por rol (médico vs paciente) · arquitecturas (MVC/MVVM, monolito vs microservicios/BFF) · índices.

**FIN DEL PARCIAL MODELO 05**
