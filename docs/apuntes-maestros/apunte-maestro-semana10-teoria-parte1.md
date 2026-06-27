# 🔐 Apunte Maestro — Semana 10 · Teoría · Parte 1
## Manejo de Sesiones: el modelo mental

> **Dónde estás parado.** Esta clase tiene dos temas grandes e independientes: **sesiones/autenticación** (lo pesado) y **CORS** (corto). Las sesiones las repartimos en dos partes:
> - **Parte 1 (esta):** el *qué* y el *por qué*. El sistema completo en la cabeza, sin ahogarte en código todavía.
> - **Parte 2:** el *cómo*. Todo el código React + Express, comentado bloque por bloque.
>
> CORS va en la Parte 3, y la info operativa (parcial, Entrega 3, fechas) en la bitácora aparte.
>
> **Aviso importante de la cátedra:** las sesiones **no son obligatorias para la Entrega 3** — son un *nice to have* y se vuelven obligatorias en la Entrega 4. Si necesitás moquear, moqueá tranquilo. Esto se da ahora para que el concepto esté maduro y para que tengas el código listo cuando lo necesites. Lo evaluable acá es **el concepto**, no tu implementación puntual del login.

---

## 1. El problema de fondo: HTTP es *stateless* 🔴

Arranquemos por la pregunta que justifica todo lo demás: **¿cómo hace un servidor para saber quién sos?**

La respuesta incómoda es que, por defecto, **no tiene la menor idea**. HTTP es un protocolo **sin estado** (*stateless*): cada request es completamente independiente del anterior y del siguiente. El servidor atiende tu pedido, te responde, y se olvida de vos por completo. No te "recuerda" entre un request y otro.

> **Recordá que** (semana 2) un request HTTP es un mensaje autocontenido: método (GET, POST...), URL, headers y body. Esa autocontención es justamente el problema: si el request no trae quién sos, el servidor no lo sabe de ningún otro lado.

Veámoslo con un caso concreto. Ana entra a la app y quiere ver sus turnos:

**Sin sesión:**

```
Request 1 →  POST /login   (email + contraseña)
Servidor   ←  "OK, sos Ana, te valido" ✓

Request 2 →  GET /mis-turnos
Servidor   ←  "¿Quién sos? No tengo idea." ✗
```

El servidor validó a Ana en el Request 1, pero en el Request 2 ya se olvidó. Para él son dos desconocidos distintos. No hay ningún hilo que conecte un request con el otro.

**Con sesión:**

```
Request 1 →  POST /login   (email + contraseña)
Servidor   ←  "OK, tomá este token que te identifica" ✓

Request 2 →  GET /mis-turnos   + el token
Servidor   ←  "¡Hola Ana! Tus turnos son..." ✓
```

La diferencia es que ahora **cada request trae consigo una credencial** (el token) que dice quién es Ana. El servidor no necesita "recordar" nada: la prueba de identidad viaja en cada pedido.

**La idea clave en una frase:** la sesión es el mecanismo que le da **memoria** al servidor entre requests. Y como el servidor sigue siendo stateless, esa "memoria" no vive en el servidor — **viaja con el cliente, en cada request.**

> **Para el parcial, si te preguntan:**
> *¿Por qué HTTP necesita un mecanismo de sesiones?*
> Porque HTTP es **sin estado**: cada request es independiente y el servidor no recuerda nada de los requests anteriores. Para identificar al usuario entre distintos requests, se usa una credencial (un token) que el cliente envía en cada pedido. Así el servidor sabe quién está haciendo cada request sin tener que "acordarse".

---

## 2. La herramienta: el JWT 🔴

La credencial que vamos a usar es un **JWT** (*JSON Web Token*). Algunos lo vieron de pasada en Diseño.

Un JWT es, simplemente, **un string firmado que el backend te genera cuando hacés login**. Lo guardás, y lo mandás en cada request posterior para probar quién sos.

Se ve así de feo:

```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6InUxIiwicm9sIjoicGFjaWVudGUiLCJub21icmUiOiJBbmEifQ.dBjftJeZ4CVP...
```

Esas tres tiras separadas por puntos son **las tres partes del token**:

| Parte | Qué es | Contenido |
|---|---|---|
| **Header** | El algoritmo usado para firmar | Ej.: `HS256` (HMAC-SHA256) |
| **Payload** | Los datos del usuario | `{ "id": "u1", "rol": "paciente", "nombre": "Ana" }` |
| **Firma** | La verificación de que nadie lo tocó | Un hash calculado con un secreto que solo el backend conoce |

El **payload** es la parte que te importa a vos como desarrollador. Fijate que ahí viaja el **`rol`**: con solo leer el token ya sabés si quien hace el request es paciente o médico, sin tener que ir a preguntarle a la base de datos. Eso es lo que después te va a dejar decidir qué puede y qué no puede hacer cada uno.

### Una sutileza que conviene tener clara desde el día uno 🟡

