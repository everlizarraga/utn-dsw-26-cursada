# 🔐 Apunte Maestro — Semana 10 · Teoría · Parte 2
## Manejo de Sesiones: la implementación

> **Dónde estás parado.** En la Parte 1 armamos el modelo mental completo (HTTP stateless → JWT → localStorage → flujo → roles). Ahora lo bajamos a código. **Todo lo que sigue te lo podés llevar y pegar en tu TP con ajustes mínimos** — es el mismo código que el profe tiró en clase. Lo importante no es memorizarlo, es **entender qué hace cada bloque**, porque lo vas a tener que adaptar a tus rutas y tus nombres.
>
> **Buena noticia:** casi todo esto reusa patrones que ya conocés. El `SessionContext` es tu mismo Context del **contador de Lucas**, ahora aplicado a la sesión. El middleware de Express es primo hermano de tu **error handler** y tu **middleware de validaciones**: misma firma `(req, res, next)`. No es territorio nuevo, es territorio conocido con otro propósito.

### Dependencias que vas a necesitar

```bash
# En el frontend
npm install axios jwt-decode

# En el backend
npm install jsonwebtoken
```

---

## 1. `SessionContext.jsx` — el estado global de la sesión 🔴

El `Context` guarda al usuario en el **estado global de React**, leyendo el token de localStorage al iniciar. Cualquier componente de tu app va a poder preguntar "¿quién está logueado?" sin que le pases props por toda la jerarquía.

```jsx
// src/contexts/SessionContext.jsx
import { createContext, useContext, useState } from 'react';
import { jwtDecode } from 'jwt-decode';

const SessionContext = createContext(null);

export function SessionProvider({ children }) {

  // ── Estado: el usuario actual ──────────────────────────────
  // useState con "lazy init": la función de adentro corre UNA
  // sola vez, al montar el provider. Perfecta para leer
  // localStorage al arrancar y recuperar la sesión si ya existía.
  const [user, setUser] = useState(() => {
    const token = localStorage.getItem('token');
    if (!token) return null;                 // no había sesión
    try {
      const payload = jwtDecode(token);      // leemos { id, rol, nombre, exp }
      // ¿El token ya venció? exp viene en segundos; lo pasamos a ms.
      return payload.exp * 1000 < Date.now() ? null : payload;
    } catch {
      return null;                           // token corrupto → sin sesión
    }
  });

  // ── login: recibe el token y arma la sesión ────────────────
  const login = (token) => {
    localStorage.setItem('token', token);    // lo persistimos
    setUser(jwtDecode(token));               // y lo ponemos en el estado
  };

  // ── logout: borra todo, solo del lado cliente ──────────────
  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <SessionContext.Provider value={{ user, login, logout }}>
      {children}
    </SessionContext.Provider>
  );
}

// Hook custom: cualquier componente llama useSession()
// y obtiene { user, login, logout } sin importar el Context a mano.
export const useSession = () => useContext(SessionContext);
```

**Cuatro cosas que conviene que entiendas, no que memorices:**

- **`useState` con lazy init** 🟡 — fijate que a `useState` le pasamos *una función* (`() => {...}`), no un valor. React la ejecuta una sola vez, al montar. Si en cambio leyéramos el localStorage directo en cada render, sería un desperdicio. Como el provider envuelve toda la app y se monta una vez, este es el lugar perfecto para "recuperar" una sesión previa.
- **Manejo de expiración** 🔴 — comparamos `exp * 1000` con `Date.now()`. Si el token venció, devolvemos `null` y es como si no hubiera sesión. (El `exp` lo pone el backend al firmar el token; lo vemos abajo.)
- **`login()` solo recibe el token** 🟡 — no le pasás el usuario armado. Le das el token crudo que te devolvió el backend, y la función se encarga de decodificarlo y guardarlo. El componente del formulario, después del `POST /login`, solo hace `login(tokenQueVino)`.
- **El hook `useSession`** 🔴 — es azúcar para no repetir `useContext(SessionContext)` en cada componente. Recordá del contador de Lucas: el provider envuelve los componentes, y cualquiera adentro accede al valor compartido. Acá ese valor es `{ user, login, logout }`.

