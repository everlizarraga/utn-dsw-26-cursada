# Apunte Resumen — Semana 6 Teoría

## Mocking, persistencia, ODM y los tres modelos del TP

> **Semana 6 — Martes 05/05 — Profes: Agus (lidera), Lucas (soporte)**
>
> Resumen consolidado de las dos partes del apunte maestro. Para repaso espaciado. Lectura densa, ya con el tema entendido. Cobertura completa, compresión dentro de cada punto.

---

## Info operativa 📘

- Material oficial: PDF "Resumen Clase 6" (header dice "Miércoles Noche" por error de template, contenido es el correcto).
- **Parcial:** sábado 27/06/2026, 9 a 11hs, Medrano (Agus gestiona Lugano, sin confirmar).
- Recuperatorio: 11/07/2026.
- **Entrega 2 TP:** 26/05 (semana 9). Liberación del enunciado durante la semana 6.
- Próxima clase teórica (12/05): arranca Frontend. Cambio total de chip.
- Práctica del sábado (09/05): profundización de Mongoose con embedded vs reference.

---

## 0. De dónde venís y dónde te deja la clase 📘

TP1 con repositorios en memoria → TP2 con MongoDB real. La clase es el primer eslabón del salto. Cubre:

1. **Cierre de mocking** (estaba diferido de clase 4): necesario para testear repositorios sin tocar la base.
2. **Mapa de persistencia**: opciones disponibles, garantías de cada una, por qué la cátedra eligió Mongo.
3. **Mongoose básico + tres modelos del TP2** (Request DTO / Dominio / Schema): condición de corrección explícita.

Conecta con Bases de Datos (futura), Operativos (transacciones, sistemas distribuidos), Paradigmas (modelado de objetos).

---

## 1. Mocking: cierre del tema de testing 🔴 🎯

### 1.1 Por qué este tema acá 📘

Clase 4 de teoría vio pirámide, TDD y BDD. Mocking quedó diferido a esta clase porque su uso natural aparece cuando hay base de datos real: testear repositorios sin tocar Mongo, testear services mockeando el repository.

### 1.2 Qué es un mock (sentido amplio) 📘

Simulación de un componente dentro de un test. Vos definís el comportamiento. Te permite acotar lo que estás testeando, ignorando lo de alrededor.

**Dónde aparece:**
- Test unitario: mockeás dependencias (otra clase, repository) para concentrarte en una unidad.
- Test de integración: mockeás base de datos, API externas, servicios.

**Qué te permite simular:** valores inesperados, errores, caídas, respuestas 404.

> **Nota terminológica:** "mockear" se usa en sentido amplio (simular en general) y en sentido estricto (uno de los tres tipos). En este apunte: "mockear" = acción genérica, "Stub/Mock/Spy" = los tres tipos específicos.

### 1.3 Los tres tipos 🔴 🎯

#### Stub — el objeto tonto

Objeto falso que devuelve datos prefijados. No ejecuta lógica real. *"Cuando te llame, devolveme esto."* No te importa cómo lo llamaron.

```javascript
const productRepositoryStub = {
  getProductById: (id) => ({
    id,
    name: "Camiseta",
    price: 500
  })
};

const product = productRepositoryStub.getProductById(productId);
// Siempre devuelve { id, name: "Camiseta", price: 500 }
```

> 🔴 **Agus dijo:** *"En el TP casi siempre van a usar Stub."* Es la herramienta principal del TP2 cuando testean Service → Repository.

#### Mock — el objeto que registra interacciones

Reemplaza la dependencia y registra cómo la usaron: cuántas veces, con qué parámetros, en qué orden. *"Validame que me hayan llamado X veces con Y parámetros."*

```javascript
function registerUser(email, mailer) {
  // ... resto del código ...
  mailer.sendWelcome(email);
}

// En el test:
const mockMailer = {
  sendWelcome: jest.fn()  // función mock vacía que registra todo
};

registerUser("email@test.com", mockMailer);

expect(mockMailer.sendWelcome).toHaveBeenCalledTimes(1);
expect(mockMailer.sendWelcome).toHaveBeenCalledWith("email@test.com");
```

**Stub vs Mock — foco distinto:**
- Stub: te importa la salida (qué te devuelven).
- Mock: te importa la entrada (cómo te llamaron).

