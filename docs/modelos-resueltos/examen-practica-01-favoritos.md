# 📝 Examen de Práctica 1 — Desarrollo de Software (DDSW)

## Dominio: Sweet Medical (sistema de turnos médicos) · Tiempo sugerido: 90 min

> **Cómo usar este archivo.** Leé SOLO la Parte A (consigna), tapá la Parte B, y respondé al vuelo en papel como si fuera el parcial real (cronometrate los 90 min si podés). Recién cuando termines, destapá la Parte B y compará tu respuesta con la modelo y con el "qué busca el corrector". El objetivo no es que coincidan palabra por palabra, sino que **digas lo justo, sin divagar y sin irte de foco**.
>
> **Presupuesto de tiempo orientativo:** UI ~20 min · Backend ~35 min · Persistencia ~15 min · Funcionalidad ~12 min · repaso ~8 min.

---
---

# PARTE A — CONSIGNA

**Apellido y Nombre:** ………………………………  **Profesor:** ………………………………

En una reunión con el equipo de producto de **Sweet Medical**, surge la necesidad de desarrollar una nueva funcionalidad dentro de la plataforma.

La funcionalidad consiste en darle al **paciente** la posibilidad de marcar **médicos favoritos**, para poder hacer un seguimiento rápido de los profesionales que más le interesan. La funcionalidad estará disponible desde la **vista de detalle del médico**; si el médico ya pertenece a los favoritos, podrá quitarse de la lista.

Además, el paciente debe poder **consultar su lista de médicos favoritos**, teniendo en cuenta que de alguna forma se debe poder **discriminar/agrupar los médicos por especialidad**. Desde esa misma vista también debe ser posible quitar un médico de favoritos.

Teniendo en cuenta las tecnologías utilizadas y la solución realizada para el Trabajo Práctico, resolvé los siguientes puntos:

## UI / Frontend
- Dibujá un croquis/mockup y describí la apariencia de cada una de las pantallas de la UI que implementan las nuevas funcionalidades.
- Mencioná qué componentes nuevos serían necesarios y/o cuáles se modificarían de los existentes.
- Comentá cómo sería la interacción con la interfaz al momento de: (a) agregar un médico a favoritos; (b) consultar la lista de favoritos.
- Señalá los aspectos de accesibilidad que tendrías en cuenta para esta funcionalidad.

## Backend
- Enumerá las rutas REST (endpoints) que debería exponer el backend, y explicá brevemente para qué se usa cada una. Indicá verbos, URLs, bodies, path variables y query params.
- Describí la lógica que tendría la capa de servicios (orquestaciones, validaciones, servicios que intervienen, etc).
- Comentá cómo gestionarías los errores, en caso de ser necesario.
- Explicá un ejemplo de una validación de un test de integración sobre esta funcionalidad.
- Escribí el código o pseudocódigo de la validación que harías en la capa de servicio para asegurar que un médico no ingrese dos o más veces a la lista de favoritos.

## Persistencia
- Explicá los cambios al modelo de datos. Además, respondé: ¿se usará una nueva colección o estará embebido dentro de otro elemento? Elegí la solución y compará una contra la otra.

## Sobre la funcionalidad
- ¿Se debería permitir agregar 2 veces el mismo médico a favoritos? ¿Por qué?
- ¿Tiene sentido agregar a favoritos el mismo médico pero teniendo en cuenta que atiende en distintas sedes o distintas especialidades? ¿Por qué?
- ¿En qué otra/s funcionalidad/es ya desarrollada/s en el TP se debería tener en cuenta el contenido de la lista de favoritos?

---
---

# ⛔ FIN DE LA CONSIGNA — NO SIGAS LEYENDO HASTA HABER RESPONDIDO

```
                  ✋  Respondé en papel primero.
                      Después destapá lo de abajo.
```

---
---

# PARTE B — RESPUESTAS MODELO + CRITERIO DEL CORRECTOR

> Recordá: las respuestas de abajo están escritas **a registro de parcial** — cortas, al hueso, como las querría leer el corrector a mano en 90 minutos. Si tu respuesta fue mucho más larga, ese es tu primer dato: estás divagando.

