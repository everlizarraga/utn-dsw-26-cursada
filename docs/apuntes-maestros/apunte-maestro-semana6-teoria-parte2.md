# Apunte Maestro — Semana 6 Teoría, Parte 2

## ODM, Mongoose y los tres modelos del TP

> **Semana 6 — Martes 05/05 — Profes: Agus (lidera) y Lucas (soporte)**
>
> **Sobre esta parte:** Segunda y última parte del apunte de teoría de la semana 6. La Parte 1 dejó el panorama conceptual: qué es persistencia, las dos familias de bases de datos, ACID, CAP. Esta Parte 2 baja todo eso al ODM concreto que vas a usar en el TP — **Mongoose** — con la demo en vivo que mostró Agus al final de clase. El núcleo de esta parte son **los tres modelos del TP2** (Request DTO / Dominio / Schema), que la cátedra dejó muy explícitos como condición de corrección. *"Esas tres cosas tienen que estar marcadas en el TP por una mínima diferencia, por favor"* — Agus, textual. Lo demás de esta parte (Mongoose básico, comportamientos a entender, feedback del TP1 que aplica al TP2) gira alrededor de eso.
>
> **Sobre la profundidad de Mongoose:** lo que se vio en clase es lo básico — Schema, Model, `create`, `find`, `findOne`. Embedded documents y referenced documents se tocaron muy por encima; ese tema entra de verdad en la clase práctica del sábado siguiente. Este apunte cubre lo que efectivamente se enseñó el martes; cuando llegues a la clase del sábado la información se complementa, no se pisa.

---

## 0. De dónde venís y dónde te deja esta clase

Venís de la Parte 1 con el panorama: las bases de datos no son una decisión única, son varias decisiones de diseño. La cátedra eligió **MongoDB** (documental, NoSQL, CP) para el TP. Ahora la pregunta práctica es: ¿cómo le habla tu código a Mongo?

La respuesta corta: con un **ODM** llamado Mongoose. La respuesta larga es esta parte del apunte.

Vas a salir de acá sabiendo:

1. Qué es un ODM y por qué es más simple que un ORM.
2. Cómo se define un Schema en Mongoose y cómo se usa.
3. Las operaciones básicas (`create`, `find`, `findOne`) y sus comportamientos puntuales.
4. **Cómo separar los tres modelos** — Request DTO, Dominio, Schema — que el TP2 va a evaluar fuerte.
5. Qué van a corregir en la entrega 2 que ya marcaron como flojo en la entrega 1.

---

## 1. ORM vs ODM 🟡

Antes de meternos en Mongoose, hay que entender qué tipo de herramienta es. Mongoose es un **ODM** (Object-Document Mapping). Eso solo cobra sentido si lo comparás con su primo más viejo, el **ORM** (Object-Relational Mapping).

### 1.1 Qué es un ORM (recordatorio del mundo SQL)

Un ORM mapea **objetos de tu lenguaje de programación** (clases, atributos, herencia) a **tablas relacionales** (filas, columnas, claves foráneas). El más conocido en el mundo Java es **Hibernate**. En .NET tenés Entity Framework. En Python, SQLAlchemy.

La idea: vos modelás tu dominio en clases con atributos, herencia, composición — y el ORM se encarga de traducir todo eso a SQL para guardarlo y recuperarlo. Vos no escribís `INSERT INTO usuarios (nombre, edad) VALUES (...)`. Escribís `usuarioRepository.save(usuario)` y el ORM lo traduce.

Pequeño recorrido por Hibernate (Java), para que tengas el contraste:

```java
// En Hibernate, marcás tu clase de dominio con anotaciones
// que le dicen al ORM cómo persistirla.

@Entity                          // "Esto es una tabla"
public class Usuario {

    @Id                          // "Esto es la primary key"
    private Long id;

    private String nombre;       // Columna automática del mismo tipo
    private Integer edad;

    @Transient                   // "Este atributo NO lo persistas"
    private String dataDeSesion; // (existe en memoria, no en la base)
}
```

Esto se ve lindo en clases simples, pero **se complica con herencia y composición**. Si tenés `class Medico extends Persona`, el ORM tiene que decidir cómo mapear esa jerarquía a tablas: ¿una sola tabla con todos los campos? ¿dos tablas con join? ¿una tabla por subclase? Hibernate te ofrece tres estrategias y vos elegís — ya te metiste en complejidad de mapeo. Las relaciones (`@OneToMany`, `@ManyToOne`, `@ManyToMany`) son otro mundo de configuración.

El ORM resuelve un problema real, pero **no es transparente**: tenés que estar consciente del mapeo, sus opciones y sus límites.

### 1.2 Qué es un ODM (lo que vamos a usar)

Un ODM mapea **objetos de tu lenguaje** a **documentos** (típicamente JSON o BSON) en una base documental. Mongoose es el ODM más usado para MongoDB en Node.

La gran diferencia respecto al ORM: **no hay impedancia**. Un objeto JavaScript se parece muchísimo a un documento JSON — porque JSON literalmente nació de la sintaxis de objetos JavaScript. No tenés que decidir cómo mapear herencia a tablas, no tenés que pelear con joins. El objeto que tenés en código se guarda casi igual.

