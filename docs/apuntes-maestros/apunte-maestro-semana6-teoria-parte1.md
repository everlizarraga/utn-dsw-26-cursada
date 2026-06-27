# Apunte Maestro — Semana 6 Teoría, Parte 1

## Mocking y panorama de persistencia

> **Semana 6 — Martes 05/05 — Profes: Agus (lidera) y Lucas (soporte)**
>
> **Sobre esta parte:** Primera de dos partes del apunte de teoría de la semana 6. La clase se dividió naturalmente en dos mitades. Esta primera parte cubre la mitad **conceptual**: el cierre del tema de mocking que había quedado pendiente, y el panorama completo de persistencia (qué tipos de almacenamiento existen, las dos grandes familias de bases de datos, ACID, teorema CAP). La **Parte 2** baja ese panorama al ODM concreto que vas a usar en el TP — Mongoose — con la demo en vivo y los tres modelos (Request, Dominio, Schema) que la cátedra dejó muy explícitos como condición de corrección.
>
> **Qué hay y qué no hay acá:** todo el armado conceptual sobre persistencia y bases de datos vive en este archivo. La aplicación práctica con Mongoose vive en la Parte 2. Si ya manejás bases de datos por afuera de la facultad, gran parte de esta primera parte te va a sonar a panorama ya conocido — está bien que sea así; la cátedra va por ese lado y lo que importa es alinearse con el lenguaje y los criterios que va a usar para evaluar.

---

## Info operativa

- **Horario martes:** 19:00 a ~22:00 presencial.
- **Profes del martes:** Agus lideró, Lucas soporte.
- **Material oficial subido:** PDF "Resumen Clase 6" (7 páginas). Atención: el header del PDF dice "Miércoles Noche" pero estás en martes. Es el mismo recycle de templates que ya pasó en la clase 4. **El contenido aplica igual**, solo está mal el rótulo de comisión.
- **Parcial confirmado:** sábado **27/06/2026**, de 9 a 11hs, en Medrano. Es parcial único para todas las comisiones (todos los cursos rinden el mismo día). Agus va a tratar de gestionar que se pueda rendir en Lugano para que les quede más cerca. **Sin confirmación todavía** — esperar novedades.
- **Recuperatorio:** también un sábado, 11/07/2026 según planificación.
- **Entrega 2 del TP:** 26/05 (semana 9). Al cierre de la clase del 05/05 todavía no estaba liberado el enunciado nuevo, lo iban a soltar "esta semana, ojalá antes que tarde". Cuando se libere, lo incorporamos al chat de la semana correspondiente.
- **Próxima clase teórica (12/05, semana 7):** intro a Frontend — HTML, CSS, JavaScript del lado cliente. Cambia el chip totalmente.
- **Clase del sábado siguiente (09/05, semana 6 práctica):** profundización de persistencia con Mongoose, ahora con todo el detalle de embedded vs reference. Esta clase del martes fue el "panorama"; el sábado entra en el código con los chicos del sábado.

---

## 0. De dónde venís y dónde te deja esta clase

En las clases anteriores construiste un backend en capas (Controller → Service → Repository → Domain) con Express y Zod. Tus repositorios todavía guardan los datos **en memoria**: un array dentro del proceso de Node, que se pierde apenas el servidor se reinicia. Eso fue suficiente para el TP1 — la propia consigna decía *"no se deben almacenar los datos en una Base de Datos. Mockear esa parte y guardar en memoria"*.

Ahora la cosa cambia. La entrega 2 del TP exige **persistencia real**: que los datos sobrevivan al reinicio, que existan en una base de datos de verdad, y que tu código sepa hablarle a esa base. Esta clase es el primer eslabón de ese salto. Te da:

1. El **cierre del tema de mocking**, que estaba previsto para la clase 4 pero el profe lo había diferido explícitamente para esta clase. Es importante porque, una vez que tengas base de datos real, vas a querer **testear sin tocarla** — y eso se hace con mocks.
2. El **mapa completo de persistencia**: qué opciones existen, qué garantías te da cada una, cómo razona la cátedra sobre el tema. La parte 2 de este apunte aterriza ese mapa a Mongoose; pero el mapa va primero.

Conexión con materias previas:

- De **Bases de Datos** (final pendiente, materia de 3er año) ya tenés o vas a tener todo el universo de SQL: relaciones, claves foráneas, normalización, ACID, transacciones. Esta clase apoya en ese conocimiento y agrega lo que la cátedra de DdS quiere que sepas: cómo se relaciona ese mundo con el otro lado, el de las NoSQL.
- De **Operativos** (final pendiente) probablemente viste el concepto de transacción atómica y de sistemas distribuidos. Si no, no es bloqueante: la clase introduce lo que necesitás.
- De **Paradigmas** ya tenés modelado de objetos, herencia y composición. La idea de mapear esos objetos a una base de datos (ORM/ODM) es justamente eso aplicado.