---

## UI / FRONTEND

### Pantallas (croquis + apariencia)

**Respuesta modelo:**
Dos pantallas:
1. **Detalle del médico (existente, modificada):** se agrega un **botón toggle de favorito** (corazón/estrella) junto al nombre del médico. Lleno = ya es favorito; vacío = no lo es.
2. **Mis Favoritos (nueva):** lista de los médicos favoritos del paciente, **agrupada por especialidad** (un subtítulo por especialidad y debajo las cards de los médicos). Cada card muestra nombre, matrícula, especialidades y un botón "Quitar". Se accede desde un ítem nuevo en la navbar.

(Croquis: en el parcial alcanza un dibujo simple a mano de las dos pantallas con los elementos nombrados.)

> **Qué busca el corrector:** que identifiques que es **una pantalla nueva + una modificada**, que la lista **discrimine por especialidad** (lo pide la consigna), y que cada ítem tenga su acción de quitar. No busca que dibujes lindo.

### Componentes nuevos / modificados

**Respuesta modelo:**
- **Nuevos:** `BotonFavorito` (toggle reutilizable), página `MisFavoritos`, y `FavoritosContext` (estado global de la lista, para reflejar el favorito en cualquier pantalla sin prop drilling). Service nuevo `favoritoService` (axios).
- **Modificados:** `DetalleMedico` (suma el `BotonFavorito`), `Navbar` (suma el link a "Mis Favoritos"). Se reutiliza la card de médico existente sumándole la acción "Quitar".

> **Qué busca el corrector:** que distingas **nuevos vs. modificados**, y que justifiques el **Context** (estado compartido entre pantallas = el caso de useContext, no prop drilling).

### Interacción

**Respuesta modelo:**
- **Agregar a favoritos:** en el detalle del médico, el paciente toca el toggle → se actualiza el estado (UI optimista) y se dispara un `POST` al back → si responde OK queda marcado; si falla, se revierte el toggle y se muestra un mensaje de error.
- **Consultar favoritos:** desde la navbar entra a "Mis Favoritos" → se dispara el `GET` → mientras carga se muestra un **loader** → llega la lista agrupada por especialidad → puede quitar un médico con su botón (`DELETE`), que lo saca de la vista.

> **Qué busca el corrector:** que el flujo conecte con el back (no que sea solo visual) y que contemples los estados **loading/éxito/error**.

### Accesibilidad

**Respuesta modelo:**
- `aria-label` en el botón de favorito ("Agregar Dr. Pérez a favoritos" / "Quitar de favoritos") y `aria-pressed` para reflejar el estado del toggle.
- `alt` descriptivo en fotos de los médicos.
- Lista de favoritos como lista semántica (`<ul>/<li>`) con encabezados (`<h2>`) por especialidad.
- Navegable por teclado (foco visible, orden lógico) y contraste de color adecuado.

> **Qué busca el corrector:** la **primera regla de ARIA** en acción (HTML semántico primero; ARIA donde no alcanza) + al menos 3-4 aspectos concretos. Esto **se evalúa**, no lo saltees.

---

## BACKEND

### Rutas REST

**Respuesta modelo:**
| Verbo | URL | Para qué |
|---|---|---|
| `GET` | `/pacientes/{pacienteId}/favoritos` | Lista los favoritos del paciente. Query opcional `?especialidadId=` para filtrar/agrupar. |
| `POST` | `/pacientes/{pacienteId}/favoritos` | Agrega un médico. Body: `{ "medicoId": "..." }`. Devuelve 201. |
| `DELETE` | `/pacientes/{pacienteId}/favoritos/{medicoId}` | Quita un médico de favoritos. Devuelve 200/204. |

**Autenticación (en todas):** header `Authorization: Bearer <JWT>`. El backend verifica la firma del token y que el `pacienteId` del path coincida con el `id` del payload — así el paciente solo accede a **sus** favoritos y no a los de otro.

