# Apunte Maestro — Semana 4 Teoría, Parte 2

## Metodologías: TDD y BDD

> **Semana 4 — Martes 21/04 — Profes: Lucas (lidera) y Facu (soporte)**
>
> **Sobre esta parte:** Segunda y última parte del apunte de teoría de la semana 4. Si la Parte 1 fue el "mapa" (qué tipos de tests hay, cómo se ordenan, qué es cobertura), esta parte es **cómo se trabaja** con tests cuando los integrás al proceso de desarrollo. Cubrimos las dos metodologías que la cátedra enseña: **TDD** (Test-Driven Development, ciclo red-green-refactor) y **BDD** (Behavior-Driven Development, estructura Given-When-Then). Cada una con el ejemplo en vivo que mostraron en clase: una **caja de ahorro** implementada paso a paso con `pytest` (TDD) y luego con `behave` (BDD).
>
> **Sobre el lenguaje del ejemplo:** El profe usó **Python** porque es donde mejor se ve la sintaxis de `behave` y porque el ejemplo de caja de ahorro le quedaba fluido. El TP que estás haciendo es en **JavaScript con Jest**. Por eso, después del contenido oficial, agrego un **apéndice opcional** con el mismo ejemplo traducido a Jest, claramente marcado como complemento — no es contenido evaluable de cátedra, es para que tengas el puente listo cuando vayas a escribir los tests del TP2.

---

## 0. Conexión con la Parte 1

En la Parte 1 vimos los **tipos** de tests (qué probar). Esta parte es sobre el **proceso** de escribir tests (cuándo y cómo, en relación al código que estás desarrollando). Son dimensiones distintas del mismo problema: podés escribir tests unitarios de la forma "tradicional" (primero el código, después los tests) o aplicando TDD (primero los tests, después el código). El tipo de test no cambia — cambia el **orden y la mentalidad** con la que lo abordás.

**Importante:** TDD y BDD no son tipos de tests. Son **enfoques de desarrollo guiados por tests**. La pirámide de la Parte 1 sigue valiendo igual; lo que estas dos metodologías te dicen es *cuándo* y *con qué vocabulario* escribir cada test de esa pirámide.

---

## 1. TDD — Test-Driven Development 🔴

### 1.1 Qué es y para qué

**TDD = Desarrollo Guiado por Pruebas.** La idea es simple y un poco contraintuitiva: **escribís el test antes que el código de producción**. Ese orden invertido cambia varias cosas.

**Por qué importa el orden:**

- Cuando escribís el test primero, te obliga a pensar la función como **un cliente que la va a usar**, no como un programador que la implementa. Lo que pensás es: "¿qué entrada le voy a pasar y qué quiero que devuelva?". Ese ejercicio mental tiende a producir interfaces más limpias.
- Cuando escribís el código primero, hay un sesgo natural a testear *lo que escribiste*, no *lo que necesitabas*. Si el código tenía un bug, el test que armás después puede tener el mismo bug embebido y dar verde por las razones equivocadas.
- TDD garantiza que **toda línea de código tiene al menos un test que la justifica**. Si no había test, no escribís el código. Eso da cobertura natural alta.

### 1.2 El ciclo Red-Green-Refactor 🔴

El ciclo es **muy corto e iterativo**: lo aplicás muchas veces por hora cuando estás en flow.

| Fase | Color | Qué hacés | Estado del código |
|---|---|---|---|
| **Red** | 🔴 | Escribís un test que *falla* | El feature todavía no existe |
| **Green** | 🟢 | Escribís el código mínimo para que *pase* | Feature funcional, pero quizás sucio |
| **Refactor** | 🟡 | Limpiás el código sin romper el test | Feature funcional y limpio |

**Las tres reglas estrictas del ciclo:**

1. **Solo escribís código de producción si hay un test fallando que lo necesita.** Nada de "ya que estoy le agrego esto otro".
2. **En Green escribís lo mínimo posible.** Si el test pide que la función devuelva 90, devolvés `return 90` literal aunque sea ridículo. La generalización viene cuando un nuevo test la fuerce. Esto suena absurdo en un ejemplo aislado, pero es el corazón del método: el código va emergiendo guiado por los tests, no por suposiciones.
3. **No saltás Refactor.** Si dejaste duplicación o un nombre malo en Green, lo limpiás antes de pasar al próximo Red. El refactor con tests verdes es seguro: tenés la red.