No vas a salir de esta clase sabiendo escribir Mongoose — para eso es la Parte 2. Salís sabiendo **qué decisiones se toman antes de elegir y usar una base de datos**, y por qué la cátedra eligió Mongo para el TP.

---

## 1. Mocking: cerrando el tema de testing 🔴

### 1.1 Por qué este tema está acá

En la clase 4 de teoría se vio toda la pirámide de tests, TDD y BDD. Pero **mocking quedó pendiente** — el profe Lucas lo había avisado explícitamente: *"esto lo paso a la próxima clase teórica, donde se ve persistencia, porque para testear repositories sin tocar la base, mockear es la herramienta"*. Esa "próxima clase teórica" es esta. Acá se cierra el círculo.

La conexión es directa. Cuando tu Repository solo tenía un array en memoria, podías testearlo sin problema: instanciás, le tirás operaciones, verificás. Pero cuando el Repository pasa a hablar con MongoDB, **ya no querés que tus tests prendan una base de datos real**. Sería lento, frágil, y dependería de tener Mongo levantado en cada máquina donde corra el test. Acá entra mocking: simulás la dependencia externa para poder testear lo tuyo.

### 1.2 Qué es un mock (en sentido amplio)

Un **mock** es la simulación de un componente dentro de un test. Vos le das el comportamiento que quieras y eso te permite **acotar lo que estás testeando**: te concentrás en una parte específica y ponés en pausa todo lo de alrededor.

Donde aparece típicamente:

- **En tests unitarios:** no querés testear lo que hace otra clase. Si tu Service depende de un Repository, en el test del Service mockés el Repository. La lógica del Repository tiene su propio test, no se mezclan.
- **En tests de integración:** no querés que tu test, al correr, escriba en la base de datos real ni le pegue a una API externa real. Mockés la base, mockés el cliente HTTP, y verificás que tu código se comporte como debe.

Lo que mockés puede ser **un objeto, una base de datos, un servicio externo** — cualquier cosa que sea una dependencia de lo que querés probar. Y como vos decidís cómo se comporta el mock, podés simular escenarios que serían imposibles o costosos de reproducir con la cosa real:

- ¿Qué pasa si el servicio externo me devuelve un valor inesperado?
- ¿Qué pasa si la base de datos lanza un error?
- ¿Qué pasa si la API que consumo me devuelve un 404?

Todos esos casos los podés testear porque el mock responde lo que vos le digas que responda.

### 1.3 Los tres tipos: Stub, Mock y Spy

La palabra "mock" se usa en dos sentidos en la industria. En sentido amplio, "mockear" es cualquier forma de simular una dependencia. En sentido estricto, "mock" es uno de los tres tipos específicos. Para no marearte, en este apunte uso **"mockear"** para la acción genérica y **Stub / Mock / Spy** para los tres tipos puntuales.

#### Stub — el objeto tonto 🔴

Un **Stub** es un objeto falso que te devuelve datos prefijados. No ejecuta lógica real. Su único trabajo es responder algo cuando lo llamás. Sirve para que la función bajo test pueda trabajar sin depender de recursos externos.

La forma de pensarlo: *"cuando te llame con este método y este parámetro, devolveme esto otro"*. Punto. No te importa nada más.

Ejemplo en Jest:

```javascript
// Estoy testeando algo que necesita un repository de productos.
// En vez de instanciar el repository real (que iría a la base),
// armo un stub que responde lo que yo decida.

const productRepositoryStub = {
  // Cuando alguien llame a getProductById con cualquier id,
  // devuelvo siempre este objeto. Hardcodeado.
  getProductById: (id) => ({
    id,
    name: "Camiseta",
    price: 500
  })
};

// Y en el test, le paso este stub al code bajo test:
const product = productRepositoryStub.getProductById(productId);
// product es siempre { id, name: "Camiseta", price: 500 }
```

**Cuándo usar Stub:** cuando el test necesita que la dependencia "esté ahí y devuelva algo razonable", pero no te importa **qué hizo el test con esa dependencia**. Solo te importa el resultado de lo que estás testeando.

> 🔴 **Anotalo:** Agus dijo en clase, casi al pasar pero es importante: *"En el TP casi siempre van a usar Stub"*. Si tu Service depende del Repository, en los tests del Service el Repository va a ser un Stub que devuelve los productos / médicos / turnos que vos quieras inyectar para armar el caso. Es la herramienta principal del TP2.

#### Mock — el objeto que registra interacciones 🔴

Un **Mock** se usa cuando lo que te importa **no es solo qué devuelve la dependencia, sino cómo la usaron**: ¿se llamó al método correcto? ¿con qué parámetros? ¿cuántas veces?

La forma de pensarlo: *"validame que me hayan llamado X veces, con Y parámetros"*.