#### Spy — el espía

No reemplaza al objeto: lo envuelve. El método real se ejecuta, pero el spy registra qué pasó. Es un wrapper/decorator.

```javascript
const calculadora = { sum: (a, b) => a + b };

const spy = jest.spyOn(calculadora, "sum");
const result = calculadora.sum(2, 3);
// result === 5 (la suma real corre)
expect(spy).toHaveBeenCalledWith(2, 3);
```

**Uso:** Agus dijo que casi no se usa. Para el TP probablemente no lo necesites.

### 1.4 Tabla de bolsillo 🎯

| Tipo | ¿Reemplaza al original? | ¿Te importa qué devuelve? | ¿Te importa cómo lo llamaron? | Uso típico TP |
|---|---|---|---|---|
| **Stub** | Sí | Sí (vos definís) | No | **Principal:** Service → Repository |
| **Mock** | Sí | Sí | Sí | Verificar `save`, `delete`, `notify` |
| **Spy** | No, envuelve | El real define | Sí | Casos puntuales |

### 1.5 Conexión con el TP2 🎯

- Tests de Dominio: lógica pura, **sin mocks**. Le pasás datos, validás resultado.
- Tests de Service: **mockeás Repository** (Stub mayormente, Mock para verificar interacciones).

> **Para el parcial, si te preguntan:**
>
> **P: Diferenciá Stub, Mock y Spy. Dá un ejemplo de cuándo usarías cada uno.**
>
> **R:** Los tres son técnicas para simular dependencias en un test. **Stub** es un objeto falso que devuelve datos prefijados sin ejecutar lógica real; lo uso cuando necesito que una dependencia "esté ahí" y responda algo razonable, pero solo me importa el resultado de la función bajo test (ej: stubear el Repository para devolver una lista de productos al testear un Service). **Mock** también reemplaza la dependencia, pero la diferencia es que registra cómo lo usaron — lo uso cuando me importa verificar la interacción (ej: validar que el Service llame al método `save` del Repository con el objeto correcto, una sola vez). **Spy** no reemplaza al objeto: lo envuelve y deja que el método real se ejecute, pero registra qué pasó; lo uso cuando quiero que el comportamiento real ocurra pero también auditar las llamadas (ej: validar que un método auxiliar se invocó la cantidad correcta de veces dentro de una operación más grande).

---

## 2. Persistencia: definición y tipos 🟡 📘

### 2.1 Definición

Capacidad de almacenar datos de forma duradera, de modo que no se pierdan cuando la app se cierra o el sistema se apaga. **Dimensión extra que subrayó Agus:** además de durar, accesibilidad controlada (no libre — seguridad y permisos son parte del problema).

### 2.2 Tipos de almacenamiento 📘

| Tipo | Ejemplos | Cuándo |
|---|---|---|
| Archivos | `.txt`, `.csv`, `.json`, `.xls`, Google Sheets | Datos chicos, configuración, exportes |
| Bases de datos | MySQL, MariaDB, PostgreSQL, MongoDB, Cassandra, Redis, Neo4J | Datos estructurados consultados |
| Almacenamiento en nube | S3, Google Cloud Storage | Archivos grandes, snapshots, binarios |
| Sistemas distribuidos | Hadoop, Iceberg | Volúmenes enormes (Big Data) |

**Datalake:** capa de presentación para BI, no es almacenamiento puro. Mención cultural, no entra en parcial ni TP.

### 2.3 Por qué Mongo en el TP 📘

Decisión pedagógica de la cátedra. Agus: *"En el TP les decimos usen Mongo para que no usen SQL, porque SQL ya la van a ver en Bases de Datos y en Diseño."* No es porque Mongo sea técnicamente superior para este problema.

---

## 3. SQL vs NoSQL: las dos familias 🔴 📘

### 3.1 SQL — relacionales

Datos en **tablas** con esquema fijo, relaciones con FKs, lenguaje SQL declarativo y bastante estandarizado.

**Motores:** MySQL/MariaDB (foca), PostgreSQL (elefantito), Supabase (Postgres gestionado), SQL Server, DB2, Oracle.

**Cuándo:** estructura estable, consistencia fuerte (plata, usuarios, transacciones), reportes con muchos JOINs.