**Cómo se ve en una herramienta:**

- **Verde** = el test corrió y pasó.
- **Rojo** = el test corrió y falló (la aserción no se cumplió, o el código tiró una excepción inesperada).
- **Amarillo** = el test corrió y la salida fue diferente a la esperada pero sin excepción (típico cuando esperabas 90 y devolvió 80, por ejemplo). En la práctica, tanto rojo como amarillo cuentan como "test fallando".

### 1.3 Ejemplo en vivo: la caja de ahorro 🔴

Lucas armó este ejemplo paso a paso en clase, en Python con `pytest`. Te lo reproduzco entero con código comentado y la lógica del ciclo en cada paso.

**Setup:** vamos a desarrollar una clase `CajaAhorro` con tres operaciones: crear con un saldo inicial, depositar, retirar. La regla de negocio es que **no se puede retirar más que el saldo disponible**.

#### Paso 1 — Crear la caja

**🔴 Red.** Escribimos primero el test, sin tener todavía la clase.

```python
# test_caja_de_ahorro.py
from caja_de_ahorro import CajaAhorro

def test_crear_caja_de_ahorro_con_saldo_inicial():
    caja = CajaAhorro(saldo=100)
    assert caja.saldo == 100
```

Lo corremos: **explota** porque `CajaAhorro` no existe.

**🟢 Green.** Escribimos lo mínimo para que pase.

```python
# caja_de_ahorro.py
class CajaAhorro:
    def __init__(self, saldo=0):
        # saldo=0 como default es un detalle Python: si no le pasás saldo,
        # la caja arranca en cero. Útil para no romper la API más adelante.
        self.saldo = saldo
```

Corremos el test: **verde**. Caja creada con saldo 100, asserción cumplida.

**🟡 Refactor.** Nada que limpiar todavía. Seguimos.

#### Paso 2 — Depositar

**🔴 Red.** Test para depositar.

```python
def test_depositar_incrementa_el_saldo():
    caja = CajaAhorro(saldo=100)
    caja.depositar(50)
    assert caja.saldo == 150
```

Corremos: **explota**. La caja no tiene método `depositar`.

**🟢 Green.** Implementamos el método.

```python
class CajaAhorro:
    def __init__(self, saldo=0):
        self.saldo = saldo

    def depositar(self, monto):
        self.saldo += monto
```

Corremos los dos tests: **ambos verdes**.

**🟡 Refactor.** Nada que limpiar. Seguimos.

#### Paso 3 — Retirar (versión ingenua)

**🔴 Red.** Test para retirar.

```python
def test_retirar_decrementa_el_saldo():
    caja = CajaAhorro(saldo=150)
    caja.retirar(100)
    assert caja.saldo == 50
```

Corremos: **explota**. No existe `retirar`.

**🟢 Green.** Implementación ingenua.

```python
def retirar(self, monto):
    self.saldo -= monto
```

Corremos los tres tests: **todos verdes**.

Acá podríamos darnos por satisfechos — pero **el ejemplo tiene un bug oculto que el test no agarró**: si retiro más que mi saldo, la cuenta queda en negativo.

```python
# Esto NO está testeado y rompe la regla de negocio:
caja = CajaAhorro(saldo=50)
caja.retirar(100)
# caja.saldo == -50  ← acabás de inventar plata para el cliente
```

Esto es **exactamente la situación que TDD ayuda a detectar antes de producción**: estás pensando en los casos antes de escribir el código, y aparece naturalmente la pregunta "¿qué pasa si retira de más?".

#### Paso 4 — La excepción de saldo insuficiente

**🔴 Red.** Test que valida que se lance una excepción cuando se quiere retirar de más.