Ejemplo en Jest:

```javascript
// Función bajo test: registra un usuario y le manda un mail de bienvenida.
function registerUser(email, mailer) {
  // ... resto del código de registro ...
  mailer.sendWelcome(email);
}

// En el test:
const mockMailer = {
  sendWelcome: jest.fn()  // jest.fn() crea una función mock vacía,
                          // que no hace nada pero registra todo lo que recibe.
};

registerUser("email@test.com", mockMailer);

// Verificaciones de interacción — ESTO es lo que distingue un Mock:
expect(mockMailer.sendWelcome).toHaveBeenCalledTimes(1);
expect(mockMailer.sendWelcome).toHaveBeenCalledWith("email@test.com");
```

**La diferencia clave con Stub** es el foco:

- **Stub:** *"cuando te llamen, respondé esto"*. Mirás la salida.
- **Mock:** *"verificá que te hayan llamado correctamente"*. Mirás la entrada y la interacción.

Un caso típico de Mock en el contexto del TP: querés testear que tu Service, al cancelar un turno, **efectivamente llama al método save del Repository con el turno cancelado**. No te importa qué devuelve el save (eso lo prueba otro test del Repository). Te importa que tu Service le hable bien al Repository.

#### Spy — el espía que deja correr al objeto real 🟡

Un **Spy** es distinto a los dos anteriores: en vez de reemplazar el objeto, **lo envuelve**. El método real se sigue ejecutando, pero el spy registra qué pasó (cuántas veces se llamó, con qué parámetros, qué devolvió).

Es lo que en patrones de diseño llamarías un wrapper, proxy o decorator del objeto real. Lo "decora" con capacidad de auditar.

Ejemplo en Jest:

```javascript
const calculadora = {
  sum: (a, b) => a + b
};

// En el test:
const spy = jest.spyOn(calculadora, "sum");
// El método sum sigue siendo el sum real — calcula bien.
// Pero spy registra cada llamada.

const result = calculadora.sum(2, 3);
// result === 5 (la suma se ejecutó de verdad)

// Y puedo verificar la interacción:
expect(spy).toHaveBeenCalledWith(2, 3);
```

**Cuándo se usa:** Agus fue honesto — *"el Spy no se usa mucho. Yo no sé si lo usé un montón"*. Sirve para casos donde querés que el método haga su trabajo real (porque mockearlo sería incorrecto o costoso), **pero también querés auditar que se haya llamado de la forma esperada**.

Para el TP probablemente no lo necesites. Lo importante es saber que existe y que es distinto a los otros dos: Stub y Mock **reemplazan** el objeto, Spy **envuelve** al objeto real.

### 1.4 Tabla de bolsillo

| Tipo | ¿Reemplaza al original? | ¿Te importa qué devuelve? | ¿Te importa cómo lo llamaron? | Uso típico en el TP |
|---|---|---|---|---|
| **Stub** | Sí | Sí, vos definís la respuesta | No | **Principal:** Service → Repository |
| **Mock** | Sí | Lo definís, pero no es el foco | Sí, validás interacción | Verificar que se llame `save`, `delete`, `notify` |
| **Spy** | No, envuelve al real | El real define la respuesta | Sí, validás interacción | Casos puntuales |

### 1.5 Conexión con el TP2

La consigna del TP2 pide tests unitarios sobre **capa de Services y capa de Dominio**. En esos tests:

- Cuando testees una clase de Dominio (ej: `Turno`, `Plan`, `CoberturaEspecialidad`), **no necesitás mockear nada**. El Dominio no depende de la base ni de servicios externos — es lógica pura. Le pasás datos, validás resultado.
- Cuando testees un Service (ej: `TurnoService.reservar()`), **sí necesitás mockear el Repository** y, según el caso, otros Services o el FactoryNotificacion. Acá entra Stub mayormente, y Mock cuando quieras verificar que efectivamente se llamó a `save`, `notify`, etc.

> **Para el parcial, si te preguntan:**
>
> **P: Diferenciá Stub, Mock y Spy. Dá un ejemplo de cuándo usarías cada uno.**
>
> **R:** Los tres son técnicas para simular dependencias en un test. **Stub** es un objeto falso que devuelve datos prefijados sin ejecutar lógica real; lo uso cuando necesito que una dependencia "esté ahí" y responda algo razonable, pero solo me importa el resultado de la función bajo test (ej: stubear el Repository para devolver una lista de productos al testear un Service). **Mock** también reemplaza la dependencia, pero la diferencia es que registra cómo lo usaron — lo uso cuando me importa verificar la interacción (ej: validar que el Service llame al método `save` del Repository con el objeto correcto, una sola vez). **Spy** no reemplaza al objeto: lo envuelve y deja que el método real se ejecute, pero registra qué pasó; lo uso cuando quiero que el comportamiento real ocurra pero también auditar las llamadas (ej: validar que un método auxiliar se invocó la cantidad correcta de veces dentro de una operación más grande).