**Trade-off:** `ALTER TABLE` en tablas grandes es caro y arriesgado.

### 3.2 NoSQL — cuatro familias (y una más de yapa)

#### Documental 🔴 🎯

Documentos JSON/BSON. **MongoDB** (TP), CouchDB.

**Schemaless:** no necesitás definir esquema rígido. Documentos en la misma colección pueden tener campos distintos. (⚠️ schemaless es de las documentales, **no de toda NoSQL**.)

#### Columnar 🟢

Cassandra. Datos en estructura tipo tabla pero distribuida por columna. Particularidad: el modelo se diseña a partir de las **consultas**, no de los datos. Lenguaje CQL.

#### Clave-valor 🟢

Redis. Clave única → valor. Optimizado para acceso ultrarrápido, vive en RAM. **Uso típico:** caché que complementa a la base principal.

#### Grafos 🟢

Neo4J. Nodos (entidades) + aristas (relaciones, con pesos). **Casos:** antifraude, recomendaciones (LinkedIn, Instagram), mapas y rutas.

#### Vectoriales 🟢

Pinecone, Weaviate, Chroma. Almacenan embeddings. Base de RAG e IA generativa. Mención cultural.

### 3.3 Cuándo SQL y cuándo NoSQL 📘

| Necesidad | Elegí |
|---|---|
| Datos críticos, consistencia fuerte (plata, usuarios) | SQL |
| Estructura estable, relaciones definidas | SQL |
| Reportes con muchos JOINs | SQL |
| Esquema flexible que evoluciona | Documental (Mongo) |
| Alta concurrencia, feed/posts | Documental |
| Caché de búsquedas, datos volátiles | Clave-valor (Redis) |
| Análisis de relaciones (fraude, recomendaciones) | Grafos (Neo4J) |
| Volumen masivo, distribución extrema | Columnar (Cassandra) |

**Frase de Agus:** *"Capaz tu solución usa MariaDB para usuarios, Cassandra para feed y Redis para caché — y está bien. La base no es UNA decisión, son varias decisiones de diseño."*

---

## 4. ACID 🔴 📘

Cuatro propiedades de transacciones en bases relacionales.

**Transacción:** unidad de trabajo que la base trata como un todo (puede ser una o varias operaciones).

| Letra | Significa | Qué garantiza |
|---|---|---|
| **A** | Atomicity | Todo o nada. Si algo falla, rollback. |
| **C** | Consistency | La base queda en estado válido antes/después. Respeta reglas de integridad. |
| **I** | Isolation | Transacciones concurrentes no se pisan. |
| **D** | Durability | Una vez commit, los cambios sobreviven a fallos del sistema. |

> ⚠️ **Cuidado:** "Consistency" en ACID ≠ "Consistency" en CAP. Misma palabra, dos cosas distintas.

**Aclaración cátedra + matiz honesto:** Agus dijo *"algunas NoSQL como Mongo también cumplen ACID"*. En la práctica: Mongo da ACID a nivel un documento desde siempre, multi-documento desde versión 4.0/4.2 con limitaciones. Para el parcial → respondé lo que dice cátedra, mencionar "a nivel documento" suma sin contradecir.

> **Para el parcial, si te preguntan:**
>
> **P: ¿Qué es ACID? Explicá cada letra.**
>
> **R:** ACID es el acrónimo que describe las cuatro propiedades que garantizan la confiabilidad de las transacciones en bases de datos relacionales. **Atomicity** (atomicidad): una transacción es una unidad indivisible — o se ejecuta completa o no se ejecuta nada; si algún paso falla, se hace rollback de todo. **Consistency** (consistencia): la base de datos siempre queda en un estado válido antes y después de cada transacción, respetando las reglas de integridad definidas. **Isolation** (aislamiento): las transacciones concurrentes no interfieren entre sí; cada una se ejecuta como si fuera la única. **Durability** (durabilidad): una vez que una transacción se confirma con commit, los cambios son permanentes y sobreviven a fallos del sistema.

---

## 5. Teorema CAP 🔴 📘

Trade-off de bases distribuidas: hay tres propiedades deseables, **solo podés tener dos al mismo tiempo**.

### Las tres propiedades