```python
def test_no_puede_retirar_mas_que_el_saldo_disponible():
    """
    Si se intenta retirar un monto mayor al saldo, la caja
    debe lanzar SaldoInsuficienteException con un mensaje claro.
    """
    caja = CajaAhorro(saldo=50)

    # pytest.raises captura la excepción esperada para que el test
    # pueda inspeccionarla. Si NO se lanza la excepción, el test falla.
    with pytest.raises(SaldoInsuficienteException) as exc_info:
        caja.retirar(100)

    # Validamos también el mensaje: que sea exactamente el que el
    # negocio espera, no uno cualquiera.
    assert str(exc_info.value) == "No puede retirar más que el saldo disponible"
```

Corremos: **rojo** con un mensaje del estilo *"DID NOT RAISE"*. La función `retirar` no está lanzando la excepción esperada (de hecho ni siquiera existe la clase `SaldoInsuficienteException`).

**🟢 Green.** Dos cosas: crear la excepción custom y modificar `retirar` para que la lance.

```python
# caja_de_ahorro.py

class SaldoInsuficienteException(Exception):
    """Excepción lanzada cuando se intenta retirar más que el saldo disponible."""
    def __init__(self, mensaje):
        # super() llama al constructor de Exception, que guarda
        # el mensaje y permite que str(exc) lo devuelva.
        super().__init__(mensaje)


class CajaAhorro:
    def __init__(self, saldo=0):
        self.saldo = saldo

    def depositar(self, monto):
        self.saldo += monto

    def retirar(self, monto):
        # Validamos la regla de negocio ANTES de modificar el saldo.
        # Si fallás esta guarda, ni se toca self.saldo.
        if self.saldo < monto:
            raise SaldoInsuficienteException(
                "No puede retirar más que el saldo disponible"
            )
        self.saldo -= monto
```

Corremos los cuatro tests: **todos verdes**.

**🟡 Refactor.** En proyectos reales, las excepciones custom suelen ir en un módulo aparte (`exceptions.py` o `errors/`) en vez de mezclarse con el dominio. Para este ejemplo se quedan juntas, pero en tu TP probablemente convenga separarlas (te queda más prolijo y se nota cuando hay muchas).

#### Lo que acabás de ver en pequeño

En un caso de juguete, esto parece exagerado: 4 tests para 30 líneas. Pero la **lógica que estás entrenando** es:

1. Cada decisión de código está justificada por un test que la forzó.
2. Cada bug detectado (el "queda en negativo") se transforma primero en un test rojo y después en código que lo arregla.
3. La cobertura emerge naturalmente — no la perseguís a posteriori.

### 1.4 Buenas prácticas mostradas en clase

**Nombres de tests largos y descriptivos.** Prohibido `test_uno`, `test_dos`. Cuando un test falla en CI, querés leer el nombre y entender de inmediato qué se rompió. Ejemplos buenos:

- `test_no_puede_retirar_mas_que_el_saldo_disponible`
- `test_depositar_incrementa_el_saldo`
- `test_crear_caja_con_saldo_negativo_lanza_excepcion`

**Docstrings cuando el nombre no alcanza.** Si el caso es complejo y el nombre se hace gigante, ponés un docstring breve adentro del test:

```python
def test_caso_complejo():
    """
    Cuando un cliente con plan VIP retira en horario nocturno
    en una sucursal de provincia, no se aplica el cargo extra.
    """
    # ...
```

**Estructura del test:** cada test típicamente tiene tres bloques mentales: **setup** (creo el objeto, configuro precondiciones), **action** (ejecuto la operación bajo test), **assertion** (verifico el resultado). En BDD esto se vuelve explícito como *Given-When-Then*; en TDD queda implícito.

### 1.5 ¿Cuándo conviene TDD?

Lucas fue honesto: **no es para todo el mundo ni para todos los casos**. Él mismo lo intentó una vez en su laburo y dijo *"no era para mí"*. Hay equipos que trabajan 100% TDD (mencionó que en TBank, donde da clases una de las profes de los miércoles, lo aplican religiosamente). Hay otros que lo usan selectivamente.

**El "mejor uso" según el PDF de cátedra y lo que enfatizó Lucas en clase:**

