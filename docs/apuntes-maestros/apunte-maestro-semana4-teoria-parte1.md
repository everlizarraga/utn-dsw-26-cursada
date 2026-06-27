# Apunte Maestro — Semana 4 Teoría, Parte 1

## Testing: el mapa conceptual

> **Semana 4 — Martes 21/04 — Profes: Lucas (lidera) y Facu (soporte)**
>
> **Sobre esta parte:** Primera de dos partes del apunte de teoría de la semana 4. Acá levantamos el mapa: para qué testeamos, qué tipos de tests existen, cómo se ordenan en la famosa pirámide, qué significa caja blanca / caja negra y qué es la cobertura. Es el "panorama" de testing — el lenguaje compartido que vas a usar el resto de la cursada y que la cátedra puede preguntar en parcial. La **Parte 2** cubre las dos metodologías que se enseñaron en clase: **TDD** (red-green-refactor) y **BDD** (given-when-then), con los ejemplos en vivo de la caja de ahorro.
>
> **Sobre el nivel de detalle:** Asumo que ya manejaste tests unitarios en JS Essentials (Jest, `describe`/`it`, asserts). Esta clase es más bien la **formalización académica** del tema — ordenar y nombrar lo que ya en parte intuís. No te voy a re-explicar qué es un test ni cómo se corre Jest; me voy a enfocar en la nomenclatura de cátedra, las distinciones formales, y los puntos donde la cátedra hace cosas particulares (orden específico de la pirámide, criterio de caja blanca/negra para integración, etc.).

---

## Info operativa

- **Horario martes:** 19:00 a ~22:30 presencial.
- **Profes del martes:** Lucas y Facu.
- **Comunicación:** Discord para consultas; encuesta de slots para corrección de TPs (votar por fuera del horario de cursada, son ~20 min por grupo).
- **Sobre el TP en defensa:** Lucas pidió **venir practicados**, no leer pantalla, rotar quién habla entre entregas. Es lo más eficiente para los 20 min asignados.
- **Material oficial subido:** PDF "Resumen Clase 4" (4 páginas) — fijate que internamente dice "Miércoles Noche", es un lapsus del template (usan el mismo resumen para todas las comisiones), no un error de contenido.
- **Tests para el TP1 (28/04, ya pasó):** se pidió mínimo **2 tests unitarios**.
- **Tests para el TP2 (26/05):** según anuncio del profe, **"es más fuerte de tests"** — esperá tests unitarios sobre capa servicios y dominio + documentación con Swagger.
- **Mocking:** estaba previsto en esta clase pero el profe lo pasó explícitamente a la próxima (clase de persistencia, 05/05). En este apunte **no aparece** porque no se enseñó.

---

## 0. Conexión con lo que ya viste

En las semanas anteriores construiste un backend en capas (controller → service → repository) con Express y Zod. Ahora aparece la pregunta inevitable: *¿cómo me aseguro de que todo eso funciona y va a seguir funcionando cuando le siga agregando cosas?* Esa es la pregunta que esta clase responde.

Recordá que en semana 3 práctica armaste la arquitectura por capas del MegaSuper: cada capa hace una cosa y delega lo demás. **Esa arquitectura no se diseñó solo para ordenar el código** — también se diseñó para que cada pieza sea testeable por separado. El service se puede testear sin levantar Express; el repository se puede testear sin tocar la base de datos real (eventualmente, mockeando). Esa es una de las razones por las que te enseñaron a separar capas antes de enseñarte a testear: testing y arquitectura por capas son hermanos.

---

## 1. ¿Por qué testeamos? 🔴

Antes de meterte en *cómo* se testea, conviene clavar bien *para qué*. La cátedra fija siete razones (las que están en la pizarra del 21/04). Te las desgrano con el matiz que cada una tiene, porque en parcial te pueden pedir nombrarlas y explicar al menos algunas.

### 1.1 Refactor seguro