- **🔄 Consistency:** todos los nodos ven los mismos datos al mismo tiempo. Lectura devuelve la escritura más reciente o error.
- **⚡ Availability:** el sistema sigue respondiendo aunque algunos nodos fallen.
- **🔌 Partition Tolerance:** el sistema sigue operando aunque haya fallos de red.

### Las tres combinaciones

| Combinación | Sacrifica | Ejemplos |
|---|---|---|
| **CA** | Partition Tolerance | Bases relacionales tradicionales en un solo nodo |
| **CP** | Availability | **MongoDB**, HBase, Redis |
| **AP** | Consistency | Cassandra, DynamoDB |

### Gráfico de pizarra

```
                  Consistency
                      C
                     /  \
                    /    \
                  CA      CP
                  /        \
                 /          \
                A____________P
            Availability  Partition
                          Tolerance
                  AP
```

### Aclaración honesta

Agus dijo: *"las versiones nuevas de Mongo cumplen las tres y te dan todo lo de ACID, está bastante refinado."*

**Matiz:** las bases modernas tienen mecanismos finos (sharding, replica sets, write concerns) que se aproximan a las tres en condiciones normales. Pero ante partición real de red, la elección sigue existiendo. El teorema sigue válido; lo que cambió son los mecanismos.

**Para el parcial → Mongo es CP.** Es lo que dice el material oficial. El matiz suma si tenés tiempo, pero no contradice la respuesta principal.

### Conceptos al pasar

- **Replication factor:** en cuántos nodos se replica un dato.
- **Quórum:** mitad + uno de los nodos. Se usa para decidir qué leer/escribir.

> **Para el parcial, si te preguntan:**
>
> **P: ¿Qué es el teorema CAP? Explicá las tres propiedades y la regla de "dos de tres".**
>
> **R:** El teorema CAP describe las tres propiedades deseables de un sistema de base de datos distribuido y establece que solo se pueden garantizar dos simultáneamente. Las propiedades son: **Consistency** (todos los nodos ven los mismos datos al mismo tiempo, cualquier lectura devuelve la escritura más reciente o un error), **Availability** (el sistema sigue respondiendo aunque algunos nodos fallen), y **Partition Tolerance** (el sistema sigue operando aunque haya fallos de red entre nodos). En presencia de una partición de red, hay que elegir entre consistencia y disponibilidad: las bases CP como MongoDB sacrifican disponibilidad por consistencia (prefieren no responder antes que dar datos potencialmente incorrectos); las bases AP como Cassandra sacrifican consistencia por disponibilidad (siempre responden, aunque los datos puedan estar desactualizados); las bases CA como las relacionales tradicionales no toleran particiones de red, por eso suelen vivir en un solo nodo lógico.

---

## 6. Preguntas que quedaron flotando 📘

**¿Por qué Mongo es CP y no CA?** Porque Mongo está pensada para ser distribuida (replica sets, sharding). Asume varios nodos, entonces tiene que decidir qué hacer cuando no se hablan. Elige consistencia: prefiere bloquearse antes que devolver datos viejos.

**¿Por qué los únicos puntos del triángulo son CA, CP y AP?** Porque en un sistema distribuido realista, Partition Tolerance no es opcional (si la red falla y no tolerás, te caés). En la práctica la decisión real es entre CP y AP. Las bases CA suponen red perfecta, lo cual solo se cumple en un único servidor.

---

## 7. ORM vs ODM 🟡 📘

### ORM

Mapea **objetos** a **tablas relacionales**. Hibernate (Java), Entity Framework (.NET), SQLAlchemy (Python).

```java
@Entity                          // "Esto es una tabla"
public class Usuario {
    @Id                          // "Esto es la primary key"
    private Long id;
    private String nombre;
    private Integer edad;
    @Transient                   // "No persistas este campo"
    private String dataDeSesion;
}
```

**Complejidad:** herencia, composición, relaciones (`@OneToMany`, `@ManyToOne`). El ORM resuelve un problema real pero no es transparente.

### ODM

Mapea **objetos** a **documentos** JSON/BSON. **Mongoose** (Node + Mongo). No hay impedancia: objeto JS ≈ documento JSON.