---

## 2. `utils/api.js` — el interceptor de Axios 🔴

Acá resolvemos algo que sería tedioso a mano: **pegar el token en cada request**. En vez de acordarte de agregar el header `Authorization` en cada llamada, configurás un **interceptor** una sola vez y se aplica a todos los requests que salgan por esta instancia de Axios.

```js
// src/utils/api.js
import axios from 'axios';

// Una instancia de axios con la URL base de tu backend.
const api = axios.create({ baseURL: 'http://localhost:3000/api' });

// ── Interceptor de REQUEST: agrega el token en CADA pedido ──
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;  // ← el patrón "Bearer <token>"
  }
  return config;   // hay que devolver la config, sí o sí
});

// ── Interceptor de RESPONSE: maneja el 401 (token vencido) ──
api.interceptors.response.use(
  null,                              // no tocamos las respuestas OK
  (error) => {
    if (error.response?.status === 401) {
      // el backend rechazó el token → cerramos sesión y al login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);    // re-propagamos el error
  }
);

export default api;
```

A partir de acá, en vez de `axios.get(...)` usás `api.get(...)`, y el token se engancha solo. El header que arma es `Authorization: Bearer <token>` — esa palabra **`Bearer`** ("portador") es la convención estándar para mandar tokens, y el backend la va a buscar exactamente así.

> 🟡 **El doble interceptor.** El de *request* es el que mete el token de salida. El de *response* es la red de seguridad de entrada: si en algún momento el backend te contesta `401` (token vencido o inválido), te saca la sesión y te manda al login automáticamente, sin que tengas que chequearlo en cada componente.

---

## 3. `middleware/auth.js` — verificar el token en Express 🔴

Del lado del backend necesitamos el guardia que **verifica el JWT** antes de dejar pasar al handler. Y acá viene lo que te decía: es **el mismo patrón de middleware que ya usás**.

```js
// middleware/auth.js  (Express)
const jwt = require('jsonwebtoken');

const verificarToken = (req, res, next) => {
  const auth = req.headers.authorization;   // "Bearer eyJhb..."

  // 1) ¿Vino el header y arranca con "Bearer "?
  if (!auth?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token requerido' });
  }

  try {
    // 2) Separamos el token del "Bearer " y verificamos la firma.
    const payload = jwt.verify(
      auth.split(' ')[1],          // el token, sin la palabra Bearer
      process.env.JWT_SECRET       // el secreto, desde variables de entorno
    );
    req.user = payload;            // dejamos el usuario disponible para el handler
    next();                        // todo OK, seguí
  } catch {
    // firma inválida o token vencido → jwt.verify tira excepción
    res.status(401).json({ error: 'Token inválido' });
  }
};
```

Y así lo **enchufás en una ruta protegida** — exactamente como enchufás tu middleware de validaciones:

```js
// El verificarToken corre ANTES del handler. Si el token no sirve,
// responde 401 y el handler ni se ejecuta.
router.get('/mis-turnos', verificarToken, async (req, res) => {
  // gracias a req.user (que puso el middleware), sabemos quién pide
  const turnos = await TurnoService.findByPaciente(req.user.id);
  res.json(turnos);
});
```

> **Recordá que** un middleware en Express es una función `(req, res, next)` que se ejecuta *antes* del handler de la ruta. Tu error handler y tu middleware de validaciones funcionan igual. La única diferencia es el trabajo que hace: este lee el header `Authorization`, valida el JWT con `jwt.verify`, y si está todo bien llama a `next()` para seguir; si no, corta con un `401`. Fijate también que **deja el usuario en `req.user`**, así el handler sabe quién está pidiendo sin volver a decodificar nada.

---

## 4. `ProtectedRoute.jsx` — proteger rutas en el frontend 🔴