---

## 2. Persistencia: qué es y qué opciones hay 🟡

### 2.1 La definición que pidió Agus

**Persistencia** es la capacidad de almacenar datos de forma duradera, de modo que no se pierdan cuando la aplicación se cierra o el sistema se apaga.

Una segunda dimensión que Agus subrayó: además de durar, los datos tienen que ser **accesibles** — pero no de forma libre, controlada. Alguien (o algo) tiene que poder leerlos, y alguien (o algo) NO tiene que poder leerlos. La seguridad y los permisos son parte del problema de persistencia, no un agregado posterior.

### 2.2 Tipos de almacenamiento

Persistencia no es "base de datos" sin más. Hay un abanico de opciones, cada una con su contexto de uso:

| Tipo | Ejemplos concretos | Cuándo se usa |
|---|---|---|
| **Archivos** | `.txt`, `.csv`, `.json`, `.xls`, Google Sheets | Datos chicos, configuración, exportes |
| **Bases de datos** | MySQL, MariaDB, PostgreSQL, MongoDB, Cassandra, Redis, Neo4J | Datos estructurados que se consultan, modifican y consultan de muchas formas |
| **Almacenamiento en la nube** | Amazon S3, Google Cloud Storage | Archivos grandes, snapshots, contenido binario (imágenes, videos), backups |
| **Sistemas distribuidos / Big Data** | Apache Hadoop, Apache Iceberg | Volúmenes enormes (millones y millones de registros) que no caben en una sola máquina |

Sobre los dos últimos:

- **Almacenamiento en la nube (S3 y similares):** vos te conectás como si fuera un FTP a un servidor remoto. No es una base de datos — es un disco que vive en la nube. Se usa mucho para snapshots de bases, para guardar contenido pesado, para backups. Es ortogonal a la base: muchas veces tu app tiene SQL para los datos transaccionales y S3 para guardar las fotos asociadas.
- **Sistemas distribuidos:** entran cuando la información es tan grande que ningún servidor único la banca. Es el mundo de Big Data, BI, data mining. La cátedra los nombra para que sepan que existen, pero **no los van a usar en el TP**.

Hay un concepto más que apareció al pasar: **Datalake**. No es exactamente almacenamiento — es una capa de presentación de datos, típicamente para equipos de Business Intelligence (BI). La idea: tenés tu base de datos transaccional con los datos del producto, y exportás copia de esos datos al datalake con otra forma (ya transformados, agregados, anonimizados) para que los equipos de análisis lo consulten sin tocar la base de producción. Mención cultural; no entra en el TP ni en el parcial.

### 2.3 Por qué se eligió MongoDB para el TP

Decisión pedagógica explícita de la cátedra. Agus lo dijo: *"En el TP les decimos usen Mongo para que no usen SQL, porque SQL ya la van a ver en Bases de Datos y en Diseño, después más adelante también la van a ver. Es como medio repetitivo. Podríamos haber dicho usen lo que sea — la realidad es que se podría usar PostgreSQL — pero JSON en Mongo es una experiencia distinta, los objetos JSON de JavaScript se guardan casi igual."*

O sea: la decisión es por exposición a una tecnología distinta a la que ya van a ver mil veces, no porque Mongo sea técnicamente superior para este problema.

---

## 3. SQL vs NoSQL: las dos grandes familias 🔴

Toda base de datos cae en uno de dos grupos:

- **Relacionales (SQL):** los datos viven en **tablas** con esquema fijo (columnas predefinidas, tipos predefinidos). Las tablas se relacionan entre sí con claves foráneas. El lenguaje de consulta es SQL — declarativo, bastante estandarizado entre motores.
- **No relacionales (NoSQL):** todo lo que no es SQL. No hay un único modelo — hay varios, cada uno pensado para un problema distinto.

### 3.1 SQL: las que van a aparecer en tu vida cien mil veces

Las relacionales son las "tradicionales". Algunas que la cátedra nombró:

- **MySQL** y su fork libre **MariaDB** (mascota: la foca). Muy usadas, mucha documentación.
- **PostgreSQL** (mascota: el elefantito). Más robusta en features avanzadas, está cada vez más de moda. Existe **Supabase**, que es básicamente PostgreSQL gestionado en la nube con extras.
- **SQL Server** (Microsoft). Vas a verla mucho en empresas con stack Microsoft.
- **DB2** (IBM). Más institucional.
- **Oracle**. La pesada del barrio enterprise.

**Cuándo conviene SQL:**

- Cuando los datos tienen **estructura estable** que no va a cambiar tanto. Stock, transacciones, ventas, productos consolidados.
- Cuando necesitás **consistencia fuerte**: usuarios, contraseñas, datos de plata, transacciones financieras. Si dos clientes consultan el saldo, los dos tienen que ver el mismo número.
- Cuando aprovechás los **JOIN**: relaciones complejas entre entidades donde la normalización tiene sentido.