- 🔴 **Corrección de bugs.** Acá TDD brilla. Cuando aparece un bug en producción, el primer paso es **reproducirlo** con un test que falla (rojo). Ese test rojo es la prueba de que entendiste el bug. Después arreglás el código hasta que pase (verde). Refactor si hace falta. Y queda como **test de regresión** automático: si alguien vuelve a introducir el mismo bug, el test rompe de inmediato.
- 🟡 **Desarrollo incremental de features nuevas.** Cuando vas a sumar funcionalidad, TDD te fuerza a pensar en los casos antes de escribir.
- 🟢 **No tan recomendado para exploración o prototipos.** Si todavía no tenés claro qué querés construir, escribir tests te frena más de lo que te ayuda.

> **Visión sistémica:** TDD se justifica solo si la lógica es no trivial. Para cosas como un getter o un mapeo dumb de DTO a entidad, hacer TDD es ceremonia sin valor. Conviene cuando hay reglas de negocio reales que querés blindar.

### Para el parcial, si te preguntan

> **P: ¿Qué es TDD y en qué consiste el ciclo Red-Green-Refactor?**
>
> **R:** TDD (Test-Driven Development) es una metodología de desarrollo en la que se escriben primero los tests y después el código de producción. Se basa en un ciclo iterativo de tres fases: **Red** (escribir un test que falle, porque la funcionalidad todavía no existe), **Green** (escribir el código mínimo necesario para que el test pase) y **Refactor** (limpiar y mejorar el código sin romper los tests existentes). Su mejor uso es la **corrección de bugs** —reproducir el bug con un test rojo y arreglar hasta verde— y el desarrollo incremental de funcionalidades. Beneficios principales: mejora el diseño del código, reduce errores y facilita el mantenimiento.

---

## 2. BDD — Behavior-Driven Development 🔴

### 2.1 Qué es y para qué

**BDD = Desarrollo Guiado por el Comportamiento.** Se basa en TDD (no lo reemplaza, lo extiende) pero **cambia el foco**: en lugar de pensar en la unidad de código y su correctitud técnica, pensás en el **comportamiento esperado del sistema** desde el punto de vista del negocio.

**El cambio clave:** los tests se escriben en **lenguaje cercano al negocio**, no en lenguaje técnico. Tan cercano que **una persona no técnica puede leerlos y entenderlos**, e incluso escribirlos. Eso convierte al test en **un contrato entre el equipo técnico y el cliente**.

**Por qué importa eso:**

- Reduce el típico problema de *"creí que el requerimiento era X, resultó que era Y"*. Si el escenario está escrito en lenguaje del negocio y aprobado por el cliente, lo que pase a código sigue ese contrato.
- Sirve como **documentación viviente**: nuevo dev entra al equipo, lee los `.feature` files y entiende qué hace la app sin tocar el código.
- Cuando un escenario falla en CI, el reporte está escrito en lenguaje natural ("Dado que tenía 100 pesos, cuando deposité 50, entonces debería tener 150 — pero tenía 145"). Cualquier persona lo entiende.

### 2.2 La estructura Given-When-Then 🔴

Cada escenario BDD tiene tres bloques obligatorios. Son los equivalentes "narrativos" del setup/action/assertion de TDD, pero formalizados:

| Bloque | Pregunta que responde | Qué contiene |
|---|---|---|
| **Given** | ¿Cuál es el contexto inicial? | Estado del sistema, datos precargados, precondiciones |
| **When** | ¿Qué acción ocurre? | El evento o la operación que disparás |
| **Then** | ¿Qué resultado se espera? | Las aserciones — qué tiene que ser verdadero después |

**Ejemplo mínimo (el que estaba en la pizarra):**

```gherkin
Scenario: Aplicar descuento a un producto
  Given un producto que vale 100 pesos
  When le aplico un descuento del 10%
  Then el precio queda en 90 pesos
```

Notá lo que **no aparece** en este escenario: clases, métodos, parámetros, tipos. No dice "instanciar la clase Producto con el constructor que recibe un Number". Dice "un producto que vale 100 pesos". Esa es la idea: el escenario describe **el qué**, no el cómo.