```javascript
// Un objeto en código:
const usuario = {
  nombre: "Ana",
  edad: 30,
  direcciones: [
    { calle: "Av. Corrientes 1234", ciudad: "Buenos Aires" },
    { calle: "Av. Cabildo 5678",   ciudad: "Buenos Aires" }
  ]
};

// El documento que se guarda en Mongo (BSON):
// {
//   "_id": ObjectId("65f1a..."),
//   "nombre": "Ana",
//   "edad": 30,
//   "direcciones": [
//     { "calle": "Av. Corrientes 1234", "ciudad": "Buenos Aires" },
//     { "calle": "Av. Cabildo 5678",   "ciudad": "Buenos Aires" }
//   ]
// }
```

Casi idéntico. No hay tabla auxiliar de direcciones, no hay foreign key. El array vive **embebido** dentro del documento del usuario. Esa es la magia del modelo documental: la estructura del código se parece a la estructura del almacenamiento.

> **Mención importante:** la simplicidad de "todo embebido" tiene un costo y un límite. ¿Qué pasa si necesitás tener "usuarios" y "pedidos" como entidades independientes que se relacionan? Ahí entra la decisión **embedded vs referenced**, que se ve en detalle en la clase práctica del sábado siguiente. Para el martes, lo importante es el concepto base: en una base documental, podés guardar estructuras anidadas directamente, y eso simplifica el mapeo.

### 1.3 Tabla comparativa rápida

| Eje | ORM (Hibernate, etc.) | ODM (Mongoose) |
|---|---|---|
| Base de datos | Relacional (SQL) | Documental (Mongo) |
| Estructura destino | Tablas con esquema rígido | Documentos JSON/BSON, schemaless |
| Mapeo de objetos | Complejo (herencia, joins, FKs) | Casi directo (estructura similar) |
| Lenguaje de consulta | SQL (vía abstracción) | API del ODM (también queries directas) |
| Cambios de modelo | Costosos (migraciones, ALTER TABLE) | Baratos (agregás un campo y listo) |
| Aprendizaje | Mucho que entender | Poco que entender |

**Conclusión que dejó Agus en clase:** *"el ODM es muy fácil de pensar porque ya pensás en tus objetos y clases de dominio, vas a ver cómo va a ser ese esquema."* La estructura mental que ya tenés sirve.

---

## 2. Mongoose: el ODM de Node 🔴

Mongoose es la librería que vas a instalar en tu proyecto del TP para hablarle a Mongo. Lo que sigue es lo mínimo que necesitás para arrancar la entrega 2; lo profundo es la clase práctica del sábado.

### 2.1 Conexión a Mongo

Antes de definir nada, tu app necesita conectarse a una base Mongo concreta. Esto se hace una sola vez al arrancar el servidor:

```javascript
// archivo: db/connection.js (o similar, en tu repo)
const mongoose = require('mongoose');

// La URI tiene la forma: mongodb://[host]:[puerto]/[nombreDB]
// Para desarrollo local, suele ser:
//   mongodb://localhost:27017/sweetmedical
// En producción, viene de una variable de entorno.
const URI = process.env.MONGO_URI || 'mongodb://localhost:27017/sweetmedical';

mongoose.connect(URI)
  .then(() => console.log('Conectado a MongoDB'))
  .catch((err) => console.error('Error conectando a MongoDB:', err));
```

Detalles que importan:

- **`mongoose.connect()` devuelve una Promise.** Si te trabaste con esto en la entrega 1 (5 grupos no usaron promises bien según Agus), acá vuelve el tema: la conexión es asíncrona, tenés que manejarla con `.then/.catch` o `await`.
- **El nombre de la base** (`sweetmedical` en el ejemplo) lo elegís vos. Mongo la crea automáticamente la primera vez que insertás algo.
- **Una sola conexión por app.** No abras conexiones nuevas por cada request — Mongoose mantiene un pool internamente.

### 2.2 Definir un Schema

El Schema es la **descripción de cómo son los documentos de una colección**. A pesar de que Mongo es schemaless por naturaleza, Mongoose te permite (y te recomienda) definir un schema en código para poder validar y estructurar.

```javascript
// archivo: domain/schemas/MedicoSchema.js
const mongoose = require('mongoose');

const medicoSchema = new mongoose.Schema({
  // Cada campo se define con un tipo y, opcionalmente, opciones.

  nombre: {
    type: String,        // Mongoose usa los tipos básicos de JS:
                         //   String, Number, Boolean, Date, Array, ObjectId
    required: true       // Si lo intentás guardar sin nombre, REVIENTA.
  },

  matricula: {
    type: String,
    required: true,
    unique: true         // No puede haber dos médicos con la misma matrícula.
                         // Mongoose crea automáticamente un índice único.
  },

  email: String,         // Forma corta cuando solo necesitás el tipo
                         // (sin opciones extra). Equivale a { type: String }.

  edad: Number,

  fechaIngreso: {
    type: Date,
    default: Date.now    // Si no se manda, usa la fecha actual.
  },

  especialidades: [String]  // Array de strings.
                            // Para arrays de objetos se usa otra notación.
});

// El Model es la clase que se usa en el código real.
// Toma un nombre (que se vuelve la colección, en plural minúscula:
// 'Medico' → colección 'medicos') y el schema.
const Medico = mongoose.model('Medico', medicoSchema);

module.exports = Medico;
```