Si querés mejorar el código sin tener miedo de romperlo, necesitás una red. Los tests son esa red: corrés la batería antes y después del cambio, y si todo sigue verde, refactorizaste sin romper nada visible. Sin tests, refactorizar es ruleta rusa: hacés un cambio "inocente" y media app deja de funcionar a las dos semanas.

### 1.2 Documentar

Un test bien escrito es **un ejemplo de uso ejecutable** del código. Si querés saber qué hace un service, leer sus tests te muestra: con esta entrada da esta salida, con esta otra entrada tira esta excepción. Es documentación que **no se desincroniza del código** porque si alguien rompe el comportamiento, el test rompe.

### 1.3 Validar

Validar que el código hace lo que querés que haga. *"Las computadoras hacen lo que les decimos, no lo que queremos"* — los tests son el puente entre las dos cosas. Sirven para chequear que tu interpretación del requerimiento coincide con el comportamiento real del código.

### 1.4 Reducción de costos

Esto es contraintuitivo y vale la pena que lo veas claro. **Sin tests** parece más barato (escribís solo el feature y listo). **Con tests** parece más caro (escribís el feature + los tests). En el corto plazo es así. Pero a partir del feature número 5, sin tests ya no sabés cuál de los cinco te rompió la app, tenés que revisar manualmente N archivos, y un bug en producción cuesta órdenes de magnitud más que el bug en desarrollo. Lucas lo dijo así: *"si subís 5 features sin tests y se rompe algo, no sabés cuál fue. Tenés que ir branch por branch."* La regla práctica: **escribir tests es invertir tiempo ahora para no quemar tiempo después**.

### 1.5 Cumplimiento normativo (ISO)

En la industria real, hay normativas que las empresas necesitan cumplir si quieren operar en ciertos rubros, salir a bolsa, o licitar contratos. La que más vas a oír (Lucas la mencionó pero no profundizó) es **ISO 25010**: estándar internacional que define el modelo de calidad de software. Un auditor externo viene, revisa que tu código cumpla ciertos parámetros (entre ellos, porcentaje mínimo de tests), y si todo está, la empresa queda certificada.

> 🟢 ISO se va a profundizar en Ingeniería de Software (cuarto año). Para esta materia alcanza con saber que existe la 25010, que se relaciona con calidad, y que tener tests es uno de los requisitos de varias normas.

### 1.6 Reputación

Si tu producto **no se cae** en producción, los usuarios te ven como confiable. Si se cae todos los martes, te ven como "esa empresa que no funciona". Reputación deriva de calidad: tests sostenidos → menos caídas → mejor reputación. Es indirecta, pero real.

### 1.7 Calidad

El paraguas que abarca todo lo anterior. Tests = calidad de código = calidad de producto = todo lo demás. Por eso al área de gente que se dedica exclusivamente a testear se la suele llamar **QA — Quality Assurance** (aseguramiento de calidad). En empresas grandes es un rol dedicado; en equipos chicos lo absorbe el dev. Su trabajo es básicamente *"romper la aplicación"* antes de que la rompa el usuario.

### Para el parcial, si te preguntan

> **P: Mencioná los beneficios principales del testing.**
>
> **R:** Los siete que fija la cátedra son: **refactor seguro** (red para cambiar código sin miedo), **documentación** (los tests son ejemplos de uso ejecutables), **validación** (chequear que el código hace lo esperado), **reducción de costos** (un bug en dev cuesta mucho menos que en producción), **cumplimiento normativo** (cumplir estándares como ISO 25010), **reputación** (un producto confiable mejora la imagen), y **calidad** (el paraguas que abarca todo lo anterior).

---

## 2. Tipos de tests 🔴

La cátedra distingue **siete tipos** de tests. No son los únicos que existen en la industria, pero son los que vas a manejar en esta materia y los que están en la pirámide oficial. Te los presento en orden de la base hacia la cima (de más granular a más cerca del usuario), que es cómo está armada la pirámide.

### 2.1 Unitarios