> 🟡 **Vocabulario:** la palabra **"feature"** en BDD viene de "funcionalidad de negocio". Un archivo `.feature` agrupa varios `Scenario` relacionados con la misma funcionalidad. La sintaxis se llama **Gherkin** y es **idéntica para todos los lenguajes** (Python, Java, .NET, JavaScript) — lo que cambia es la herramienta que la ejecuta.

### 2.3 Ejemplo en vivo: behave con la caja de ahorro 🔴

`behave` es la herramienta de BDD para Python. Hay equivalentes en todos los lenguajes (Cucumber para Java, SpecFlow para .NET, cucumber-js para Node). Lo que ves acá es 100% transferible — solo cambia la sintaxis del lenguaje host.

**Estructura de carpetas que esperaba `behave`:**

```
proyecto/
├── caja_de_ahorro.py              ← clase de dominio (la del ejemplo TDD)
└── features/                       ← carpeta especial, behave la busca por nombre
    ├── caja_de_ahorro.feature     ← los escenarios en Gherkin
    └── steps/                      ← carpeta especial dentro de features
        └── caja_de_ahorro_steps.py ← código Python que mapea cada paso
```

**Instalación:** `pip install behave`. Lucas mencionó que existe la dependencia equivalente para Java (Maven), Node (npm), .NET (NuGet), etc.

#### El archivo `.feature` (lenguaje natural en Gherkin)

```gherkin
Feature: Operar sobre caja de ahorro

  Scenario: Depositar en caja de ahorro
    Given Tengo una caja de ahorro con 100 pesos
    When Le deposito 50 pesos
    Then La caja de ahorro tiene 150 pesos
```

**Esto es lo que verían los analistas funcionales y el cliente.** No hay código Python acá. Si el cliente lee esto y dice "sí, eso es lo que necesito", tenés un acuerdo cerrado.

#### El archivo de steps (mapeo a código Python)

Cada línea Given/When/Then del `.feature` necesita una **implementación** en código que diga *qué hacer cuando aparece esa frase*. Eso vive en `steps/`:

```python
# features/steps/caja_de_ahorro_steps.py
from behave import given, when, then
from caja_de_ahorro import CajaAhorro


@given('Tengo una caja de ahorro con {saldo:d} pesos')
def step_crear_caja(context, saldo):
    """
    Las llaves {saldo:d} son una variable que behave extrae de la frase.
    Si en el .feature dice "100 pesos", aquí saldo = 100.
    El sufijo :d indica que es entero (decimal), behave lo castea solo.
    """
    context.caja = CajaAhorro(saldo)


@when('Le deposito {monto:d} pesos')
def step_depositar(context, monto):
    """
    context es un objeto compartido entre todos los steps del mismo
    escenario. Lo que guardás en él en el Given lo recuperás en el When.
    """
    context.caja.depositar(monto)


@then('La caja de ahorro tiene {saldo_final:d} pesos')
def step_verificar_saldo(context, saldo_final):
    """
    El Then es donde van las aserciones. Se sigue usando assert.
    """
    assert context.caja.saldo == saldo_final
```

**Cómo se conecta todo:**

1. behave lee `caja_de_ahorro.feature`.
2. Para cada línea Given/When/Then, busca un step que matchee la frase.
3. El **decorador** `@given('...')` es lo que ata la frase del feature al código que la ejecuta.
4. El parámetro `context` se pasa automáticamente a todos los steps del mismo escenario, y lo usás como bolsa para guardar estado entre pasos (`context.caja = ...` en el Given se recupera en el When).

#### Cómo se ejecuta y qué imprime

Corrés `behave` desde la raíz del proyecto y obtenés un output como este:

```
Feature: Operar sobre caja de ahorro

  Scenario: Depositar en caja de ahorro
    Given Tengo una caja de ahorro con 100 pesos    ✓
    When Le deposito 50 pesos                        ✓
    Then La caja de ahorro tiene 150 pesos           ✓

1 feature passed, 0 failed, 0 skipped
1 scenario passed, 0 failed, 0 skipped
3 steps passed, 0 failed, 0 skipped
```