**Las cuatro opciones de campo más usadas:**

| Opción | Qué hace | Ejemplo |
|---|---|---|
| `type` | Define el tipo (String, Number, Date, Boolean, etc.) | `type: String` |
| `required` | Si es `true`, el campo es obligatorio. Sin él, falla al guardar. | `required: true` |
| `unique` | Crea un índice único: no puede haber dos documentos con el mismo valor. | `unique: true` |
| `default` | Valor por defecto si no se provee. | `default: Date.now` |

### 2.3 Operaciones básicas

Una vez tenés el Model, las operaciones son métodos sobre él. Las más usadas:

```javascript
// CREAR un documento (devuelve Promise)
const nuevoMedico = await Medico.create({
  nombre: 'Juan',
  matricula: 'M-12345',
  email: 'juan@hospital.com',
  edad: 45,
  especialidades: ['Cardiología', 'Clínica Médica']
});
// nuevoMedico ahora tiene _id, __v y todos los campos.


// BUSCAR TODOS los documentos que matcheen un criterio
// Devuelve un ARRAY (puede estar vacío).
const medicos = await Medico.find({});                    // Todos.
const cardiologos = await Medico.find({ 
  especialidades: 'Cardiología' 
});                                                        // Filtrados.


// BUSCAR UNO solo
// Devuelve el OBJETO directamente (o null si no encuentra).
const medico = await Medico.findOne({ matricula: 'M-12345' });
if (medico) {
  console.log(medico.nombre);
}


// BUSCAR POR ID (atajo común)
const medico = await Medico.findById(idDelMedico);


// ACTUALIZAR
await Medico.findByIdAndUpdate(idDelMedico, { 
  edad: 46 
});


// ELIMINAR
await Medico.findByIdAndDelete(idDelMedico);
```

> 🔴 **La diferencia entre `find` y `findOne` es importante:** `find` siempre devuelve un **array** (vacío si no hay matches). `findOne` devuelve un **objeto** (o `null` si no encuentra). Si te confundís, vas a tener errores tipo "cannot read property X of undefined" cuando trates un array como objeto.

### 2.4 Pregunta natural: ¿qué pasa con `findOne` si hay varios que matchean?

Sale el primero que encuentre la base, y "el primero" depende del **orden interno** que use Mongo en ese momento (típicamente orden de inserción si no hay índice especial, pero **no es determinístico** y puede cambiar si la base reorganiza datos).

**Implicación práctica:** `findOne` solo tiene sentido cuando estás buscando algo que sabés que es único (matrícula, email, `_id`). Si el criterio no garantiza unicidad y te importa cuál te devuelve, usá `find` y elegí vos cuál tomar (con un `.sort()`, por ejemplo).

---

## 3. La demo en vivo: tres comportamientos a entender 🔴

Agus prendió la app y mostró tres comportamientos puntuales que vale la pena tener registrados, porque te van a aparecer en el TP y si no los conocés perdés tiempo.

### 3.1 Si mandás un atributo que NO está en el schema → Mongoose lo IGNORA silenciosamente

```javascript
// El schema declara solo: nombre, apellido, username, email
const userSchema = new mongoose.Schema({
  nombre: String,
  apellido: String,
  username: { type: String, required: true, unique: true },
  email: String
});

const User = mongoose.model('User', userSchema);

// En el controller, llega un POST con un atributo extra: "color".
const userCreado = await User.create({
  nombre: 'Ever',
  apellido: 'Lizarraga',
  username: 'ever123',
  email: 'ever@test.com',
  color: 'azul'                  // ← este NO está en el schema
});

// userCreado.color === undefined
// La base solo guardó los 4 campos del schema. "color" desapareció.
```

Esto es **Mongoose protegiéndote**: si por error te llega basura en el body del request, no se persiste. La validación silenciosa es una defensa natural contra contaminar la base.

**Implicación para el TP:** *NO te confíes de que "lo que entró se guardó"*. Si esperabas que se guardara un campo y no aparece en la base, chequeá primero que ese campo esté declarado en el schema.

### 3.2 Si mandás un POST sin un campo `required` → REVIENTA con error 500

```javascript
// El schema marca username como required.
// Si llega un POST sin username:
const userCreado = await User.create({
  nombre: 'Ever',
  apellido: 'Lizarraga',
  email: 'ever@test.com'
  // falta username
});
// → ValidationError: User validation failed: username: Path `username` is required.
```

Si tu controller no tiene un `try/catch` que atrape esto y lo traduzca a un 422 (Unprocessable Entity) o 400 (Bad Request), el cliente recibe un **500 (Internal Server Error)**, que es lo peor: dice "se rompió algo en el servidor" cuando en realidad fue un error del cliente (mandó datos incompletos).

