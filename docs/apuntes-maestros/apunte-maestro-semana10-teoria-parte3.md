# 🌐 Apunte Maestro — Semana 10 · Teoría · Parte 3
## CORS: el error que te va a saltar al conectar el front con el back

> **Dónde estás parado.** Esto es un tema corto y autocontenido, distinto de las sesiones de las Partes 1 y 2. Pero es **el que más probablemente te frene en la Entrega 3**: apenas tu frontend intente pegarle a tu backend, hay buenas chances de que te aparezca un error de CORS. La buena noticia es que se entiende rápido y se resuelve en una línea. Y, a diferencia de las sesiones, **CORS sí puede aparecer en el parcial.**

---

## 1. ¿Qué es CORS? 🔴

Empecemos por desarmar la sigla, porque ayuda: **CORS = Cross-Origin Resource Sharing** (compartir recursos entre orígenes distintos).

En la práctica, la primera vez que te lo cruzás, **CORS es un error**. Es lo que pasa cuando tu página intenta pedir un recurso (hacer un request) a un servidor que está en un **origen distinto** al de la página.

Y acá está la clave que mucha gente no entiende: **CORS no es un problema de tu servidor. Es una política de seguridad del navegador.** El servidor *sí* recibe el request y *sí* responde — es el **browser** el que, al ver que la respuesta viene de otro origen y no trae los permisos adecuados, **la bloquea antes de entregártela a tu JavaScript.**

```
   ┌─────────────────┐         request         ┌─────────────────┐
   │   mipagina.com  │ ──────────────────────► │  mibackend.com  │
   │   (tu frontend) │                         │  (tu backend)   │
   │                 │ ◄────────  ✗  ────────── │                 │
   └─────────────────┘     CORS (bloqueado      └─────────────────┘
                            por el browser)
```

> ⚠️ **Una precisión (clase vs. realidad), porque importa para entenderlo bien.** En la explicación oral se dijo que "el request no se hizo, se bloqueó del lado del browser". Eso es cierto desde el punto de vista de tu código (tu JS nunca recibe la respuesta), pero la PPT lo dice con más precisión: **"el server realmente responde, el browser bloquea"**. O sea: para un request simple, el pedido *sí* llega al servidor y el servidor *sí* contesta — lo que el navegador bloquea es que esa respuesta llegue a tu código. **Para el parcial alcanza con la idea central: CORS es una política de seguridad del navegador que bloquea respuestas entre orígenes distintos cuando faltan los permisos.**

---

## 2. ¿Por qué ocurre? La idea de "origen" 🔴

Un **origen** se define por tres cosas: **protocolo + dominio + puerto**. Si cualquiera de las tres difiere entre tu front y tu back, para el navegador son **orígenes distintos**, y entra en juego la política de CORS.

El caso de manual con dominios:

```
Front:  https://mipagina.com
Back:   https://mibackend.com     ← dominio distinto → CORS
```

Pero el caso que **te va a pasar a vos en el TP** es con puertos en `localhost`:

```
Front:  http://localhost:5173     ← Vite (React) levanta acá por defecto
Back:   http://localhost:3000     ← tu Express levanta acá
```

Mismo `localhost`, mismo protocolo... pero **puerto distinto** (`5173` vs `3000`). Para el navegador, eso es suficiente para considerarlos orígenes diferentes. Por eso, en cuanto tu React en el `5173` le quiera pegar a tu Express en el `3000`, salta el error.

---

## 3. Cómo se ve el error 🟡

Conviene que lo reconozcas para no perder tiempo cuando aparezca. Tiene una pinta muy particular:

- **En la pestaña Network** del navegador: el request aparece **fallado, sin un response HTTP normal** (no hay un 200, ni un 404, ni un 500). Vas a ver algo tipo `net::ERR_FAILED`. Esto es distinto de un error del servidor: no es que el back te contestó mal, es que el browser cortó la entrega.
- **En la consola**: ahí tenés el mensaje útil, algo como *"...has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present..."*, y te aclara el origen desde el que se intentó (`from origin http://localhost:5173`).

La pista de que es CORS y no otra cosa es justamente que **el error habla de "CORS policy" y de un header `Access-Control-Allow-Origin` que falta**. Ese header es, literalmente, el permiso que el navegador esperaba y no encontró.

> 💡 **Ojo con Axios.** Dependiendo de cómo lo uses, Axios a veces maneja parte de esto y a veces no. Si al hacer clic en "buscar turnos" se te frena con un error de este tipo, sospechá de CORS antes que de tu código.

---

## 4. Cómo se resuelve: las tres formas 🔴

La solución va siempre **en el backend** (en tu `server.js`), porque es el server el que tiene que mandar el header de permiso que el navegador busca. El repo de ejemplo (`solucion-a-error-cors.txt`) te da tres maneras:

**Opción A — Middleware manual con `append`:**

```js
app.use((req, res, next) => {
  res.append('Access-Control-Allow-Origin', ['*']);
  res.append('Access-Control-Allow-Methods', ['GET']);
  next();
});
```

**Opción B — Middleware manual con `setHeader`:**

```js
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  next();
});
```

**Opción C — El paquete `cors` (la recomendada):**

```js
const cors = require('cors');
app.use(cors());
```

Las tres logran lo mismo: que el server agregue el header `Access-Control-Allow-Origin` que el navegador necesita para no bloquear la respuesta. **La opción C es la más limpia** — instalás el paquete (`npm install cors`) y con `app.use(cors())` ya está. El `'*'` significa "permito cualquier origen", que para el TP está perfecto.

Si querés afinar y permitir **solo tu frontend** (más cercano a lo que harías en producción), el paquete acepta opciones:

```js
app.use(cors({
  origin: 'http://localhost:5173'   // solo mi front puede pegarme
}));
```

> 🛠️ **Heads-up práctico con el repo.** En el `package.json` del backend de ejemplo, la dependencia figura como `"cors": "^2.8.6"`. Esa versión puede no existir en npm (la estable es la `2.8.5`), así que si al hacer `npm install` te tira *"no matching version found"*, cambiala a `"^2.8.5"` y listo.

---

## 5. Preflight (para que sepas que existe) 🟡

El profe lo mencionó al pasar y no profundizó, así que lo dejo igual: liviano.

Para ciertos requests "no simples" (por ejemplo, los que usan métodos como `PUT`/`DELETE` o ciertos headers personalizados), el navegador hace algo extra **antes** del request real: manda un pedido de tipo `OPTIONS` llamado **preflight**, que es como tocar la puerta y preguntar *"che, ¿me dejás hacer esta operación desde este origen?"*. Solo si el server responde que sí, el navegador manda el request de verdad.

La buena noticia: **si usás el paquete `cors`, todo esto lo maneja por vos.** No tenés que hacer nada especial. Lo importante es que el concepto te suene, por si lo ves en el parcial o en la consola.

---

## 6. El repo de ejemplo, en dos patadas

Para que ubiques las piezas cuando lo abras (es el repo `ejemplo-cors` que compartió la cátedra):

- **Backend** (`backend/server.js`): un Express en el puerto `3000` con un único endpoint `GET /api/profes` que devuelve una lista de nombres. **Viene a propósito SIN el fix de CORS**, para que veas el error primero.
- **Frontend** (Vite + React): una pantalla con un botón que, al hacer clic, dispara un `fetch('http://localhost:3000/api/profes')`. Como el front corre en `5173` y el back en `3000` → salta el CORS.
- **`solucion-a-error-cors.txt`**: las tres soluciones de arriba, para pegar en `server.js`.

> 📌 **Dos detalles del repo, por si los notás:**
> 1. El ejemplo usa `fetch` pelado para que el error sea bien evidente. En tu TP vas a usar **Axios** (el del interceptor de la Parte 2). El concepto de CORS y su solución son idénticos; solo cambia el cliente HTTP.
> 2. En el `server.js` del ejemplo, el `app.listen(3000)` está escrito *antes* de registrar la ruta. Funciona igual en Express, pero la forma canónica es registrar primero las rutas y los middlewares, y dejar el `listen` al final.

---

## Cierre

CORS en una frase: **es el navegador protegiéndote, bloqueando respuestas entre orígenes distintos hasta que el server diga explícitamente que ese origen tiene permiso.** Lo arreglás en el backend con el paquete `cors`, y se acabó.

> **Para el parcial, si te preguntan:**
> *¿Qué es CORS y cómo se soluciona?*
> Es una **política de seguridad del navegador** que bloquea las respuestas de requests hechos a un **origen distinto** (protocolo + dominio + puerto) al de la página. No es un error del servidor: el server responde, pero el browser bloquea la respuesta si no encuentra el header `Access-Control-Allow-Origin`. Se resuelve **en el backend**, haciendo que el server envíe ese header — en Express, lo más simple es el paquete `cors` con `app.use(cors())`.

### Checkpoint (respondé sin mirar)

1. ¿CORS es un error del servidor o del navegador? Justificá.
2. ¿Qué tres cosas definen un "origen"? Dado `localhost:5173` y `localhost:3000`, ¿por qué son distintos?
3. ¿Cómo distinguís un error de CORS de un 404 o un 500 mirando la consola y la pestaña Network?
4. Nombrá las tres formas de resolverlo en Express. ¿Cuál es la más recomendada y por qué?
5. ¿Qué es un preflight y por qué normalmente no tenés que ocuparte de él?

---

**FIN — Parte 3 · La info operativa (parcial, Entrega 3, fechas) está en la bitácora de la semana.**