Si el saldo final no fuera 150 (por un bug en `depositar`), el output mostraría el step que falló en rojo, con el mensaje del assert. Eso es lo que cualquiera del equipo puede leer y entender — incluso quien no sabe Python.

> 🟡 **Cómo se compara con TDD:** los steps que escribiste son, por debajo, código de test normal. El `assert` final es el mismo `assert` de pytest. La diferencia es la **capa de lenguaje natural** que hay encima, que comunica el comportamiento esperado a no-técnicos.

### 2.4 ¿Cuándo conviene BDD?

- 🔴 **Construcción de funcionalidades nuevas.** Cuando empezás un feature de cero, especialmente si tiene reglas de negocio complejas, el escenario BDD te ayuda a fijar la conversación con el cliente antes de codear.
- 🟡 **Tests de integración o end-to-end con escenarios complejos.** Acá BDD muestra su valor: un E2E de "el paciente busca turno, lo reserva, recibe la notificación" se lee mucho mejor en Gherkin que en código Jest puro.
- 🟢 **Menos útil para tests muy técnicos.** Si vas a testear un cálculo matemático específico de una utility, escribir el escenario en lenguaje natural es overhead. Mantenelo en TDD/pytest común.

**Criterio profesional que dejó Lucas:**

> *"El criterio profesional es saber responder: ¿cuánto suma este test con respecto a lo enquilombado que es escribirlo y mantenerlo?"*

Si el escenario BDD es tu salvoconducto para que negocio te apruebe el feature antes de codear, vale el costo. Si lo estás haciendo por gusto en un cálculo interno, es ceremonia inútil.

### 2.5 La conexión con TDD

BDD **no reemplaza** a TDD. La forma natural de combinarlos en proyectos reales es:

- **BDD para los flujos de alto nivel** (escenarios de usuario, integración entre componentes, casos de uso completos). Ahí es donde el lenguaje natural y el contrato con negocio aportan.
- **TDD para los detalles internos** (cálculos, validaciones, edge cases técnicos). Ahí la velocidad del ciclo red-green-refactor en código directo es lo que importa.

Un escenario BDD puede internamente disparar varios ciclos TDD: cada step del Gherkin podés haberlo desarrollado red-green-refactor por dentro. No son metodologías rivales, son **niveles distintos de zoom**.

### Para el parcial, si te preguntan

> **P: ¿Qué es BDD y cuál es la estructura de un escenario? Diferenciá BDD de TDD.**
>
> **R:** BDD (Behavior-Driven Development) es una metodología que extiende TDD enfocándose en el **comportamiento esperado del sistema desde el punto de vista del negocio**. Los escenarios se escriben en lenguaje cercano al negocio usando la estructura **Given-When-Then**: **Given** describe el contexto inicial o estado del sistema, **When** describe la acción o evento, y **Then** describe el resultado esperado. **Diferencias con TDD:** TDD se enfoca en la calidad del código y la corrección técnica (es práctica de devs); BDD se enfoca en el cumplimiento de los requisitos de negocio y mejora la comunicación con stakeholders no técnicos. TDD usa código de test directo; BDD usa una capa de lenguaje natural (Gherkin) sobre el código. Mejor uso: TDD para corrección de bugs y desarrollo incremental; BDD para construcción de funcionalidades nuevas con escenarios de negocio.

---

## 3. TDD vs BDD — cuándo cuál 🟡

Una tabla de bolsillo para tener clara la decisión:

| Eje | TDD | BDD |
|---|---|---|
| Foco | Código correcto | Comportamiento del negocio |
| Lenguaje | Técnico (código) | Natural (Gherkin) |
| Audiencia | Devs | Devs + negocio + QA |
| Nivel típico | Unitarios | Integración / E2E / aceptación |
| Mejor uso | Bugs + desarrollo incremental | Features nuevos con reglas de negocio |
| Herramienta en clase | `pytest` | `behave` |
| Equivalente JS | Jest | cucumber-js |

**No son excluyentes.** Lo más común en proyectos reales es **usar las dos en distintos niveles** del mismo proyecto. Un E2E de "reserva de turno" se escribe en BDD; los tests unitarios del cálculo de cobertura del plan, en TDD.