**Trade-off:** modificar el esquema de una tabla con muchos registros es caro y arriesgado. `ALTER TABLE` en una tabla grande puede bloquear la base por un rato y romper aplicaciones en vivo. Por eso las migraciones de SQL serias se planifican y se hacen en ventanas de bajo tráfico.

### 3.2 NoSQL: cuatro familias (y una más, de yapa)

NoSQL no es una cosa, son cuatro grandes formas de modelar datos sin tablas clásicas.

#### Documental 🔴 — la que vas a usar

Almacena **documentos**, típicamente en formato JSON (o BSON, que es JSON binario, más compacto). Cada documento es como un objeto JavaScript: tiene campos, valores, puede contener objetos anidados, listas.

**Ejemplos:** **MongoDB** (la que usás en el TP), CouchDB.

Una clave: las bases documentales son **schemaless** — no necesitás definir el esquema rígidamente antes de guardar nada. Podés guardar un documento con cinco campos hoy, y mañana guardar otro con siete campos en la misma colección. La base no protesta.

> ⚠️ **Cuidado con simplificar:** "schemaless" se aplica a las **documentales**, no a TODAS las NoSQL. Las columnares (Cassandra) sí tienen esquema. Mezclarlo es un error común.

#### Columnar 🟢

**Ejemplo:** Cassandra.

Las columnares almacenan datos en estructuras tipo tabla pero distribuidas por columna, no por fila. Tienen una particularidad pedagógica importante: **el modelo se diseña a partir de las consultas que vas a hacer, no a partir de los datos**. Vos primero decís "qué quiero consultar" y desde ahí derivás el esquema. Es lo opuesto a la lógica relacional.

El lenguaje se llama CQL (Cassandra Query Language) — parecido a SQL pero adaptado.

Para el TP no la vas a tocar. Es importante saber que existe y por qué es distinta.

#### Clave-valor 🟢

**Ejemplo:** Redis.

Es como un mapa o diccionario: una clave única apunta a un valor. Optimizado para **acceso ultrarrápido**, casi siempre vive en memoria RAM (no en disco), distribuido entre nodos.

**Uso típico:** caché. Suponé que tu API tiene búsquedas que muchos usuarios hacen igual. En vez de ir a la base cada vez, guardás el resultado en Redis con una clave del estilo `busqueda:medico:cardiologia:caballito` y cuando llega la misma búsqueda devolvés el resultado cacheado. Si pasaron más de N minutos, invalidás la caché y vas a la base.

No reemplaza a la base — la complementa.

#### Grafos 🟢

**Ejemplo:** Neo4J.

Modelan los datos como un grafo: nodos (entidades) conectados por aristas (relaciones), donde las aristas pueden tener propiedades (peso, tipo, dirección).

**Casos de uso típicos:**

- **Sistemas antifraude.** Cada operación es un nodo, cada conexión (ej: misma tarjeta usada por dos cuentas, mismo dispositivo, misma IP) es una arista. Recorrés el grafo asignando pesos y al final tenés un score de riesgo.
- **Recomendaciones.** "Personas que conocés" en LinkedIn, sugerencias de amigos en Facebook/Instagram. El grafo de relaciones se camina para encontrar caminos cortos hasta gente nueva.
- **Mapas y rutas.** Cada intersección es un nodo, cada cuadra es una arista con distancia/tiempo como peso.

#### Vectoriales 🟢 — la nueva moda

**Ejemplos:** Pinecone, Weaviate, Chroma.

Son la onda de los últimos años por la IA. Almacenan vectores numéricos (embeddings). Cuando hacés una pregunta a un LLM, el texto se transforma matemáticamente en un vector, y la base devuelve los documentos cuyos vectores estén "cerca" matemáticamente. Es la base del concepto de **RAG** (Retrieval Augmented Generation) que está de moda.

Mención cultural. No entra en el parcial. La cátedra solo la nombró para que sepan que existe.

### 3.3 Cuándo elegir SQL y cuándo NoSQL

No hay una regla absoluta. Depende del problema. Algunas heurísticas que se mencionaron en clase:

| Si tu necesidad es... | Elegí... |
|---|---|
| Datos críticos, consistencia fuerte (plata, usuarios, transacciones) | SQL |
| Estructura estable, relaciones bien definidas | SQL |
| Reportes con muchos JOINs | SQL |
| Esquema flexible que va a evolucionar | Documental (Mongo) |
| Alta concurrencia, datos tipo "feed" o "posts" | Documental |
| Caché de búsquedas, datos volátiles ultrarrápidos | Clave-valor (Redis) |
| Análisis de relaciones (fraude, recomendaciones) | Grafos (Neo4J) |
| Volumen masivo, distribución horizontal extrema | Columnar (Cassandra) |