**Implicación para el TP:** las validaciones de schema son la última línea de defensa, no la primera. Lo ideal:

1. **Controller valida la forma con Zod** → si falla, 400.
2. **Service valida reglas de negocio** → si falla, 422.
3. **Schema de Mongoose valida estructura** → si llega algo mal acá, ya es un bug tuyo de capas anteriores. Atrapalo igual con `try/catch` y traducilo a 500 con mensaje útil, pero idealmente nunca debería disparar.

### 3.3 El campo `__v` y el versionado interno

Cuando creás un documento, Mongoose le agrega automáticamente un campo `__v` (versionKey).

```javascript
// Si imprimís un documento recién creado:
{
  _id: ObjectId("65f1a..."),
  nombre: "Ever",
  apellido: "Lizarraga",
  username: "ever123",
  email: "ever@test.com",
  __v: 0                       // ← agregado por Mongoose
}
```

Sirve para detectar conflictos de concurrencia: si dos procesos quieren actualizar el mismo documento al mismo tiempo, Mongoose puede usar `__v` para detectar que uno de los dos está laburando con una versión vieja y rechazar el segundo update.

Para el TP no lo vas a tocar explícitamente. Solo registrá que existe y que **no es un campo tuyo** — es metadata de Mongoose. Como con el `_id`, **no lo expongas en la respuesta de tu API**.

---

## 4. El `_id` y el ObjectId: por qué nunca lo exponés 🔴

Cada documento que guardás en Mongo tiene automáticamente un campo `_id` que es de tipo **ObjectId**. Es la "primary key" de Mongo.

```javascript
// Un documento real visto desde la consola de Mongo:
{
  _id: ObjectId("65f1a3b8c8e9f1234567890a"),  // 24 caracteres hex
  nombre: "Ever",
  ...
}
```

El ObjectId no es un número entero auto-incremental como en SQL. Es un identificador único que Mongo genera con una mezcla de timestamp, machine id, process id y un contador. Como dato curioso: se puede extraer la fecha de creación de un ObjectId, porque los primeros 4 bytes son el timestamp.

### 4.1 Por qué nunca exponer el `_id` afuera

Agus fue muy claro acá. **Lo importante es que esto nunca lo expongan en el modelo de dominio, en el modelo de respuesta de la API, ni en URLs**. ¿Por qué?

**Razón 1: seguridad — inyección NoSQL.** Si tu API tiene rutas tipo `GET /api/medicos/:id` donde `id` es directamente el `_id` de Mongo, un atacante puede manipular ese parámetro y hacer **inyecciones NoSQL**. Pasar `id` con una estructura `{ "$ne": null }` es un ejemplo clásico — si tu código no lo sanitiza bien, podés terminar devolviendo todos los médicos en vez de uno.

**Razón 2: acoplamiento.** Si exponés el `_id` y mañana migrás de Mongo a otra base con otro tipo de identificador (UUIDs, ints), rompés el contrato de tu API. Toda la integración con frontend o terceros se cae.

**Razón 3: filtración de información.** El ObjectId contiene timestamp y datos del servidor. Exponer eso es filtración mínima, pero filtración al fin.

### 4.2 Qué hacer en su lugar

Hay dos enfoques aceptables, ambos los mencionó Agus:

**Opción A: usar el `_id` internamente (en repository) pero NUNCA exponerlo.** Tu Service y tu Domain trabajan con un `id` que es la representación pública (un UUID que generás vos, o un id de negocio como la matrícula del médico). El Repository es el único que sabe que internamente se mapea a `_id`.

**Opción B: tener tu propio campo `id` además del `_id`.** Generás un UUID al crear el médico, lo guardás como `id` en el schema, y todas las búsquedas externas las hacés por `id`, no por `_id`. El `_id` queda como metadato interno de Mongo.

Para el TP probablemente alcance con la **Opción A**: usás el `_id` internamente, lo mapeás a `id` (string) cuando armás la respuesta, y no lo exponés con su nombre nativo.

```javascript
// En la capa Repository, cuando convertís un documento Mongo
// a una entidad de dominio:
function toDomain(doc) {
  return {
    id: doc._id.toString(),     // _id de Mongo → id de string para el dominio
    nombre: doc.nombre,
    matricula: doc.matricula,
    // ... el resto, SIN exponer _id ni __v
  };
}
```

---

## 5. Los TRES modelos del TP2 🔴🔴🔴

Esta es la sección más importante de toda la clase. Agus la dijo dos veces y Lucas la reforzó. Es **la condición de corrección** de la entrega 2 más mencionada explícitamente.

### 5.1 El planteo

En el TP1 vos tenías, probablemente, **dos representaciones** del médico (o de cualquier entidad):

- **El schema de Zod** que validaba lo que llegaba en el body del request.
- **La clase de dominio** (`Medico`) que vivía en la capa Domain y tenía la lógica de negocio.

Eso era suficiente cuando el Repository guardaba en memoria — el "objeto que va a la base" era exactamente el objeto de dominio. No había una tercera representación.