```javascript
// Objeto en código:
const usuario = {
  nombre: "Ana",
  edad: 30,
  direcciones: [
    { calle: "Av. Corrientes 1234", ciudad: "Buenos Aires" },
    { calle: "Av. Cabildo 5678",   ciudad: "Buenos Aires" }
  ]
};

// Documento en Mongo (BSON), casi idéntico:
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

### Tabla comparativa

| Eje | ORM | ODM (Mongoose) |
|---|---|---|
| Base | Relacional (SQL) | Documental (Mongo) |
| Estructura | Tablas con esquema rígido | Documentos JSON, schemaless |
| Mapeo | Complejo (herencia, joins, FKs) | Casi directo |
| Cambios de modelo | Costosos (ALTER TABLE) | Baratos |
| Aprendizaje | Mucho | Poco |

**Conclusión de Agus:** *"el ODM es fácil de pensar porque ya pensás en tus objetos de dominio."*

**Sobre embedded vs reference:** la simplicidad de "todo embebido" tiene límite. La decisión cómo modelar relaciones se ve en la práctica del sábado.

---

## 8. Mongoose: el ODM de Node 🔴 🎯

### 8.1 Conexión

```javascript
// db/connection.js
const mongoose = require('mongoose');

// URI: mongodb://[host]:[puerto]/[nombreDB]
// Local típico: mongodb://localhost:27017/sweetmedical
// Producción: variable de entorno
const URI = process.env.MONGO_URI || 'mongodb://localhost:27017/sweetmedical';

mongoose.connect(URI)
  .then(() => console.log('Conectado a MongoDB'))
  .catch((err) => console.error('Error conectando a MongoDB:', err));
```

**Detalles:**
- `mongoose.connect()` devuelve Promise. Manejala con `.then/.catch` o `await`.
- El nombre de la base lo elegís vos; Mongo la crea al primer insert.
- Una sola conexión por app (Mongoose mantiene pool).

### 8.2 Schema 🎯

Descripción de cómo son los documentos de una colección. Mongo es schemaless, pero Mongoose te recomienda definirlo para validar y estructurar.

```javascript
// domain/schemas/MedicoSchema.js
const mongoose = require('mongoose');

const medicoSchema = new mongoose.Schema({
  // Cada campo se define con un tipo y, opcionalmente, opciones.

  nombre: {
    type: String,        // Tipos básicos de JS:
                         //   String, Number, Boolean, Date, Array, ObjectId
    required: true       // Sin nombre, REVIENTA al guardar
  },

  matricula: {
    type: String,
    required: true,
    unique: true         // No puede haber dos médicos con la misma matrícula
                         // Mongoose crea automáticamente un índice único
  },

  email: String,         // Forma corta cuando solo necesitás el tipo
                         // (sin opciones extra). Equivale a { type: String }

  edad: Number,

  fechaIngreso: {
    type: Date,
    default: Date.now    // Si no se manda, usa fecha actual
  },

  especialidades: [String]  // Array de strings
});

// Model: clase usable en código real.
// 'Medico' → colección 'medicos' (plural minúscula automático)
const Medico = mongoose.model('Medico', medicoSchema);

module.exports = Medico;
```

**Cuatro opciones más usadas:** `type`, `required`, `unique`, `default`.

### 8.3 Operaciones básicas 🎯

```javascript
// CREAR
const nuevoMedico = await Medico.create({
  nombre: 'Juan',
  matricula: 'M-12345',
  email: 'juan@hospital.com',
  edad: 45,
  especialidades: ['Cardiología', 'Clínica Médica']
});


// BUSCAR TODOS (devuelve ARRAY, puede estar vacío)
const medicos = await Medico.find({});                    // todos
const cardiologos = await Medico.find({                   // filtrados
  especialidades: 'Cardiología'
});


// BUSCAR UNO (devuelve OBJETO o null)
const medico = await Medico.findOne({ matricula: 'M-12345' });
if (medico) {
  console.log(medico.nombre);
}


// BUSCAR POR ID
const medico = await Medico.findById(idDelMedico);


// ACTUALIZAR
await Medico.findByIdAndUpdate(idDelMedico, { edad: 46 });