Pruebas de **la unidad más pequeña** del código: una función, un método, una clase aislada. Le pasás una entrada controlada y verificás que la salida sea la esperada. Lo crítico es que estás aislando: el unitario no necesita Express levantado, ni base de datos, ni nada externo. Si tu service depende de un repository, en el unitario el repository se *mockea* (lo verás formalmente la próxima clase con persistencia).

**En tu arquitectura por capas:** el caso típico de unitario es testear un método de un service o un método de una entidad de dominio (por ejemplo, `Producto.aplicarDescuento(0.10)` y verificar que el precio quedó en 90).

### 2.2 Funcionales

Validan que el sistema **cumpla los requerimientos funcionales** definidos en las historias de usuario o casos de uso. Se enfocan en el **"qué"** hace el sistema, no en el "cómo" lo hace internamente. Le pasás distintas entradas y verificás que el resultado sea el esperado, sin importar la implementación.

La diferencia con unitarios es de **alcance**: unitario testea una función, funcional testea una funcionalidad completa (ej: "el flujo de aplicar descuento a un producto"). Pueden cruzar varias unidades, pero el foco está en el comportamiento observable, no en la lógica interna.

### 2.3 Integración

Prueban **la interacción entre componentes**. Si los unitarios verifican que cada pieza funciona aislada, los de integración verifican que las piezas hablen bien entre sí.

**En tu arquitectura:** el caso típico es testear que el `ProductoService` se comunique correctamente con el `ProductoRepository`. O ir más arriba: testear todo el camino desde que llega una request al controller, pasa al service, pasa al repository, y vuelve. Lucas lo describió así: *"levantás la app, mandás un GET a un endpoint, y verificás que la respuesta sea la esperada después de pasar por las tres capas."*

> 🟡 Hay subtipos: cuando el alcance es chiquito (dos componentes nada más) algunos lo llaman **test de componentes**; cuando atraviesa todas las capas y simula desde la entrada del request, otros lo llaman **test de sistema**. En clase hubo discusión sobre esos nombres. La cátedra usa **integración** como término paraguas para todos.

### 2.4 E2E (End-to-End)

Validan **el flujo completo** de la aplicación, desde el inicio hasta el fin, simulando el comportamiento de un usuario real. La diferencia clave con integración: integración prueba a nivel API (mandás un request HTTP y validás la respuesta JSON); E2E prueba a nivel UI (la herramienta levanta un navegador, hace clic en botones, llena formularios, valida que aparezcan textos en pantalla).

**Herramientas:** la vieja escuela usa **Selenium**. Hoy lo más común es **Cypress** (que vas a usar en el TP4 según mencionó Lucas) o **Playwright** (más moderno, integra bien con asistentes IA). Con cualquiera de las tres, el flujo es: la herramienta levanta tu app, abre Chrome, hace `click`, escribe en inputs, espera, valida que aparezca lo esperado, y todo eso de forma scriptada.

### 2.5 Regresión

Tests que se ejecutan **después de cualquier cambio** (feature nuevo, fix, refactor) para verificar que **lo que ya funcionaba siga funcionando**. En la práctica son tu batería completa de tests existentes: la corrés, y si todos pasan, no introdujiste regresiones (no rompiste cosas que andaban).

Esto **no es un tipo nuevo de test** que escribís especialmente — son los tests que ya tenés, ejecutados con un propósito específico. Por eso la cátedra los pone como categoría aparte: el rol que cumplen (verificar que nada se rompió) es distinto al rol que cumplen cuando los escribís inicialmente (verificar que el feature anda).

> 🟡 La regresión se automatiza típicamente en el pipeline de CI/CD: cada commit, cada PR, cada build antes de deploy ejecuta toda la batería. Si algo falla, no sube a producción. Esto lo van a ver con más detalle en la clase de infra (penúltima clase de la cursada).

### 2.6 Carga / Stress / Rendimiento

Acá ya no estás probando lógica del código sino **comportamiento bajo presión**. Tres sabores que vale distinguir:

- **Rendimiento (performance):** cuánto tarda en responder tu sistema bajo carga normal. Tiempos de respuesta razonables.
- **Carga:** cuántos usuarios simultáneos aguanta sin degradarse. Mercado Libre en un Cyber Monday: 100 millones de requests concurrentes.
- **Stress:** llevarlo al límite a propósito para ver dónde se rompe. *"Tirar el sistema"* hasta que reviente, para conocer el techo.

**Herramientas:** **JMeter**, **K6**, **Gatling**. **Postman** también permite hacer disparos concurrentes (no solo es un cliente HTTP, también tiene esa capacidad). Lucas mencionó casos reales: en Despegar todos los años antes del Hot Sale (10/05) hacen tests de carga estimando el tráfico esperado a partir del año anterior + algo de margen.

> 🟢 Esto se conecta con escalabilidad horizontal vs vertical (más máquinas vs máquinas más grandes), que la cátedra va a profundizar en la última clase de infraestructura. Hoy las plataformas cloud (AWS, Render, etc.) hacen esto automáticamente según parámetros que definís.

### 2.7 UAT — User Acceptance Testing

La **etapa final** del ciclo de pruebas, antes de salir a producción. La hacen **usuarios finales, clientes o stakeholders** (no devs, no QAs). El objetivo no es encontrar bugs técnicos sino confirmar que **el software efectivamente cumple lo que el negocio necesita**.

> 🟡 **Acá hay un punto importante para tu TP:** Lucas dijo literalmente que **la corrección del TP funciona como un UAT**. La cátedra no va a auditar línea por línea tu código (eso ya lo viste con tests automatizados); van a actuar como usuarios: probar la app, ver si funciona, ver si tiene sentido la justificación de tus decisiones. Esto cambia cómo te conviene preparar la defensa.

### Para el parcial, si te preguntan

> **P: Enumerá los tipos de tests vistos en la cátedra y explicá brevemente cada uno.**
>
> **R:** Son siete: **unitarios** (prueban la unidad más pequeña aislada, sin levantar la app), **funcionales** (validan el "qué" hace el sistema según los requerimientos), **integración** (prueban la interacción entre componentes), **E2E** (simulan al usuario real desde la UI hasta la base, herramientas como Cypress), **regresión** (re-ejecución de la batería existente para verificar que cambios nuevos no rompieron lo que andaba), **carga/stress/rendimiento** (prueban el comportamiento bajo presión, herramientas como JMeter o K6), y **UAT** (validación final del usuario o cliente, valida cumplimiento del negocio).

---

## 3. La pirámide de tests 🔴

Los siete tipos no se aplican todos al mismo nivel. La industria los organiza en una pirámide que muestra **cuánto deberías tener de cada uno** y **qué tan cerca está cada uno del usuario final**.

### El orden oficial de cátedra

De la base a la cima:

```
                    ▲
                  /UAT\
                /      \
              /  Carga  \
            /            \
          /  Regresión    \
        /                  \
      /        E2E          \
    /                        \
  /     Integración           \
/                              \
   Funcionales
─────────────────────────────────
         Unitarios
```

| Posición | Tipo | Cantidad típica | Cerca del... |
|---|---|---|---|
| Base | Unitarios | Muchos (cientos) | Código (lo más granular) |
| | Funcionales | Bastantes | Requerimientos |
| | Integración | Bastantes | Componentes |
| | E2E | Pocos | Flujos completos |
| | Regresión | (es la batería completa) | Todo |
| | Carga / Stress | Pocos | Infra / capacidad |
| Cima | UAT | Muy pocos | Usuario final |

**Lectura del eje vertical:** la base es **lo más granular y rápido** (unitarios corren en milisegundos, no necesitan nada externo). La cima es **lo más cerca del usuario y del sistema en su totalidad** (UAT necesita el sistema entero levantado, ambiente productivo, usuarios reales).