Del lado de React, queremos que ciertas pantallas solo se rendericen si hay sesión y si el rol coincide. Eso lo hace un componente envoltorio: `ProtectedRoute`.

```jsx
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useSession } from '../contexts/SessionContext';

export function ProtectedRoute({ children, rolRequerido }) {
  const { user } = useSession();

  // 1) Sin sesión → al login
  if (!user) {
    return <Navigate to="/login" replace />;
  }

  // 2) Hay sesión, pero el rol no es el que pide la ruta → no autorizado
  if (rolRequerido && user.rol !== rolRequerido) {
    return <Navigate to="/unauthorized" replace />;
  }

  // 3) Todo OK → renderizamos lo que envuelve
  return children;
}
```

Y se usa en el armado de rutas, envolviendo cada zona privada:

```jsx
// src/App.jsx
export default function App() {
  return (
    <SessionProvider>            {/* la sesión disponible para toda la app */}
      <BrowserRouter>
        <Routes>

          {/* Pública: cualquiera entra */}
          <Route path="/login" element={<LoginPage />} />

          {/* Solo pacientes */}
          <Route path="/paciente/*" element={
            <ProtectedRoute rolRequerido="paciente">
              <DashboardPaciente />
            </ProtectedRoute>
          } />

          {/* Solo médicos */}
          <Route path="/medico/*" element={
            <ProtectedRoute rolRequerido="medico">
              <DashboardMedico />
            </ProtectedRoute>
          } />

        </Routes>
      </BrowserRouter>
    </SessionProvider>
  );
}
```

Leelo de afuera hacia adentro: `SessionProvider` envuelve todo (igual que el provider del contador envolvía la app), `BrowserRouter` maneja la navegación, y cada `Route` privada está envuelta en un `ProtectedRoute` que primero chequea sesión y rol. Si un paciente intenta entrar a `/medico/algo`, el `rolRequerido="medico"` no coincide con su `user.rol`, y lo manda a `/unauthorized`.

> ⚠️ **La advertencia más importante de toda la clase.** `ProtectedRoute` protege la **interfaz**, no los **datos**. Si alguien borra el componente desde las DevTools, o le pega directo a tu API con Postman saltándose tu frontend, `ProtectedRoute` no lo frena — porque vive en el navegador de quien ataca. **El que de verdad protege los datos es el `verificarToken` del backend**, que valida el JWT en cada endpoint. El front es UX; la seguridad es del back. Por eso ambos: ocultás en el front Y validás en el back.

---

## 5. La estructura completa + checklist mínimo

Así queda armado el proyecto del lado del front (tu TP puede usar otros nombres, esto es la forma):

```
src/
├── contexts/
│   └── SessionContext.jsx      ← proveedor global de la sesión
├── components/
│   └── ProtectedRoute.jsx      ← guardián de rutas
├── utils/
│   └── api.js                  ← axios + interceptor
├── pages/
│   ├── LoginPage.jsx           ← pública
│   ├── paciente/
│   │   ├── DashboardPaciente.jsx
│   │   ├── BuscarTurnos.jsx
│   │   └── Carrito.jsx         ← ¡solo estado de React! (sin backend)
│   └── medico/
│       ├── DashboardMedico.jsx
│       └── Agenda.jsx
└── App.jsx                     ← Routes + SessionProvider
```

**Checklist mínimo para que la sesión funcione punta a punta:**

- [ ] El `POST /login` devuelve `{ token }`, con el `rol` adentro del payload.
- [ ] `SessionContext` lee y guarda el token en localStorage.
- [ ] Se maneja el token expirado al inicializar (el chequeo de `exp`).
- [ ] Axios tiene el interceptor de **request** (agrega `Authorization`).
- [ ] Axios tiene el interceptor de **response** (redirige al login en `401`).
- [ ] `ProtectedRoute` verifica sesión y rol.
- [ ] **El backend también valida el JWT en cada endpoint.** ← el que más se olvida.