// ELIMINAR
await Medico.findByIdAndDelete(idDelMedico);
```

> 🔴 **`find` vs `findOne`:** `find` devuelve ARRAY (vacío si no hay). `findOne` devuelve OBJETO (o `null`). Confundirlos = errores "cannot read property X of undefined".

### 8.4 `findOne` con varios matches

Devuelve el primero según orden interno de Mongo. **No determinístico.** Usá `findOne` solo cuando el criterio garantiza unicidad (matrícula, email, `_id`). Si no, usá `find` + `.sort()`.

---

## 9. Tres comportamientos a entender 🔴 🎯

### 9.1 Atributo NO declarado en schema → Mongoose lo IGNORA

```javascript
const userSchema = new mongoose.Schema({
  nombre: String,
  apellido: String,
  username: { type: String, required: true, unique: true },
  email: String
});

const User = mongoose.model('User', userSchema);

// POST con atributo extra "color"
const userCreado = await User.create({
  nombre: 'Ever',
  apellido: 'Lizarraga',
  username: 'ever123',
  email: 'ever@test.com',
  color: 'azul'                  // ← no está en el schema
});

// userCreado.color === undefined
// La base guardó solo los 4 campos del schema
```

**Defensa silenciosa** contra contaminar la base. Si esperabas guardar un campo y no aparece → chequeá el schema primero.

### 9.2 Falta un `required` → REVIENTA con 500

```javascript
// username es required, llega un POST sin username
await User.create({
  nombre: 'Ever',
  apellido: 'Lizarraga',
  email: 'ever@test.com'
});
// → ValidationError: User validation failed: username: Path `username` is required.
```

Sin `try/catch` en controller → cliente recibe **500** (debería ser **400** o **422**).

**Capas de validación:**

1. Controller con Zod → 400 si falla.
2. Service con reglas de negocio → 422 si falla.
3. Schema de Mongoose → última línea, idealmente nunca debería dispararse.

### 9.3 Campo `__v` automático

```javascript
{
  _id: ObjectId("65f1a..."),
  nombre: "Ever",
  apellido: "Lizarraga",
  username: "ever123",
  email: "ever@test.com",
  __v: 0                       // ← agregado por Mongoose
}
```

**Versionado interno** para detectar conflictos de concurrencia. Metadata de Mongoose, no campo tuyo. **No exponer en la API.**

---

## 10. `_id` y ObjectId — nunca exponer 🔴 🎯

Cada documento tiene `_id` automático de tipo **ObjectId** (24 caracteres hex, mezcla de timestamp + machine id + process id + contador).

### Por qué nunca exponer

**1. Seguridad — inyección NoSQL.** Rutas como `GET /api/medicos/:id` con `id` = `_id` directo se pueden atacar con payloads tipo `{"$ne": null}` y devolver todos los médicos.

**2. Acoplamiento.** Migrar a otra base (UUIDs, ints) rompe el contrato de la API.

**3. Filtración.** ObjectId contiene timestamp y datos del servidor.

### Qué hacer

**Opción A:** usar `_id` solo internamente (en Repository). Mapearlo a `id` (string) al construir la entidad de dominio.

**Opción B:** tener tu propio campo `id` (UUID) declarado en el schema.

```javascript
// En Repository, al construir entidad de dominio
function toDomain(doc) {
  return {
    id: doc._id.toString(),     // _id de Mongo → id de string para dominio
    nombre: doc.nombre,
    matricula: doc.matricula,
    // ... resto, SIN exponer _id ni __v
  };
}
```

---

## 11. Los TRES modelos del TP2 🔴🔴🔴 🎯

**Condición de corrección explícita.** Agus, textual: *"Esas tres cosas tienen que estar marcadas en el TP por una mínima diferencia, por favor."*

### Planteo

TP1 tenía dos representaciones (DTO Zod + Dominio). TP2 suma una tercera: el documento de Mongo con `_id`, `__v`, decisiones de serialización, embedded vs reference. Mezclar = mezclar capas, error penalizado.

### Los tres modelos

| Modelo | Capa | Para qué sirve | Particularidad |
|---|---|---|---|
| **Request DTO** ("Médico Request") | Controller | Lo que llega del cliente (body POST/PUT) | Validado con Zod. Sin lógica. Sin `_id`. |
| **Dominio** (`Medico`) | Domain/Service | Médico como concepto de negocio | Métodos de comportamiento. Sin `_id`. Sin persistencia. |
| **Schema/DAO** ("Médico Schema") | Repository | Documento como vive en Mongo | Tiene `_id`, `__v`. Definido con `mongoose.Schema`. |

### Flujo completo de una request

```
POST /api/medicos
Body: { "nombre": "Ana", "matricula": "M-789", "edad": 40 }

  1. CONTROLLER
     ─ Recibe body como DTO Request
     ─ Valida con Zod
     ─ Si falla: 400
     ─ Si OK: pasa DTO al Service
        ↓
  2. SERVICE
     ─ Recibe DTO
     ─ CONVIERTE a entidad de dominio (Medico)
     ─ Aplica reglas de negocio
     ─ Si regla falla: tira error de dominio (→ 422 vía middleware)
     ─ Si OK: llama Repository.guardar(medicoDeDominio)
        ↓
  3. REPOSITORY
     ─ Recibe entidad de dominio
     ─ CONVIERTE a documento (Schema/DAO)
     ─ MedicoModel.create({...})
     ─ Mongoose guarda (le pone _id, __v)
     ─ Devuelve entidad de dominio (NO el documento)
        ↑
  4. SERVICE → CONTROLLER
     ─ Controller SERIALIZA como respuesta (DTO de salida, sin _id ni __v)
     ─ res.status(201).json(respuesta)