**Lectura del eje del volumen:** la base es ancha porque tenés *muchos* tests unitarios (uno por método, fácil); la cima es angosta porque tenés *pocos* UAT (son caros en tiempo y en gente). Si invirtieras la pirámide (muchos E2E, pocos unitarios), terminás con tests lentos, frágiles y caros de mantener.

> 🔴 **El orden específico de la cátedra puede ser pregunta de parcial.** Otras fuentes ordenan distinto (algunos meten "componentes" entre unitarios e integración; otros suben funcionales). Vos ceñite al orden de cátedra: **Unitarios → Funcionales → Integración → E2E → Regresión → Carga → UAT.**

> 🟢 Existen otras representaciones (el "trofeo de testing" de Sonar es la más popular). Lucas las mencionó pero no las profundizó. Para esta materia: pirámide.

### Para el parcial, si te preguntan

> **P: Dibujá y explicá la pirámide de tests.**
>
> **R:** Es un esquema que ordena los tipos de tests según granularidad y proximidad al usuario. De la base (más granulares, más rápidos, más numerosos) a la cima (más cerca del usuario, más caros, menos numerosos): unitarios, funcionales, integración, E2E, regresión, carga/stress, UAT. La base ancha refleja que conviene tener muchos tests rápidos y aislados; la cima angosta refleja que los tests más globales son escasos y costosos.

---

## 4. Caja blanca vs caja negra 🔴

Otra forma de clasificar tests, **transversal a la pirámide**. No mira *qué nivel* del sistema testeás, sino **qué información usás para escribir el test**.

### 4.1 Definiciones

**Caja blanca (white box):** el test se escribe **conociendo el código por dentro**. Sabés qué condiciones, qué bucles, qué caminos hay, y diseñás el test para recorrerlos. Te importa cómo se transforma la entrada paso a paso, no solo el resultado final.

**Caja negra (black box):** el test se escribe **sin mirar el código**, solo en base a la especificación. Le pasás una entrada, esperás una salida, y no te importa qué pasó adentro. La caja es opaca.

### 4.2 Mapeo en la pirámide (posición de cátedra)

Según la clasificación que dejaron en la pizarra:

| Tipo | Caja |
|---|---|
| Unitarios | **Blanca** (escribís sabiendo qué hace cada función) |
| Integración | **Blanca** (entendés cómo se conectan las capas) |
| Funcionales | **Negra** (te importa que el resultado sea el esperado, no la lógica interna) |
| E2E | **Negra** (simulás al usuario, no te importa el código) |
| Regresión | **Negra** (verificás resultado, no el cómo) |
| Carga / Stress | **Negra** (mirás métricas externas: tiempo, throughput) |
| UAT | **Negra** (el usuario no ve código) |

### 4.3 Aclaración honesta sobre integración

En la clase hubo discusión sobre dónde cae integración. Algunos alumnos votaron negra (te importa que las cosas se integren bien, no necesariamente cómo); la mayoría y los profes se inclinaron por **blanca** (porque al diseñar el test estás pensando en cómo los componentes se hablan entre sí).

Lucas dijo textual: *"seguramente alguien les va a decir que lo que nosotros pusimos como caja blanca no es caja blanca, es caja negra."* Reconoció que es **subjetivo** y que podés argumentar las dos. Otros profes hablan de "caja gris" para integración, justamente por esto.

**¿Qué hacés en parcial?** Si te preguntan dónde cae integración, respondé **caja blanca** (es la posición de cátedra) y, si tenés espacio, podés agregar la frase *"aunque algunos autores la clasifican como gris porque combina elementos de ambos enfoques"*. Eso muestra que entendés la sutileza sin contradecir a la cátedra.

### Para el parcial, si te preguntan

> **P: Diferenciá tests de caja blanca y caja negra. Dame un ejemplo de cada uno.**
>
> **R:** **Caja blanca:** se escriben conociendo la implementación interna. Se diseñan para recorrer condiciones, bucles y caminos específicos del código. Ejemplo: tests **unitarios** sobre un método de un service, donde elijo casos de prueba sabiendo qué `if` y qué `for` tiene la función. **Caja negra:** se escriben en base a la especificación, sin mirar el código. Solo importa que la entrada produzca la salida esperada. Ejemplo: tests **funcionales** o **E2E**, donde valido el comportamiento observable del sistema según los requerimientos, sin importar la implementación.