> **Qué busca el corrector:** recurso **anidado en plural** (`favoritos` como sub-recurso de `pacientes`), verbos correctos, y **la autenticación aclarada** (el tip de oro). Sin la auth, el punto queda flojo aunque las rutas estén bien.

> 🟡 **Divague típico (lo que NO conviene):** "Voy a hacer un endpoint `GET /traerFavoritosDelPaciente` que reciba el id y devuelva todos los médicos que el paciente marcó como favoritos, recorriendo la base de datos…" → ruta no-REST (verbo en la URL), te perdés explicando lo que ya se entiende, y **te olvidaste la autenticación**.
> **Respuesta justa:** `GET /pacientes/{id}/favoritos` + "auth con Bearer JWT, valida que el id sea el del token". Una línea por endpoint.

### Lógica de la capa de servicios

**Respuesta modelo:**
Un `FavoritoService` (o el `PacienteService` extendido) orquesta:
- **Agregar:** valida que el paciente exista (vía `PacienteService`) y que el médico exista (vía `MedicoService`); valida la **regla de unicidad** (que el médico no esté ya en favoritos); persiste.
- **Quitar:** valida que el médico esté en favoritos; lo saca.
- **Listar:** trae los favoritos del paciente; si vino `especialidadId`, filtra.

Servicios que intervienen: `PacienteService` y `MedicoService` (validar existencia). El service llama al repository; no toca HTTP.

> **Qué busca el corrector:** que la **lógica de negocio viva en el service** (no en el controller ni en el repo), que valides existencia de las dos entidades, y que aparezca la regla de **no duplicar**.

> 🟡 **Divague típico:** explicar línea por línea cómo Mongoose hace el push, mezclar la validación de formato (que va en el controller con Zod) acá, o describir el render del front. **Respuesta justa:** "valida paciente, valida médico, valida que no esté duplicado, persiste — usando PacienteService y MedicoService". Eso es todo.

### Gestión de errores

**Respuesta modelo:**
- Paciente o médico inexistente → **404 Not Found**.
- Médico ya en favoritos → **409 Conflict**.
- `medicoId` mal formado / body inválido → **400 Bad Request** (validación Zod en el controller).
- Token ausente/inválido o `pacienteId` que no es el del token → **401 / 403**.

El service tira errores de dominio (`NotFoundError`, `ConflictError`); el **middleware único de errores** los mapea a esos status. El controller solo hace `next(err)`.

> **Qué busca el corrector:** status codes correctos (especialmente **409** para el duplicado y **403/401** para la auth), y que el dominio **no conozca HTTP** (lo mapea el middleware).

### Ejemplo de test de integración

**Respuesta modelo:**
Levantar la app y mandar `POST /pacientes/{id}/favoritos` con un token válido y un `medicoId` válido → verificar que la respuesta sea **201**, que el favorito **quede persistido** (consultando el `GET` después aparece), y que un segundo `POST` del mismo médico devuelva **409**. Cubre el camino controller → service → repository completo.

> **Qué busca el corrector:** que sea un test que cruza capas (no unitario aislado), que valide **respuesta HTTP + estado persistido**, y bonus si testeás el caso de duplicado.

### Pseudocódigo de la validación de no-duplicado

**Respuesta modelo:**
```js
async agregarFavorito(pacienteId, medicoId) {
  const paciente = await this.pacienteService.obtenerPorId(pacienteId); // 404 si no existe
  await this.medicoService.obtenerPorId(medicoId);                      // 404 si no existe

  // Regla de unicidad: no permitir el mismo médico dos veces
  if (paciente.favoritos.includes(medicoId)) {
    throw new ConflictError("El médico ya está en favoritos"); // → 409
  }

  paciente.favoritos.push(medicoId);
  return await this.pacienteRepository.guardar(paciente);
}
```

> **Qué busca el corrector:** la **guarda explícita** antes de insertar (el `if includes → throw`). No busca un algoritmo complejo; busca que la unicidad se resuelva **en el servidor**, no confiando en el front.

---

## PERSISTENCIA

**Respuesta modelo:**
Cambio al modelo: el paciente necesita una lista de médicos favoritos. Dos opciones:

- **Embebido (recomendado acá):** un array `favoritos: [ObjectId<Medico>]` dentro del documento del **Paciente**. Guarda **referencias** (ids) a los médicos, no copias del médico. Pros: una sola lectura trae al paciente con sus favoritos; la lista pertenece exclusivamente al paciente; simple. Contras: si la lista creciera muchísimo, infla el documento (no es el caso de una lista de favoritos).
- **Colección aparte (referenciada):** colección `favoritos` con documentos `{ pacienteId, medicoId, fecha }`. Pros: escalable, podés sumar metadata (fecha en que lo marcó) y consultarla independientemente. Contras: requiere `populate`/lookup y más queries.

**Elijo embebido** (array de ids en el Paciente): la lista es acotada, pertenece solo al paciente y se lee junto con él. Importante: embebo **ids**, no el médico entero — el médico es una entidad propia que se actualiza por su cuenta.

> **Qué busca el corrector:** que **elijas y justifiques**, que **compares las dos** (lo pide la consigna), y la sutileza de **referenciar por id** en vez de duplicar la entidad médico. Cualquiera de las dos opciones es válida si la justificás bien.

> 🟡 **Divague típico:** describir las dos opciones en detalle sin comprometerte con ninguna, o explicar todo Mongoose de nuevo. **Respuesta justa:** una opción elegida + 2-3 pros/contras de cada una + la frase clave "embebo ids, no copias del médico".

---

## SOBRE LA FUNCIONALIDAD

### ¿Permitir agregar 2 veces el mismo médico?

**Respuesta modelo:**
No. Una lista de favoritos es un **conjunto**: el mismo médico dos veces no aporta nada y rompería el sentido de la funcionalidad (seguimiento). Se garantiza con la validación de unicidad en el service (y opcionalmente con un índice único en la base).

> **Qué busca el corrector:** un **no** rotundo + el **por qué** en una oración. No busca un párrafo.

### ¿Tiene sentido el mismo médico por distinta sede/especialidad?

**Respuesta modelo:**
No. El favorito es del **médico (la persona)**, no de una combinación médico-sede o médico-especialidad. La sede o la especialidad son datos para **agrupar/discriminar en la vista**, no parte de la identidad del favorito. (Es defendible lo contrario solo si el negocio decidiera que se favorea un "servicio puntual del médico" — pero por defecto, un favorito por médico.)

> **Qué busca el corrector:** que distingas **identidad del favorito** (el médico) de **criterio de presentación** (especialidad/sede). El "se podría justificar lo contrario" suma criterio, pero comprometé una postura.

### ¿Qué otras funcionalidades del TP se ven afectadas?

**Respuesta modelo:**
- **Búsqueda de turnos:** poder filtrar/destacar disponibilidades de médicos favoritos ("ver solo mis médicos").
- **Notificaciones:** avisar al paciente cuando un médico favorito libera una disponibilidad.
- **Mis turnos / detalle del médico:** mostrar el indicador de favorito de forma consistente.

> **Qué busca el corrector:** que conectes la feature nueva con el **resto del sistema ya hecho** (no que vivan aisladas). Con 2-3 alcanza.

---

## 📊 Cobertura de este examen / qué falta tocar

**Tocado fuerte:** REST + autenticación en endpoints · capas y validaciones por capa · manejo de errores (404/409/403) · test de integración · regla de unicidad en service · embedded vs referenced · accesibilidad/ARIA · React (Context, loading/error, componentes) · criterio de negocio.

**Todavía NO tocado (para próximos exámenes):** idempotencia explícita (PUT vs POST) · ACID/CAP · ORM vs ODM · pirámide de tests / TDD-BDD / caja blanca-negra · CORS · CSR vs SSR · arquitecturas de front (MVC/MVVM/Flux) · useEffect/useReducer en detalle · status codes 200/201/500 en otros contextos · versionado / ambientes / metodologías · monolito/microservicios/BFF.

---

**FIN DEL EXAMEN DE PRÁCTICA 1**