```

**Cuatro conversiones explícitas:** DTO entrada → Dominio (Service), Dominio → Schema (Repository al guardar), Schema → Dominio (Repository al leer), Dominio → DTO salida (Controller al responder).

### Por qué no usar uno solo

- **Schema = Dominio:** lógica de negocio atada a Mongo. Migrar = refactor total. Métodos de dominio mezclados con métodos de Mongoose.
- **DTO = Dominio:** dominio depende de la forma del request. Cualquier cambio en API rompe dominio.
- **Schema = DTO salida:** exponés `_id`, `__v`, abrís puerta a inyección NoSQL.

Cuesta líneas, pero da independencia entre capas (lo que vendía la arquitectura por capas de semana 4 práctica, aplicada acá).

### Ejemplo concreto de las tres representaciones

```javascript
// 1. Request DTO (Controller — validado con Zod)
const MedicoRequestSchema = z.object({
  nombre:    z.string().min(3),
  matricula: z.string().regex(/^M-\d+$/),
  edad:      z.number().int().positive(),
  especialidades: z.array(z.string()).optional()
});
// → { nombre, matricula, edad, especialidades? }


// 2. Dominio (Domain — clase con comportamiento)
class Medico {
  constructor({ id, nombre, matricula, edad, especialidades }) {
    this.id = id;                    // string, no ObjectId
    this.nombre = nombre;
    this.matricula = matricula;
    this.edad = edad;
    this.especialidades = especialidades || [];
  }

  atiendeEspecialidad(esp) {
    return this.especialidades.includes(esp);
  }

  agregarEspecialidad(esp) {
    if (!this.atiendeEspecialidad(esp)) {
      this.especialidades.push(esp);
    }
  }
}


// 3. Schema / DAO (Repository — definido con Mongoose)
const medicoSchema = new mongoose.Schema({
  nombre:    { type: String, required: true },
  matricula: { type: String, required: true, unique: true },
  edad:      { type: Number, required: true },
  especialidades: [String]
});

const MedicoModel = mongoose.model('Medico', medicoSchema);
// Documento en Mongo tendrá ADEMÁS: _id, __v
```

> 🔴 **Para la corrección del TP2:** tres archivos distintos en el repo:
> - `controllers/MedicoController.js` (Zod schema del request)
> - `domain/Medico.js` (clase de dominio)
> - `repositories/MedicoRepository.js` + `infrastructure/schemas/MedicoSchema.js`
>
> Si no aparecen, es probable que estés mezclando capas.

---

## 12. Refactor TP1 → TP2 capa por capa 🟡 🎯

### Controller — casi no cambia

Sigue igual: recibe request, valida con Zod, llama service, mapea errores, devuelve response. Único cambio: `id` ahora viene mapeado del `_id` de Mongo.

### Service — casi no cambia

Sigue orquestando lógica. **Posible cambio:** operaciones que antes eran sincrónicas ahora son async — si tu repo devolvía objetos directos antes y ahora devuelve Promises, agregá `await`.

### Repository — cambia BASTANTE

**Antes (TP1, memoria):**

```javascript
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