---

## 5. Cobertura de tests (coverage) 🟡

La **cobertura de código** es una métrica que indica **qué porcentaje del código está siendo ejecutado por los tests**. Sirve para identificar partes del sistema que nadie está probando.

### 5.1 Cómo se mide

Hay herramientas que instrumentan el código y, mientras corrés tu batería de tests, registran qué líneas se ejecutaron y cuáles no. Al final te tiran un reporte: "tu suite cubre 73% del código". **Sonar** es la herramienta más popular en la industria para esto.

### 5.2 Tipos de cobertura

El PDF del profe distingue tres formas de medir cobertura — esto **no se vio en clase**, lo agregó el resumen escrito, así que lo incluyo acá tal cual lo formaliza la cátedra:

- **Line coverage:** porcentaje de **líneas** ejecutadas por los tests.
- **Branch coverage:** porcentaje de **caminos** posibles recorridos (cada `if/else`, cada `try/catch` cuenta como branches separados).
- **Function/method coverage:** porcentaje de **funciones o métodos** que fueron invocados al menos una vez.

Una clase puede tener 100% de line coverage pero 50% de branch coverage si tus tests pasan solo por una rama del `if` y nunca por la otra. Branch coverage es más estricto y más informativo.

### 5.3 Realismo

**Tener 100% de cobertura no es realista ni deseable.** Lucas fue claro: en proyectos grandes **el 80% ya es mucho**. Hay clases que no se testean y eso está bien:

- **Clases que solo tienen atributos** (DTOs, modelos sin comportamiento): testear sus getters no agrega valor.
- **Controllers**, en muchos enfoques, no se testean unitariamente. Lo que se valida del controller (mapeo de body a JSON, status codes) ya queda cubierto por tests de integración.

Y **alta cobertura no garantiza ausencia de bugs**. Podés tener 95% de cobertura y un caso edge no contemplado que rompe en producción. La cobertura te da **visibilidad** sobre qué testeaste, no certeza absoluta.

> 🟡 Cuando llegues al TP2 (full backend + persistencia), la cátedra va a esperar tests sobre **capa de servicios y dominio**. Controllers no son obligatorios y no es gran pérdida si no los testeás unitariamente.

### Para el parcial, si te preguntan

> **P: ¿Qué es la cobertura de tests y para qué sirve?**
>
> **R:** Es la métrica que indica qué porcentaje del código está siendo ejecutado por los tests. Sirve para identificar zonas del sistema sin probar. Se mide en tres formas: line coverage (líneas ejecutadas), branch coverage (caminos recorridos en condicionales) y function coverage (funciones invocadas). Una alta cobertura no garantiza ausencia de bugs, solo da mayor visibilidad sobre qué se está probando. En la práctica, valores cercanos al 80% se consideran muy buenos en proyectos grandes; clases sin lógica (solo atributos) y a veces controllers no se testean unitariamente.

---

## 6. Rol de QA 🟢

Lo nombro al pasar porque la clase lo nombró al pasar. **QA — Quality Assurance** es el rol dedicado a testing en una empresa. Su trabajo no es escribir el código del producto sino **encontrar problemas que el dev no ve**.

Por qué importa que exista como rol separado: el dev está sumergido en cómo funciona el código, asume cosas, no piensa en el caso edge. El QA viene "desde afuera" y rompe la app a propósito. Es complementario, no suplente. En equipos chicos lo absorbe el dev, en equipos grandes es área dedicada (con metodologías propias, herramientas propias, métricas propias).

> 🟢 Esto se va a expandir mucho más en Ingeniería de Software (cuarto año). Para esta materia: saber que el rol existe y que es complementario al dev.