**Con persistencia real, aparece una tercera.** El objeto que va a Mongo tiene cosas que el objeto de dominio NO debería tener: el `_id`, el `__v`, decisiones de cómo se serializa, qué campos son embedded, qué campos son references. Mezclar ese objeto con el de dominio es lo que Agus llamó *"mezclar capas"* — uno de los errores más penalizados.

### 5.2 Los tres modelos en detalle

| Modelo | Capa donde vive | Para qué sirve | Qué tiene de particular |
|---|---|---|---|
| **Request DTO** (o "Médico Request") | Controller | Representa **lo que llega del cliente** (body del POST/PUT) | Validado con Zod. Sin lógica. Sin `_id`. |
| **Dominio** (`Medico`) | Domain / Service | Representa **el médico como concepto del negocio** | Tiene métodos de comportamiento. Sin `_id`. Sin detalles de persistencia. |
| **Schema** (o DAO, "Médico Schema") | Repository | Representa **el documento como vive en Mongo** | Tiene `_id`, `__v`. Definido con `mongoose.Schema`. |

### 5.3 El flujo completo de una request

Para que se vea cómo conviven los tres modelos, seguí mentalmente esta request:

```
POST /api/medicos
Body: { "nombre": "Ana", "matricula": "M-789", "edad": 40 }

  1. CONTROLLER
     ─ Recibe el body como un objeto JS (Request DTO)
     ─ Lo valida con Zod → ¿forma correcta? ¿tipos correctos?
     ─ Si no valida: 400 Bad Request
     ─ Si valida: pasa el DTO al Service
        ↓
  2. SERVICE
     ─ Recibe el DTO
     ─ Lo CONVIERTE a entidad de dominio (Medico)
     ─ Aplica reglas de negocio: ¿matrícula válida? ¿edad razonable?
     ─ Si una regla falla: tira un error de dominio (que el controller mapea a 422)
     ─ Si todo ok: llama al Repository.guardar(medicoDeDominio)
        ↓
  3. REPOSITORY
     ─ Recibe la entidad de dominio
     ─ La CONVIERTE a documento (Schema/DAO de Mongo)
     ─ Llama a Mongoose: MedicoModel.create({...})
     ─ Mongoose guarda el documento (le pone _id, __v)
     ─ Repository devuelve la entidad de dominio (NO el documento)
        ↑
  4. SERVICE
     ─ Recibe la entidad de dominio guardada
     ─ La devuelve al Controller
        ↑
  5. CONTROLLER
     ─ Recibe la entidad de dominio
     ─ La SERIALIZA como respuesta (otro DTO de salida, sin _id ni __v)
     ─ res.status(201).json(respuesta)
```

**Tres conversiones explícitas:**

- **DTO entrada → Dominio** (en Service)
- **Dominio → Schema/DAO** (en Repository, al guardar)
- **Schema/DAO → Dominio** (en Repository, al leer)
- **Dominio → DTO salida** (en Controller, al responder)

Cuatro conversiones en total. Parece mucho, pero cada una está justificada y cada una vive en una capa específica.

### 5.4 ¿Por qué no usar uno solo y ahorrar conversiones?

Porque mezclar capas se paga caro. Algunos ejemplos concretos de qué pasa si no separás:

- **Si usás el Schema de Mongoose como objeto de dominio:** tu lógica de negocio queda atada a Mongo. Si mañana migrás a Postgres, refactorizás todo. Además, los métodos de dominio (`turno.cancelar()`, `turno.estaPasado()`) terminan compartiendo objeto con métodos de Mongoose (`save()`, `populate()`) — confuso.

- **Si usás el DTO de entrada como objeto de dominio:** la lógica de negocio termina dependiendo de la forma exacta del request. Cualquier cambio en la API rompe el dominio. Y el dominio no tiene métodos — solo es un puñado de campos planos.

- **Si exponés el Schema/DAO como respuesta:** estás mostrando `_id`, `__v`, y todo el detalle de persistencia. Acoplás la API a Mongo, abrís puerta a inyección NoSQL.

La separación cuesta líneas de código. Pero **te da independencia entre capas**. Cada capa puede cambiar sin que las otras se enteren. Eso es exactamente lo que vendía la arquitectura por capas que viste en la semana 4 práctica — esta separación es la encarnación de ese principio aplicada a persistencia.

### 5.5 Un ejemplo en código de las tres representaciones

Para que se vea concreto:

```javascript
// ────────────────────────────────────────────────────────
// 1. Request DTO (Controller — validado con Zod)
// ────────────────────────────────────────────────────────
const MedicoRequestSchema = z.object({
  nombre:    z.string().min(3),
  matricula: z.string().regex(/^M-\d+$/),
  edad:      z.number().int().positive(),
  especialidades: z.array(z.string()).optional()
});
// type MedicoRequest = z.infer<typeof MedicoRequestSchema>;
// → { nombre, matricula, edad, especialidades? }


// ────────────────────────────────────────────────────────
// 2. Dominio (Domain — clase con comportamiento)
// ────────────────────────────────────────────────────────
class Medico {
  constructor({ id, nombre, matricula, edad, especialidades }) {
    this.id = id;                    // string, no ObjectId
    this.nombre = nombre;
    this.matricula = matricula;
    this.edad = edad;
    this.especialidades = especialidades || [];
  }

  // Métodos de comportamiento — la "inteligencia" del médico:
  atiendeEspecialidad(esp) {
    return this.especialidades.includes(esp);
  }

  agregarEspecialidad(esp) {
    if (!this.atiendeEspecialidad(esp)) {
      this.especialidades.push(esp);
    }
  }
}


// ────────────────────────────────────────────────────────
// 3. Schema / DAO (Repository — definido con Mongoose)
// ────────────────────────────────────────────────────────
const medicoSchema = new mongoose.Schema({
  nombre:    { type: String, required: true },
  matricula: { type: String, required: true, unique: true },
  edad:      { type: Number, required: true },
  especialidades: [String]
});

const MedicoModel = mongoose.model('Medico', medicoSchema);
// El documento que vive en Mongo tendrá ADEMÁS:
//   _id: ObjectId(...)
//   __v: 0
```

Las tres representaciones son **claramente distintas**, y cada una vive en su capa. Cuando el grupo de la cátedra corrija tu TP, va a buscar exactamente esto.

> 🔴 **Para la corrección del TP2:** asegurate de que en el repo se vean los tres tipos de archivos. Algo como:
>
> - `controllers/MedicoController.js` (con el Zod schema del request)
> - `domain/Medico.js` (la clase de dominio)
> - `repositories/MedicoRepository.js` que importa `infrastructure/schemas/MedicoSchema.js` (el Mongoose schema)
>
> Si tres archivos distintos no aparecen, es probable que estés mezclando capas.

---

## 6. Refactor del TP1 al TP2: qué cambia capa por capa 🟡

Tu TP1 ya tiene estructura por capas (al menos así estaba pensado). El TP2 es **incremental**: NO empezás de cero, mejorás lo que tenías. Cada capa cambia o no cambia algo distinto:

### Controller — casi no cambia

El controller seguía siendo el mismo de antes: recibe la request, valida con Zod, llama al service, mapea errores a HTTP, devuelve la response.

**Único cambio:** la respuesta de salida ahora puede tener un `id` propio (string) que viene del `_id` de Mongo mapeado en el repository. Antes tu repository en memoria asignaba IDs como un contador entero; ahora vienen de Mongo.

### Service — casi no cambia

El service también queda igual: orquesta la lógica de negocio, llama al repository, aplica reglas, lanza errores de dominio.

**Posible cambio:** algunas operaciones que antes eran sincrónicas (porque la base era memoria) ahora son async. Si tu repository devolvía objetos directos antes y ahora devuelve Promises, tu service tiene que `await` esas llamadas. Si lo tenías bien hecho, ya estaba con `async/await`. Si no, es buen momento para corregir.

### Repository — cambia BASTANTE

Esta es la capa que más se modifica. Antes:

```javascript
// REPOSITORY del TP1 — en memoria
class MedicoRepository {
  constructor() {
    this.medicos = [];
    this.nextId = 1;
  }

  guardar(medico) {
    medico.id = this.nextId++;
    this.medicos.push(medico);
    return medico;
  }

  obtenerTodos() {
    return this.medicos;
  }

  obtenerPorId(id) {
    return this.medicos.find(m => m.id === id);
  }

  eliminar(id) {
    const idx = this.medicos.findIndex(m => m.id === id);
    if (idx === -1) throw new NotFoundError('Médico no encontrado');
    this.medicos.splice(idx, 1);
  }
}
```

Ahora:

```javascript
// REPOSITORY del TP2 — con Mongoose
const MedicoModel = require('../infrastructure/schemas/MedicoSchema');
const Medico = require('../domain/Medico');

class MedicoRepository {
  // ─ Conversión doc → dominio
  toDomain(doc) {
    if (!doc) return null;
    return new Medico({
      id: doc._id.toString(),
      nombre: doc.nombre,
      matricula: doc.matricula,
      edad: doc.edad,
      especialidades: doc.especialidades
    });
  }

  // ─ Conversión dominio → doc (para create/update)
  toDocument(medico) {
    return {
      nombre: medico.nombre,
      matricula: medico.matricula,
      edad: medico.edad,
      especialidades: medico.especialidades
    };
  }

  async guardar(medico) {
    const doc = await MedicoModel.create(this.toDocument(medico));
    return this.toDomain(doc);
  }

  async obtenerTodos() {
    const docs = await MedicoModel.find({});
    return docs.map(d => this.toDomain(d));
  }

  async obtenerPorId(id) {
    const doc = await MedicoModel.findById(id);
    if (!doc) throw new NotFoundError('Médico no encontrado');
    return this.toDomain(doc);
  }

  async eliminar(id) {
    const result = await MedicoModel.findByIdAndDelete(id);
    if (!result) throw new NotFoundError('Médico no encontrado');
  }
}
```

**Cosas que cambian:**