---

## 4. Cierre y conexión con el TP

**El profe lo dijo explícito al final de clase:** para el TP van a usar **JavaScript con Jest**. El ejemplo en vivo fue en Python por preferencia pedagógica del momento, pero la herramienta que vas a usar concretamente para los tests del TP1 (mínimo 2 unitarios) y del TP2 (más fuerte de tests, sobre capa servicios y dominio) es **Jest**.

**Lo que se traduce directo de Python a Jest:**

- El ciclo red-green-refactor es idéntico — solo cambia la sintaxis de los asserts (`assert x == 90` en pytest pasa a `expect(x).toBe(90)` en Jest).
- Las excepciones custom se manejan parecido (`pytest.raises` ⇄ `expect(...).toThrow(...)`).
- La estructura `describe`/`it` de Jest cumple un rol similar a los nombres descriptivos que mostró Lucas en pytest, pero con anidamiento explícito.

**Lo que NO se traduce directo:**

- BDD con `behave` no tiene equivalente exacto en el ecosistema Jest puro. Existe **cucumber-js** (la herramienta hermana de behave para Node), pero la cátedra **no la pidió ni la mencionó como obligatoria**. Para el TP, BDD con Gherkin **no se aplica** — solo entran tests unitarios escritos con Jest.

**Próxima clase teórica:** persistencia con MongoDB y ODM. Ahí también se mete el tema de **mocking** (que en esta clase quedó pendiente), porque para testear repositories sin tocar la base real, mockear es la herramienta. Cuando llegue ese contenido, conectalo con lo de esta clase: lo que estás mockeando es la dependencia externa del *unitario* de service.

---

## Apéndice opcional: el ejemplo en Jest 🟢

> **Sobre este apéndice:** Esto **no se enseñó en clase** y **no es contenido evaluable**. Es un complemento que puse acá porque vas a necesitar escribir tests en Jest para tu TP, y tener el mismo ejemplo de la caja de ahorro traducido te sirve de referencia mecánica. Si vas al parcial, lo que cuenta es el ejemplo en Python que se vio en clase. Acá va el equivalente JS por si querés escribirlo en tu repo y verlo correr.

### El código de dominio

```javascript
// caja-de-ahorro.js

// Excepción custom — en JS heredamos de Error.
class SaldoInsuficienteError extends Error {
  constructor(mensaje) {
    super(mensaje);
    // Sin esta línea, error.name sería 'Error' siempre.
    // Así queda 'SaldoInsuficienteError' que es lo útil para debugear.
    this.name = 'SaldoInsuficienteError';
  }
}

class CajaAhorro {
  constructor(saldo = 0) {
    this.saldo = saldo;
  }

  depositar(monto) {
    this.saldo += monto;
  }

  retirar(monto) {
    // Validamos la regla de negocio antes de tocar el saldo.
    if (this.saldo < monto) {
      throw new SaldoInsuficienteError(
        'No puede retirar más que el saldo disponible'
      );
    }
    this.saldo -= monto;
  }
}

module.exports = { CajaAhorro, SaldoInsuficienteError };
```

### Los tests con Jest

```javascript
// caja-de-ahorro.test.js
const { CajaAhorro, SaldoInsuficienteError } = require('./caja-de-ahorro');

describe('CajaAhorro', () => {
  // describe agrupa tests relacionados; aporta a la legibilidad
  // y al output cuando algo falla.

  it('se crea con un saldo inicial', () => {
    const caja = new CajaAhorro(100);
    expect(caja.saldo).toBe(100);
  });

  it('al depositar incrementa el saldo', () => {
    const caja = new CajaAhorro(100);
    caja.depositar(50);
    expect(caja.saldo).toBe(150);
  });

  it('al retirar decrementa el saldo', () => {
    const caja = new CajaAhorro(150);
    caja.retirar(100);
    expect(caja.saldo).toBe(50);
  });

  it('no permite retirar más que el saldo disponible', () => {
    const caja = new CajaAhorro(50);

    // En Jest, para validar que algo lanza una excepción,
    // envolvés la operación en una arrow function y se la pasás
    // a expect. Si no la envolvés, la excepción se tira "afuera"
    // de Jest y el test rompe en vez de aprobar.
    expect(() => caja.retirar(100)).toThrow(SaldoInsuficienteError);

    // Y podés validar también el mensaje exacto.
    expect(() => caja.retirar(100)).toThrow(
      'No puede retirar más que el saldo disponible'
    );
  });
});
```