**Frase clave de Agus, que vale el oro:**

> *"Lo que no son herramientas, son cosas que vos usás como solución a tu problema. Vos sos la que hablás con ellas; ellas no hablan entre sí. Capaz tu solución usa MariaDB para usuarios, Cassandra para feed y Redis para caché — y está bien. La base de datos no es UNA decisión, son varias decisiones de diseño."*

En sistemas reales, lo común es **combinar** varias bases, cada una para lo que mejor hace.

---

## 4. ACID: las garantías de las relacionales 🔴

ACID es un acrónimo que describe las cuatro propiedades que tradicionalmente garantizan las bases de datos relacionales. Es uno de esos temas que **caen seguro en parcial**.

Antes de las cuatro letras, una palabra clave: **transacción**. Una transacción es una unidad de trabajo que la base trata como un todo. Puede ser una sola operación (`INSERT`) o varias (sacar plata de una cuenta + sumarla a otra cuenta). Lo importante es que la base sabe dónde empieza y dónde termina.

### A — Atomicity (Atomicidad) ⚛️

Una transacción es **indivisible**: o se ejecuta completa, o no se ejecuta nada. **Todo o nada.** No hay estados intermedios.

Si en el medio falla algo, la base hace **rollback**: vuelve todo al estado anterior, como si nada hubiera pasado. Si todo sale bien, la base hace **commit** y los cambios quedan firmes.

**Ejemplo clásico — transferencia bancaria:**

1. Restar $1000 de la cuenta A
2. Sumar $1000 a la cuenta B

Si los dos pasos están en una transacción y el segundo falla (caída de red, error de validación, lo que sea), la base revierte el primero. La cuenta A no queda con menos plata si la B no recibió. La plata no se pierde en el limbo.

### C — Consistency (Consistencia) ⚖️

La base de datos siempre queda en un **estado válido** antes y después de cada transacción. Las reglas de integridad (claves foráneas, restricciones, triggers) se respetan.

Si una transacción intentaría dejar la base en un estado inválido, no se aplica.

> **Ojo:** "Consistency" en ACID es distinto de "Consistency" en CAP (que ya vamos a ver). Es la misma palabra para dos cosas distintas. Para ACID, consistencia = estado válido. Para CAP, consistencia = todos los nodos ven lo mismo. Las cátedras a veces lo mezclan; no te dejes confundir.

### I — Isolation (Aislamiento) 🔒

**Las transacciones concurrentes no se pisan entre sí.** Cada una se ejecuta como si fuera la única que está corriendo en la base.

Sin aislamiento, podrías tener cosas como dos usuarios comprando el último producto en stock al mismo tiempo, y los dos confirmando — porque ninguno vio que el otro estaba en proceso de compra. Con aislamiento, la base coordina y solo uno termina exitoso.

### D — Durability (Durabilidad) 💾

Una vez que una transacción se confirmó (commit), **los cambios son permanentes**. Sobreviven a fallos del sistema, cortes de luz, lo que sea. Aunque se apague todo, cuando vuelva a prender los datos están.

Se implementa con logs de transacciones que se escriben a disco antes de confirmar, y mecanismos de recuperación al reiniciar.

### Aclaración importante de la cátedra

Agus subrayó: *"ACID es un concepto fuerte de las relacionales, pero algunas no relacionales también lo cumplen. MongoDB en versiones modernas mantiene ACID a nivel documento."*

En la práctica, esto es **un poco más matizado** de lo que dijo en clase: Mongo da ACID a nivel **un solo documento** desde siempre, y a nivel **multi-documento (transacciones distribuidas)** desde la versión 4.0/4.2 con limitaciones de performance y configuración. No es exactamente "Mongo cumple ACID así nomás" — depende del alcance.

> **Para el parcial:** respondé lo que dijo cátedra ("ACID es propio de relacionales pero algunas NoSQL como Mongo lo cumplen"). Si te preguntan más detalle, mencionar "a nivel documento" suma puntos sin contradecir.

> **Para el parcial, si te preguntan:**
>
> **P: ¿Qué es ACID? Explicá cada letra.**
>
> **R:** ACID es el acrónimo que describe las cuatro propiedades que garantizan la confiabilidad de las transacciones en bases de datos relacionales. **Atomicity** (atomicidad): una transacción es una unidad indivisible — o se ejecuta completa o no se ejecuta nada; si algún paso falla, se hace rollback de todo. **Consistency** (consistencia): la base de datos siempre queda en un estado válido antes y después de cada transacción, respetando las reglas de integridad definidas. **Isolation** (aislamiento): las transacciones concurrentes no interfieren entre sí; cada una se ejecuta como si fuera la única. **Durability** (durabilidad): una vez que una transacción se confirma con commit, los cambios son permanentes y sobreviven a fallos del sistema.