- Todos los métodos son `async` ahora.
- Aparecen los métodos `toDomain` y `toDocument` — son los conversores entre las dos representaciones.
- `MedicoModel` viene de `mongoose.model(...)` — es la "tabla" en lenguaje Mongo.
- Los IDs ahora son strings (vienen del `_id.toString()`), no ints incrementales.
- Las validaciones de "no existe" ahora se hacen con `if (!doc)` o con el resultado de `findByIdAndDelete`.

### Domain — no cambia para nada

La capa Domain queda **exactamente igual**. Tus clases (`Medico`, `Turno`, `Plan`, `CoberturaEspecialidad`...) siguen siendo lo que eran, con sus métodos de comportamiento. No saben que existe Mongo. No las tenés que tocar.

> **Esto es el premio por haber separado en capas en su momento:** el cambio de "memoria" a "Mongo" es un cambio en una sola capa (el Repository). Las otras tres capas se mantienen estables. Si hubieras tenido todo el código mezclado en `index.js`, este refactor sería un infierno.

---

## 7. Feedback de la entrega 1 que aplica directo a la entrega 2 🟡

Agus y Lucas dieron una ronda de comentarios generales sobre lo que vieron en los PRs de TP1. Esto **te aplica si tu PR del grupo tuvo comentarios** — chequealo con tus compañeros antes de arrancar TP2. Algunos puntos no son específicos a un grupo, son patrones que vieron en muchos:

### 7.1 Rutas que no son REST

Algunas rutas no respetaban convenciones REST. Ejemplos comunes:

- ❌ `POST /crearMedico` → ✅ `POST /medicos`
- ❌ `GET /obtenerTurnosDelMedico/123` → ✅ `GET /medicos/123/turnos`
- ❌ `POST /turnos/cancelar/456` → ✅ `PATCH /turnos/456` (con body indicando el cambio de estado) o `DELETE /turnos/456`

**Acción:** revisá tu archivo de rutas y validá que cada endpoint use el verbo HTTP correcto (GET, POST, PUT, PATCH, DELETE) y el path como sustantivo en plural. Para la entrega 2 esto se valida más fuerte.

### 7.2 Middleware de errores copiado pero no usado

Varios grupos copiaron el patrón del middleware de errores que se vio en clase, pero **siguieron poniendo `try/catch` en cada controller**, lo cual hace al middleware redundante.

```javascript
// ❌ Patrón mal aplicado
async function crearMedico(req, res, next) {
  try {
    const medico = await medicoService.crear(req.body);
    res.status(201).json(medico);
  } catch (err) {
    // Mapeo manual del error a HTTP — el middleware no se usa
    if (err instanceof NotFoundError) return res.status(404).json({...});
    if (err instanceof ValidationError) return res.status(422).json({...});
    return res.status(500).json({ error: 'Error interno' });
  }
}

// ✅ Patrón correcto: dejar que el error suba, el middleware lo atrapa
async function crearMedico(req, res, next) {
  try {
    const medico = await medicoService.crear(req.body);
    res.status(201).json(medico);
  } catch (err) {
    next(err);  // ← deja que el middleware se encargue
  }
}
```

El middleware único de errores debería estar en `app.use((err, req, res, next) => {...})` después de todas las rutas, mapeando los errores de dominio a HTTP en un solo lugar.

### 7.3 Promises mal usadas

Cinco grupos no usaron promises bien — Lucas lo nombró explícito. Esto es: o no había `await` donde correspondía, o se mezclaban patterns (callbacks con `.then()` con `async/await` en el mismo flujo). Para el TP2 es **más crítico todavía** porque todas las operaciones de Mongoose son async.

**Chequeo rápido:** abrí cada función del Service y del Repository del TP2. Si una función llama a algo de Mongoose, **tiene que ser `async`** y tiene que `await` la llamada. No hay excepciones.

### 7.4 Validaciones en Controller que deberían estar en Zod o en Service

Algunos validaban con `if`s en el controller cuando podrían haber armado un schema de Zod más completo, o cuando la validación era de regla de negocio (que va en Service).

**Regla rápida del criterio de capas:**

- **Forma del request** (que existan los campos, que sean del tipo correcto): **Zod en el Controller**.
- **Reglas de negocio** (que el médico tenga al menos 18 años, que la matrícula no esté duplicada): **Service** con errores de dominio.
- **Estructura del documento que va a la base**: **Schema de Mongoose en el Repository**.

### 7.5 Errores HTTP-mapeados tirados desde el Dominio

Algunos grupos tenían cosas como:

```javascript
// ❌ MAL: el dominio no debería conocer códigos HTTP
class Medico {
  cancelarTurno(turno) {
    if (turno.fechaPasada()) {
      throw new BadRequestError('No se puede cancelar...');  // ← mal lugar
    }
  }
}
```

El Dominio tira **errores de dominio**, semánticos, que el Controller (con ayuda del middleware) mapea a códigos HTTP. La capa Domain no debería saber que existe HTTP.

```javascript
// ✅ BIEN
class Medico {
  cancelarTurno(turno) {
    if (turno.fechaPasada()) {
      throw new TurnoYaRealizadoError('No se puede cancelar un turno pasado');
    }
  }
}

// Y en el middleware o en el controller:
if (err instanceof TurnoYaRealizadoError) {
  return res.status(409).json({ error: err.message });
}
```