---

## 7. Aclaraciones sobre TP1 (histórico) 🟢

Durante esta clase Lucas y Facu se dieron tiempo para responder dudas operativas sobre el TP1, cuando todavía no se había entregado (la entrega fue el 28/04, semana siguiente). Lo dejo registrado como **referencia histórica** porque parte del criterio de cátedra que se manifestó acá sigue valiendo para entregas futuras:

**Sobre el diagrama de clases del TP:** la cátedra lo dejó **modificable**. *"Es tentativo, no tienen que seguirlo a rajatabla."* Cada grupo decide cómo modelar disponibilidad, turnos, especialidades, prácticas — siempre y cuando puedan **justificar** la decisión. La cátedra no participó en el armado del enunciado, así que ellos mismos están abiertos a ajustes.

**Sobre quién crea los turnos:** dudaron entre médico (basado en disponibilidad) o paciente (creando ad hoc). Lo dejaron a criterio del grupo. La sugerencia más frecuente fue: el médico define rangos de disponibilidad y el sistema **genera los slots** automáticamente para que el paciente reserve.

**Sobre especialidad vs práctica:** pueden tener distinta duración (consulta = 30 min, electrocardiograma = 60 min). Cómo se modela queda libre, mientras encaje coherentemente en la disponibilidad.

**Criterio general que vale para todas las entregas:** *"importa que sepan justificar."* La defensa del TP es una conversación sobre por qué tomaste cada decisión, no un examen sobre si seguiste el diagrama exacto. Esto refuerza el punto de UAT visto en sección 2.7: la cátedra valida que la app cumpla la necesidad del negocio + que el grupo entienda lo que hizo.

---

## Checkpoint

Si pudiste responder estas preguntas sin dudar, esta parte del apunte está digerida. Si te trabás en alguna, volvé a la sección correspondiente. Las respuestas en formato examen están en el complemento (cuando lo generes).

1. ¿Cuáles son los siete beneficios de testing que enumera la cátedra?
2. Explicá la diferencia entre tests unitarios y tests de integración. ¿Qué necesita cada uno para ejecutarse?
3. ¿Cuál es la diferencia entre tests funcionales y tests E2E?
4. Decí qué tipo de test es cada uno: regresión, carga, UAT. Para cada uno, ¿qué pregunta responde?
5. Dibujá la pirámide de tests con el orden específico de la cátedra (de la base a la cima).
6. ¿Por qué la base de la pirámide es ancha y la cima angosta?
7. Definí caja blanca y caja negra. ¿Qué información se usa para diseñar cada tipo?
8. ¿Dónde cae integración según la cátedra: caja blanca o caja negra? ¿Por qué la decisión es discutible?
9. ¿Qué es la cobertura de código? Nombrá los tres tipos formales.
10. ¿Por qué tener 100% de cobertura no es necesariamente bueno?
11. ¿Por qué los controllers no se suelen testear unitariamente?
12. ¿Qué hace el rol de QA y por qué es valioso que sea independiente del dev?
13. ¿Qué es UAT y cómo se relaciona con la corrección del TP en la cátedra?

---

## Qué viene en la Parte 2

La Parte 2 cubre las **dos metodologías de desarrollo** que se enseñaron en clase:

- **TDD (Test-Driven Development):** escribir el test antes que el código. Ciclo **red-green-refactor**. Ejemplo en vivo en Python con la caja de ahorro (constructor → depositar → retirar → excepción `SaldoInsuficiente`).
- **BDD (Behavior-Driven Development):** describir el comportamiento esperado en lenguaje cercano al negocio. Estructura **Given-When-Then**. Ejemplo en vivo con `behave` y archivos `.feature` (Gherkin).
- Cuándo conviene cada una.
- Conexión con tu TP (Jest en JavaScript) — voy a incluir un complemento opcional con el equivalente en Jest del ejemplo de la caja de ahorro, claramente separado del contenido oficial de la clase.

---

**FIN DE LA PARTE 1**