---

## 5. Teorema CAP: el trade-off de las distribuidas 🔴

ACID describe lo que te garantizan las bases tradicionales centralizadas. Pero cuando una base se distribuye en varios nodos (varios servidores, posiblemente en distintas regiones), aparecen tensiones nuevas. El **teorema CAP** describe esas tensiones.

CAP dice: en una base de datos distribuida, hay tres propiedades deseables, **pero solo podés tener dos al mismo tiempo**.

### Las tres propiedades

**🔄 Consistency (Consistencia)**

Todos los nodos ven los mismos datos al mismo tiempo. Cualquier lectura recibe la escritura más reciente — o un error si no se puede garantizar.

> *"Si hago un posteo nuevo en Instagram, todos mis seguidores deben ver inmediatamente la nueva información."*

**⚡ Availability (Disponibilidad)**

El sistema sigue respondiendo aunque algunos nodos fallen. Cualquier solicitud recibe una respuesta (sin garantizar que la respuesta sea la más reciente).

> *"Puedo seguir usando WhatsApp aunque algunos servidores estén caídos."*

**🔌 Partition Tolerance (Tolerancia a particiones)**

El sistema sigue operando aunque haya fallos de red entre los nodos. Si un nodo en Buenos Aires no puede hablar con uno en Miami, el sistema no se para.

> *"El sistema funciona aunque se corte la conexión entre centros de datos."*

### "Elige dos de tres"

La regla central del teorema: en presencia de fallos de red (que en sistemas distribuidos son inevitables), tenés que elegir entre **consistencia** y **disponibilidad**. Esto da tres combinaciones posibles:

| Combinación | Qué sacrifica | Ejemplos |
|---|---|---|
| **CA** (Consistency + Availability) | Tolerancia a particiones | Bases relacionales tradicionales en un solo nodo (MySQL, PostgreSQL standalone) |
| **CP** (Consistency + Partition Tolerance) | Disponibilidad | **MongoDB**, HBase, Redis. Si no puede garantizar consistencia, prefiere no responder. |
| **AP** (Availability + Partition Tolerance) | Consistencia | Cassandra, DynamoDB. Siempre responde, aunque los datos estén un poco desactualizados. |

### El gráfico que dibujó Agus en pizarra

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
       (queda abajo, sin C)