### 7.6 Otros aspectos operativos

Lucas también remarcó algunos temas operativos que te conviene tener en mente para la entrega 2:

- **Mensajes de commit descriptivos.** Comits con nombres como `"test"`, `"fix"`, `"asd"` no van. El nombre del commit tiene que decir qué cambió, para que un rollback futuro sea fácil.
- **No commitear después del horario de cierre.** Si el plazo era a las 17:00, no podés tener un commit del último cambio a las 17:44. Eso se nota.
- **Preparar la presentación 20 minutos antes.** Tener IDE listo, Postman abierto, ejemplos cargados. No se puede gastar 10 minutos en setup el día de la defensa.
- **El que no presentó la entrega 1, presenta la entrega 2.** Si en tu grupo presentaste vos la primera, esta vez le toca a otro. El que comparte pantalla es el que habla.

---

## 8. Cierre y conexión con próximas clases

Esta clase fue **panorama + introducción al ODM**. Lo que hicimos hasta acá:

1. Cerramos el tema de mocking que había quedado pendiente de la clase 4 — ahora podés escribir tests del Service mockeando el Repository.
2. Levantamos el mapa completo de persistencia: tipos de almacenamiento, dos familias de bases, las cuatro NoSQL principales, ACID, CAP.
3. Entramos a Mongoose: Schema, Model, operaciones básicas, comportamientos a tener en cuenta, `_id` y por qué no exponerlo.
4. Marcamos los tres modelos del TP2 (Request DTO / Dominio / Schema) como condición de corrección explícita.
5. Sacamos el feedback del TP1 que aplica directo a TP2.

**Lo que NO se cubrió bien y vas a ver pronto:**

- **Embedded vs Referenced** — la decisión de cómo modelar relaciones entre documentos en Mongo. ¿El médico tiene una lista de turnos embebidos en su documento, o los turnos son una colección aparte que referencia al médico? Esa decisión es central para el TP2 y se ve **en la clase práctica del sábado 09/05**, donde Mongoose se trabaja en código con todos los detalles.
- **Queries más complejas** — `$lookup` (el equivalente a JOIN en Mongo), agregaciones, sort, skip/limit para paginación. También en la clase del sábado.
- **Tests con mocks de Mongoose** — cómo testear un Service mockeando el Repository sin necesidad de levantar Mongo. Aplicación directa de lo de la sección 1 de este apunte.

**La próxima clase de teoría (12/05, semana 7)** salta de tema: arranca Frontend con HTML, CSS y JavaScript del lado cliente. La transición es brusca, pero el backend ya estará en buen estado para que vos arranques la entrega 2 en paralelo a las clases de frontend.

---

## Checkpoint Parte 2

Si pudiste responder estas sin volver al apunte, la Parte 2 está digerida.

1. ¿Qué es un ODM y en qué se diferencia de un ORM? Dá un ejemplo de cada uno.
2. ¿Por qué el mapeo es más simple en un ODM que en un ORM? Mencioná dos razones concretas.
3. ¿Qué hace `mongoose.connect()` y por qué se llama una sola vez al arrancar la app?
4. Definí qué es un Schema en Mongoose y para qué sirve. Mencioná las cuatro opciones de campo más usadas.
5. ¿Cuál es la diferencia entre `find()` y `findOne()` en términos de qué devuelven?
6. Si llega un POST con un atributo que no está en el schema, ¿qué hace Mongoose? ¿Por qué es una buena defensa por defecto?
7. Si llega un POST sin un campo `required`, ¿qué pasa? ¿Cómo deberías manejarlo en el controller?
8. ¿Qué es el campo `__v` en un documento de Mongoose? ¿Lo expondrías en la API?
9. ¿Qué es el `_id` y de qué tipo es? ¿Por qué la cátedra dice que NUNCA hay que exponerlo en la API? Mencioná las tres razones.
10. Nombrá los tres modelos que el TP2 va a evaluar y dónde vive cada uno (qué capa).
11. ¿Cuántas conversiones explícitas tiene un flujo POST completo si separás bien los tres modelos? Listalas.
12. Si el TP1 estaba bien hecho con capas, ¿qué capa cambia mucho al pasar al TP2? ¿Cuáles cambian poco o nada?
13. ¿Por qué el Domain "no sabe que existe Mongo"? ¿Qué problema previene esa ignorancia deliberada?
14. Mencioná tres errores de patrón que Agus y Lucas señalaron en los TPs1 que vas a tener que evitar en TP2.
15. ¿Dónde van las validaciones de forma del request? ¿Y las de regla de negocio? ¿Y las de estructura del documento?
16. ¿Qué tema se anticipó para la clase práctica del sábado siguiente que esta clase no cubrió?

> Las respuestas en formato examen están en `complemento-semana6-teoria.md`.

---

**FIN DEL APUNTE MAESTRO — SEMANA 6 TEORÍA, PARTE 2**
**FIN DEL APUNTE MAESTRO DE LA SEMANA 6 TEORÍA**