> 💡 **Truco para probar la expiración:** cuando lo implementes, ponele un tiempo de vida cortito al token (ej.: 30 segundos) y verificá que, una vez vencido, te mande de nuevo al login. Es la forma más rápida de comprobar que toda la cadena de expiración → 401 → redirect funciona.

---

## 6. Errores comunes (y cómo no caer en ellos) 🔴

Estos son los cuatro tropezones clásicos. Los marco porque son justo el tipo de detalle que se evalúa y que en producción te puede costar caro:

| ✗ El error | ✓ Lo correcto | Por qué |
|---|---|---|
| Solo ocultar botones en el frontend | El backend **también** valida el rol | Cualquiera puede llamar a la API directo. El front es UX, no seguridad. |
| Guardar datos sensibles en el Context | Guardar solo `id`, `rol` y `nombre` | El Context vive en el cliente. Nunca el JWT completo ni el historial médico. |
| Commitear el `JWT_SECRET` al repo | Usarlo desde `.env` y agregar `.env` al `.gitignore` | Si el secreto se filtra, cualquiera puede **forjar tokens válidos** y entrar como quien quiera. |
| No manejar el token expirado | Verificar `exp` al iniciar y capturar el `401` | El JWT tiene `expiresIn`. Pasado ese tiempo el backend lo rechaza con 401. |

El hilo conductor de los cuatro es el mismo principio de la Parte 1: **la seguridad real está en el backend.** El front colabora con la experiencia (no te muestra lo que no podés ver), pero la barrera que de verdad importa la pone el server validando cada request.

> **Para el parcial, si te preguntan** *(esto lo remarcó el profe — es de los que más se olvidan en finales):*
> Cuando te pidan diseñar un **endpoint REST**, no alcanza con la URL y el verbo. **Siempre aclará cómo se autentica al usuario.** Por ejemplo, si te piden "una ruta para acceder a mis reservas":
> ```
> GET /usuarios/{userId}/reservas
> Autenticación: header Authorization: Bearer <JWT>
>   (el backend verifica el token y que el userId coincida con el del payload)
> ```
> Lo que el corrector busca es que respondas la pregunta implícita: *¿cómo sabe el servidor que ese usuario está accediendo a lo suyo y no a lo de otro?* Mucha gente da el endpoint y se olvida de la autenticación. No seas esa persona.

---

## Cierre y puente a la Parte 3

Ya tenés el modelo mental (Parte 1) y el código que lo implementa (esta parte). Si tuvieras que reconstruir todo de memoria, el esqueleto es:

1. **`SessionContext`** guarda al usuario, lee el token al iniciar, expone `login` / `logout` vía `useSession`.
2. **`api.js`** (Axios) mete el token en cada request y maneja el 401 de salida.
3. **`verificarToken`** (Express) valida el JWT en el backend, igual que tu middleware de validaciones.
4. **`ProtectedRoute`** protege la UI por sesión y rol — pero el backend manda.

La **Parte 3** cambia de tema por completo: **CORS**, ese error que te va a saltar apenas conectes tu frontend con tu backend en la Entrega 3. Es corto y se resuelve fácil, pero conviene entender *por qué* pasa.

### Checkpoint (respondé sin mirar)

1. ¿Para qué sirve el "lazy init" de `useState` en el `SessionContext`?
2. ¿Qué hacen, por separado, el interceptor de request y el de response de Axios?
3. ¿Qué hace el middleware `verificarToken` y por qué deja el usuario en `req.user`?
4. ¿Qué pasa si un paciente intenta entrar a una ruta de médico, paso por paso?
5. ¿Por qué `ProtectedRoute` no es suficiente para la seguridad?
6. Nombrá los cuatro errores comunes y su corrección.
7. Al diseñar un endpoint REST en el parcial, ¿qué no te podés olvidar de aclarar?

---

**FIN — Parte 2 · Continúa en la Parte 3: CORS**