```

Cada vértice es una propiedad. Cada lado del triángulo es una combinación de dos propiedades — las que están en cada extremo de ese lado. La combinación opuesta al vértice es la que SACRIFICA esa propiedad.

### Aclaración honesta sobre el matiz

Agus avisó algo que vale la pena registrar literal: *"Esto era cuando se creó CAP. Ahora en la actualidad, las versiones más nuevas de Mongo cumplen las tres y te dan todo lo de ACID. Está bastante refinado."*

Esto es **parcialmente cierto**: las bases modernas tienen mecanismos sofisticados (sharding, replica sets, write concerns ajustables) que les permiten aproximarse a las tres propiedades en condiciones normales. Pero ante una partición real de red, la elección sigue existiendo: o respondés con datos potencialmente viejos (AP) o no respondés (CP). El teorema sigue siendo cierto en sus términos originales; lo que cambió es que las bases se volvieron más finas en cómo manejan el trade-off.

> **Para el parcial:** si te preguntan dónde cae Mongo, respondé **CP**. Es lo que dice el material oficial. El matiz "las versiones nuevas son más flexibles" suma si tenés tiempo, pero no contradigas la respuesta principal.

### Conceptos que aparecen al pasar (replication factor y quórum)

Agus mencionó dos términos que vienen de Cassandra (donde el modelo de replicación es muy explícito):

- **Replication factor:** cuando guardás un dato, ¿en cuántos nodos se replica? Si tenés 3 nodos y replication factor 2, cada dato vive en 2 nodos.
- **Quórum:** la mitad más uno de los nodos. Si tenés 3 nodos y querés leer un dato, podés exigir que al menos 2 de los 3 nodos te confirmen el mismo valor antes de aceptarlo como correcto.

Para el TP no los necesitás, pero son palabras que aparecen en sistemas distribuidos en general. Mención cultural.

> **Para el parcial, si te preguntan:**
>
> **P: ¿Qué es el teorema CAP? Explicá las tres propiedades y la regla de "dos de tres".**
>
> **R:** El teorema CAP describe las tres propiedades deseables de un sistema de base de datos distribuido y establece que solo se pueden garantizar dos simultáneamente. Las propiedades son: **Consistency** (todos los nodos ven los mismos datos al mismo tiempo, cualquier lectura devuelve la escritura más reciente o un error), **Availability** (el sistema sigue respondiendo aunque algunos nodos fallen), y **Partition Tolerance** (el sistema sigue operando aunque haya fallos de red entre nodos). En presencia de una partición de red, hay que elegir entre consistencia y disponibilidad: las bases CP como MongoDB sacrifican disponibilidad por consistencia (prefieren no responder antes que dar datos potencialmente incorrectos); las bases AP como Cassandra sacrifican consistencia por disponibilidad (siempre responden, aunque los datos puedan estar desactualizados); las bases CA como las relacionales tradicionales no toleran particiones de red, por eso suelen vivir en un solo nodo lógico.

---

## 6. Preguntas que quedaron flotando (las cierro acá)

Mientras procesaba la clase aparecieron dos preguntas que vale la pena responder explícitamente, porque pueden quedar dando vueltas:

**¿Por qué Mongo es CP y no CA?**

Porque Mongo está pensada para ser **distribuida** (replica sets, sharding). En un sistema con un solo nodo no tenés problema de partición, pero tampoco tenés tolerancia a fallos. Mongo ya parte de la base de que va a vivir en varios nodos, entonces tiene que tomar postura sobre qué hacer cuando esos nodos no se pueden hablar entre sí. Y elige consistencia: prefiere bloquearse antes que devolver datos desfasados.

**Si el teorema CAP dice "elegí dos de tres", ¿por qué los tres puntos del triángulo (las combinaciones CA, CP, AP) son las únicas opciones? ¿No podrías tener "una sola" o "ninguna"?**

Buena pregunta sistémica. La respuesta: en un sistema distribuido **realista** (donde la red puede fallar), Partition Tolerance no es opcional — si la red falla y tu sistema no la tolera, simplemente se cae. Entonces, en la práctica, la decisión real es entre **CP y AP**. Las bases CA suponen una red perfectamente confiable, lo cual solo se cumple en un único servidor sin replicación; ahí no hay decisión que tomar.

---

## Checkpoint Parte 1

Antes de pasar a la Parte 2 (que entra a Mongoose en código), verificá que podés responder estas sin volver al apunte:

1. ¿Por qué se vio mocking en esta clase y no en la clase 4 de testing? ¿Cuál es la conexión técnica entre mocking y persistencia?
2. Definí Stub, Mock y Spy. Para cada uno, dá un caso del TP donde lo usarías.
3. ¿Cuál de los tres tipos de mocking dijo Agus que es el "principal" para los tests del TP? ¿Por qué tiene sentido?
4. ¿Cuál es la diferencia conceptual entre Stub y Mock en términos de *qué te importa* del comportamiento simulado?
5. ¿Por qué el Spy es el único que no reemplaza al objeto original?
6. Definí persistencia en una oración. ¿Qué dimensión más allá de "almacenar" subrayó Agus?
7. Mencioná los cuatro grandes tipos de almacenamiento que vio la clase, con un ejemplo concreto de cada uno.
8. ¿Cuáles son las dos grandes familias de bases de datos? ¿Por qué la cátedra eligió Mongo y no SQL para el TP?
9. Mencioná las cuatro familias principales de NoSQL con un ejemplo y un caso de uso típico de cada una. (Bonus: la quinta que es novedad.)
10. Explicá las cuatro letras de ACID con una oración cada una. ¿Qué propiedad garantiza el rollback?
11. ¿Por qué "consistency" significa cosas distintas en ACID y en CAP?
12. ¿Qué dice el teorema CAP? Nombrá las tres propiedades y las tres combinaciones posibles, con un ejemplo de base para cada combinación.
13. ¿Dónde cae Mongo en CAP? ¿Qué sacrifica?
14. ¿Por qué Agus dijo que "Mongo cumple las tres ahora"? ¿Es estrictamente cierto? ¿Cómo lo respondés en parcial?
15. ¿Qué es replication factor? ¿Qué es quórum?

> Las respuestas en formato examen están en `complemento-semana6-teoria.md`. Las aclaraciones extendidas sobre temas que pueden generar duda están también ahí.

---

## Qué viene en la Parte 2

La Parte 2 baja todo este panorama a Mongoose concreto. Cubre:

- ORM vs ODM en detalle, con la comparación de Hibernate (Java) vs Mongoose (Node).
- Cómo se define un Schema en Mongoose, con `type`, `required`, `unique`.
- Cómo se crea un Model y cómo se hacen operaciones básicas: `create`, `find`, `findOne`.
- La demo en vivo que mostró Agus: qué pasa cuando le mandás un atributo extra (lo ignora), qué pasa cuando te falta un required (revienta con 500), qué hace el `_id` interno de Mongo.
- **Los tres modelos del TP2** (Request DTO / Dominio / Schema) — el tema que la cátedra explicitó como condición de corrección. *"Por una mínima diferencia, por favor."*
- El feedback que Agus y Lucas dieron sobre la entrega 1, que es directamente aplicable a la entrega 2.

---

**FIN DEL APUNTE MAESTRO — SEMANA 6 TEORÍA, PARTE 1**