### Tabla de equivalencias rápida

| Concepto | Python (pytest) | JavaScript (Jest) |
|---|---|---|
| Definir test | `def test_xxx():` | `it('xxx', () => {})` |
| Agrupar tests | (nombrado por convención) | `describe('Grupo', () => {})` |
| Aserción de igualdad | `assert x == 90` | `expect(x).toBe(90)` |
| Aserción de excepción | `with pytest.raises(Err): ...` | `expect(() => ...).toThrow(Err)` |
| Validar mensaje de error | `assert str(exc.value) == "..."` | `expect(() => ...).toThrow("...")` |
| Excepción custom | `class MiError(Exception):` | `class MiError extends Error {}` |
| Correr la suite | `pytest` | `npm test` o `npx jest` |

### Estructura sugerida para tu TP

```
tu-proyecto/
├── server/
│   ├── domain/
│   │   ├── Producto.js
│   │   └── Producto.test.js          ← tests unitarios al lado del archivo
│   ├── services/
│   │   ├── ProductoService.js
│   │   └── ProductoService.test.js
│   └── ...
├── jest.config.js                     ← config de Jest (opcional)
└── package.json                       ← script "test": "jest"
```

Convención común: el archivo de test va al lado del archivo testeado, con sufijo `.test.js`. Jest los detecta automáticamente.

---

## Checkpoint

Si pudiste responder estas sin dudar, la Parte 2 está digerida. Si no, volvé a la sección.

1. ¿Qué significa que TDD invierta el orden tradicional? ¿Cuál es la primera línea que escribís cuando aplicás TDD?
2. Nombrá las tres fases del ciclo TDD y explicá el estado del código al final de cada una.
3. ¿Por qué en Green se escribe lo mínimo posible para hacer pasar el test, aunque parezca absurdo?
4. ¿Qué diferencia hay entre rojo y amarillo en el output de un runner de tests?
5. Mencioná dos buenas prácticas para nombrar tests que mostró Lucas.
6. Según la cátedra, ¿cuál es el "mejor uso" de TDD? Explicá por qué.
7. ¿Qué significa BDD y en qué se basa?
8. Explicá cada uno de los tres bloques de un escenario BDD (Given, When, Then). ¿Qué pregunta responde cada uno?
9. ¿Qué es Gherkin? ¿Es exclusivo de un lenguaje?
10. En el ejemplo de `behave`, ¿qué función cumple el objeto `context`? ¿Por qué se necesita?
11. ¿Cómo conecta una línea del archivo `.feature` con el código Python que la ejecuta?
12. ¿Por qué BDD es valioso para la comunicación con stakeholders no técnicos?
13. ¿Cuándo conviene TDD y cuándo conviene BDD? ¿Son excluyentes?
14. Para tu TP en JavaScript con Jest, ¿qué partes de lo visto en clase aplican directo y cuáles no?

---

## Cierre de la semana

Esta clase fue el "mapa" formal de testing. La próxima clase teórica (martes 05/05, semana 6) ya pasó el tema a **persistencia con MongoDB y ODM**, donde se mete también el **mocking** que quedó diferido. Cuando llegues a esa, vas a poder cerrar el círculo: con mocks aprendés a testear el repository sin levantar la base, y eso te permite escribir tests unitarios de calidad para todo tu TP2.

Para la **Entrega 2 del TP** (martes 26/05), apuntá a tener tests unitarios sobre tu capa de **services** y **dominio** (los controllers son opcionales). Aplicar TDD para los casos de regla de negocio del dominio (los equivalentes al "no se puede retirar más que el saldo") es exactamente el escenario donde más rinde.

---

**FIN DE LA PARTE 2 — FIN DEL APUNTE MAESTRO DE LA SEMANA 4 TEORÍA**