El JWT está **firmado**, no **encriptado**. Son cosas distintas y la diferencia importa:

- **Firmado** significa que el backend puede verificar que el token es legítimo y que nadie lo modificó. Si alguien le cambia una letra al payload (por ejemplo, se autoasciende de `paciente` a `medico`), la firma deja de coincidir y el backend lo rechaza. La firma garantiza **integridad y autenticidad**.
- Pero el payload **no es secreto**: está apenas codificado en Base64, así que *cualquiera* que tenga el token puede leer su contenido (de hecho podés pegarlo en [jwt.io](https://jwt.io) y verlo). La firma **no** garantiza confidencialidad.

> ⚠️ **Nota — clase vs. realidad:** En la clase se habló del JWT como si usara "un algoritmo de encriptación". En rigor, el HS256 es un algoritmo de **firma/hash**, no de cifrado: el payload se puede leer. **Para el examen alcanza con saber que el token está firmado y que la firma lo protege de modificaciones.** Pero la consecuencia práctica es la que sigue, y esa sí no la pierdas de vista.

La consecuencia directa: **nunca pongas datos sensibles en el payload.** Va el `id`, el `rol` y el `nombre`, y poco más. Nunca el password, ni el historial médico completo, ni nada que no quieras que un tercero lea. (Sobre esto volvemos en la Parte 2, en los errores comunes.)

> **Para el parcial, si te preguntan:**
> *¿Qué es un JWT y qué información lleva?*
> Es un **JSON Web Token**: un string firmado que el backend genera al hacer login. Tiene tres partes: **header** (algoritmo de firma), **payload** (datos del usuario, típicamente `id`, `rol` y `nombre`) y **firma** (verifica que el token no fue alterado). El cliente lo envía en cada request para identificarse.

---

## 3. ¿Dónde se guarda el token? localStorage vs. cookie 🟡

Una vez que el backend te dio el JWT, ¿dónde lo guardás del lado del cliente? Hay dos opciones, y la cátedra recomienda una para el TP.

| | **localStorage** *(recomendado para el TP)* | **Cookie httpOnly** |
|---|---|---|
| **Cómo se accede** | El JavaScript lo lee con `localStorage.getItem('token')` | El browser la manda solo, automáticamente |
| **Visible desde JS** | Sí (lo leés con código) | No (el JS no puede tocarla) |
| **Persistencia** | Sobrevive a recargas de página | Sobrevive a recargas de página |
| **Complejidad** | Simple de implementar | Requiere configurar CORS en Express |
| **Seguridad** | Más expuesta a XSS | Más segura contra XSS |

El trade-off real es de **seguridad**. Como el localStorage es accesible desde JavaScript, si alguien logra inyectar un script malicioso en tu página (un ataque **XSS**) o se mete en el medio de la comunicación (**man-in-the-middle**), podría leer ese token y robar la sesión. La cookie `httpOnly` es más segura justamente porque el JS no puede leerla.

**¿Y por qué la cátedra recomienda localStorage igual?** Porque para el TP el plus de seguridad de la cookie no justifica la complejidad extra (entre otras cosas, te obliga a configurar bien CORS). Con localStorage el código te queda más simple y directo, y para el alcance del trabajo está perfecto. En un sistema productivo real la decisión sería más matizada, pero para esto: **localStorage y a otra cosa.**

---

## 4. El flujo completo: login → token → request → logout 🔴

Ahora juntemos las piezas. Este es el corazón de la clase: el recorrido completo de una sesión, de punta a punta. Seguilo como una película.

### Acto 1 — Login (una sola vez)

```
1. Usuario        Completa el formulario (email + contraseña) y aprieta "Entrar"
       │
2. React          POST /login  con { email, password }
       │
3. Express        Valida las credenciales y, si están bien,
       │          GENERA un JWT con el rol adentro  →  { id, rol, nombre }
       │
4. localStorage   Guarda el token:  setItem('token', jwt)
       │
5. React Router   Redirige según el rol del payload:
                    rol = paciente  →  /paciente/...
                    rol = medico    →  /medico/...
```

Listo el login. A partir de acá el token vive en el localStorage del navegador.

### Acto 2 — Cada request autenticado (todas las veces siguientes)

```
React     axios.get('/mis-turnos')
   │      Un "interceptor" agrega automáticamente el header:
   │          Authorization: Bearer <token>
   │
Express   Recibe el request, verifica la firma del JWT
   │
   ├──  ✓ token válido  →  200 OK + los datos pedidos
   │
   └──  ✗ token inválido o vencido  →  401 Unauthorized
                                        → el front redirige al /login
```

Dos conceptos nuevos asoman acá y los desarrollamos en la Parte 2; por ahora quedate con la idea:

- El **interceptor** es un punto único por el que pasan todos tus requests de salida, donde "enganchás" el token sin tener que acordarte de pegarlo a mano en cada llamada. Lo configurás una vez y se aplica a todo.
- El **401** (Unauthorized) es el código que devuelve el backend cuando no estás autorizado — por ejemplo, porque tu token **venció**. A los tokens se les pone un tiempo de vida (expiración) para que no queden válidos para siempre; si quedó tu sesión abierta en otra compu, esa expiración limita el daño.

> **Recordá que** (semana 2) cada código HTTP tiene un significado: 200 = todo bien con datos, 401 = no autenticado/autorizado. Ese 401 es el que dispara la vuelta al login.

### Acto 3 — Logout

El logout es lo más simple de todo, y tiene una característica que sorprende: **vive enteramente del lado del cliente.**

```
1. Borrar el token:   localStorage.removeItem('token')
2. Limpiar el estado: setUser(null)
3. Volver al login:   navigate('/login')
```

No hace falta llamar al backend para cerrar sesión. ¿Por qué? Porque, como vimos, el servidor es stateless: no tiene una "sesión abierta" guardada que haya que cerrar. La sesión *es* el token, y el token vive en tu navegador. Si lo borrás del localStorage, la sesión dejó de existir. Punto.

---

## 5. Los roles en nuestro TP: Paciente vs. Médico 🔴

Todo lo anterior se vuelve concreto en el TP gracias a un solo campo del payload: **`rol`**, que vale `"paciente"` o `"medico"`. Ese campo es el que define **qué pantallas renderiza React y a qué tiene acceso cada usuario.**

| | **Paciente** | **Médico** |
|---|---|---|
| **Ruta base** | `/paciente/*` | `/medico/*` |
| **Vistas** | Buscar turnos disponibles · Mis turnos / historial · Notificaciones | Agenda del día · Administrar disponibilidad · Cancelar / modificar turnos · Historial del paciente |
| **Puede** | Reservar · Cancelar (hasta 1 h antes del turno) | Marcar turno como realizado · Cancelar / modificar turnos |
| **NO puede** | Ver la agenda del médico | Buscar turnos como paciente |

La lógica es directa: cuando un usuario intenta entrar a una ruta, se compara **el rol que pide la ruta** con **el rol que trae su token**. Si un paciente intenta colarse en `/medico/agenda`, los roles no coinciden y no entra. Esa comparación es lo que en la Parte 2 vamos a implementar con un componente llamado `ProtectedRoute`.

> **Una advertencia que vale oro (y volvemos a ella en la Parte 2):** proteger las rutas en el front es **UX, no seguridad**. El front decide qué *se muestra*; pero la seguridad de verdad la tiene que poner el **backend**, validando el JWT en cada endpoint. Si alguien borra el componente desde las DevTools o le pega directo a la API saltándose tu interfaz, el único que lo frena es el back. Grabate esto: **doble validación, siempre.**

> 🟢 **Nota menor de nomenclatura:** en la PPT las rutas aparecen como `/pacientes` y `/medicos` (plural), pero el código del ejemplo usa `/paciente` y `/medico` (singular). Es lo mismo conceptualmente; en este apunte uso **singular** para alinearme con el código, que es el que vas a copiar. Elegí una forma y sé consistente en todo tu proyecto.

---

## Cierre y puente a la Parte 2

Quedate con este mapa mental, que es todo lo que necesitás antes de tocar una línea de código:

1. **HTTP es stateless** → el servidor no te recuerda entre requests.
2. Por eso usamos un **JWT**: un token firmado, con `{ id, rol, nombre }` en el payload, que viaja en cada request y le da "memoria" al server.
3. Lo guardamos en **localStorage** (simple y suficiente para el TP).
4. El **flujo** es: login genera el token → cada request lo manda en el header `Authorization: Bearer` → el back lo verifica → logout lo borra del cliente.
5. El **`rol` del payload** decide qué ve y qué puede hacer cada usuario (paciente / médico).
6. La **seguridad real está en el backend**, no en ocultar botones.

En la **Parte 2** bajamos todo esto a código: vas a ver el `SessionContext` (que es tu mismo patrón de Context del contador de Lucas, ahora aplicado a la sesión), el interceptor de Axios, el middleware de Express (primo hermano de tu error handler y tu middleware de validaciones) y el `ProtectedRoute`. Todo comentado y listo para llevar.

### Checkpoint (respondé sin mirar)

1. ¿Por qué se dice que HTTP es *stateless* y qué problema genera eso?
2. ¿Qué es un JWT y cuáles son sus tres partes?
3. ¿Qué lleva el payload en nuestro TP y por qué es útil que incluya el `rol`?
4. ¿Por qué no hay que poner datos sensibles en el payload?
5. ¿Por qué la cátedra recomienda localStorage en vez de cookie httpOnly para el TP?
6. Contá el flujo completo de una sesión: del login al logout.
7. ¿Por qué el logout no necesita llamar al backend?
8. ¿Por qué proteger las rutas solo en el frontend no alcanza?

---

**FIN — Parte 1 · Continúa en la Parte 2: la implementación**