**Ahora (TP2, Mongoose):**

```javascript
const MedicoModel = require('../infrastructure/schemas/MedicoSchema');
const Medico = require('../domain/Medico');

class MedicoRepository {
  // Conversión doc → dominio
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

  // Conversión dominio → doc (para create/update)
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

**Cambios clave:** todos los métodos async, aparecen `toDomain` y `toDocument`, IDs como strings (no ints), validaciones "no existe" con resultado de `find/findByIdAndDelete`.

### Domain — no cambia para nada

Las clases de dominio (`Medico`, `Turno`, `Plan`, etc.) quedan exactas. No saben que existe Mongo. **Premio por haber separado en capas:** cambio en una sola capa, las otras estables.

---

## 13. Feedback TP1 que aplica directo a TP2 🟡 🎯

### 13.1 Rutas no-REST

- ❌ `POST /crearMedico` → ✅ `POST /medicos`
- ❌ `GET /obtenerTurnosDelMedico/123` → ✅ `GET /medicos/123/turnos`
- ❌ `POST /turnos/cancelar/456` → ✅ `PATCH /turnos/456` o `DELETE /turnos/456`

### 13.2 Middleware de errores copiado pero no usado

```javascript
// ❌ Mal: try/catch + mapeo manual a HTTP en cada controller
async function crearMedico(req, res, next) {
  try {
    const medico = await medicoService.crear(req.body);
    res.status(201).json(medico);
  } catch (err) {
    if (err instanceof NotFoundError) return res.status(404).json({...});
    if (err instanceof ValidationError) return res.status(422).json({...});
    return res.status(500).json({ error: 'Error interno' });
  }
}

// ✅ Bien: dejar que el error suba al middleware único
async function crearMedico(req, res, next) {
  try {
    const medico = await medicoService.crear(req.body);
    res.status(201).json(medico);
  } catch (err) {
    next(err);  // middleware se encarga
  }
}
```

Middleware único en `app.use((err, req, res, next) => {...})` después de todas las rutas.

### 13.3 Promises mal usadas

Cinco grupos no usaron promises bien (sin `await` donde correspondía, o mezcla de patterns). Más crítico en TP2 porque todo Mongoose es async.

**Chequeo:** funciones de Service y Repository que llamen a Mongoose deben ser `async` y `await`ear.

### 13.4 Validaciones en Controller que iban a Zod o Service

**Reglas de capa:**
- **Forma del request** (campos, tipos): **Zod en Controller**.
- **Reglas de negocio** (edad mínima, matrícula única): **Service** con errores de dominio.
- **Estructura de documento**: **Schema de Mongoose**.

### 13.5 Errores HTTP-mapeados tirados desde el Dominio

```javascript
// ❌ Mal: dominio no debería conocer HTTP
class Medico {
  cancelarTurno(turno) {
    if (turno.fechaPasada()) {
      throw new BadRequestError('No se puede cancelar...');
    }
  }
}

// ✅ Bien: dominio tira error de dominio, controller/middleware mapea a HTTP
class Medico {
  cancelarTurno(turno) {
    if (turno.fechaPasada()) {
      throw new TurnoYaRealizadoError('No se puede cancelar un turno pasado');
    }
  }
}

// En middleware:
if (err instanceof TurnoYaRealizadoError) {
  return res.status(409).json({ error: err.message });
}
```

### 13.6 Operativos

- Mensajes de commit descriptivos (no `"test"`, `"fix"`, `"asd"`).
- No commitear después del horario de cierre (se nota).
- Preparar presentación 20 min antes (IDE, Postman, ejemplos cargados).
- El que no presentó TP1, presenta TP2.

---

## 14. Qué NO se cubrió bien (viene pronto) 📘

- **Embedded vs Referenced:** decisión cómo modelar relaciones en Mongo. **Clase práctica del sábado 09/05.**
- **Queries complejas:** `$lookup` (JOIN de Mongo), agregaciones, sort, skip/limit. Sábado.
- **Tests con mocks de Mongoose:** aplicación directa de la sección 1. Sábado.

**Próxima clase de teoría (12/05):** arranca Frontend. Cambio total de chip.

---

**FIN DEL APUNTE RESUMEN — SEMANA 6 TEORÍA**
