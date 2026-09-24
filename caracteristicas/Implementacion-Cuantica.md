# Implementación Cuántica de Quantum Coffee

---

## 1. Ficha Técnica

| Campo | Detalle |
|---|---|
| **Característica** | Extensión cuántica de Quantum Coffee |
| **Propósito** | Permitir que un programa escrito en Quantum Coffee declare qubits, aplique compuertas cuánticas y convierta resultados cuánticos en valores clásicos mediante medición |
| **Primitivas de la V1** | `qubit`, `H`, `X`, `Z`, `CNOT`, `medir` |
| **Representación interna** | Vector de estado global de amplitudes complejas |
| **Modelo de ejecución** | Simulación clásica de un circuito cuántico ideal |
| **Máximo inicial** | 16 qubits |
| **Orden interno de qubits** | El primer qubit asignado corresponde al bit menos significativo (LSB) |
| **Resultado de medición** | `0 -> ralo`, `1 -> fuerte`, ambos representados mediante el tipo clásico `cafe` |
| **Frameworks externos** | No se requiere Qiskit, Cirq, Guppy ni otro framework cuántico |
| **Integración esperada** | Lexer, parser, AST, análisis semántico y runtime de Quantum Coffee |
| **Módulo principal** | `QuantumSimulator` |

---

## 2. Sinopsis

La parte cuántica de Quantum Coffee se plantea como una extensión pequeña y controlada sobre un lenguaje que continúa siendo fundamentalmente imperativo y clásico. El objetivo no es convertir Quantum Coffee en una plataforma completa de programación cuántica ni reproducir la arquitectura de herramientas industriales como Qiskit. La intención es introducir suficientes conceptos cuánticos para que el lenguaje pueda representar y ejecutar pequeños circuitos, al mismo tiempo que la implementación siga siendo razonable dentro de un proyecto universitario de Compiladores.

Desde el punto de vista del programador, la extensión aparece mediante nuevas construcciones sintácticas. Un programa puede declarar qubits, aplicar compuertas y medir sus resultados:

```text
barista chorreador() {
    qubit q0;
    qubit q1;

    H(q0);
    CNOT(q0, q1);

    cafe resultado0 = medir(q0);
    cafe resultado1 = medir(q1);

    servir(resultado0);
    servir(resultado1);
}
```

Estas instrucciones parecen similares a llamadas ordinarias, pero internamente no representan operaciones clásicas. `H(q0)` no modifica una variable de la misma forma que `x = x + 1`, porque un qubit no contiene directamente un cero o un uno. El qubit forma parte de un estado cuántico conjunto descrito mediante amplitudes, y las compuertas modifican matemáticamente dichas amplitudes.

Por esta razón, la implementación debe mantener una separación clara entre el **compilador** y el **simulador cuántico**. El compilador sigue siendo responsable de reconocer tokens, validar la gramática, construir el AST, manejar ámbitos y detectar errores semánticos. El simulador, en cambio, recibe operaciones ya validadas y se encarga exclusivamente de la matemática cuántica.

La separación conceptual puede resumirse así:

```text
Código Quantum Coffee
        |
        v
      Lexer
        |
        v
      Parser
        |
        v
       AST
        |
        v
Análisis semántico
        |
        v
 Runtime / Intérprete
        |
        +--------------------+
        |                    |
        v                    v
 ejecución clásica     QuantumSimulator
                            |
                            v
                     vector de estado
```

El simulador de Quantum Coffee funciona sobre una computadora clásica. No existe hardware cuántico detrás de la ejecución. El comportamiento cuántico se reproduce manteniendo explícitamente el vector de estado del sistema y aplicando transformaciones equivalentes a las matrices de las compuertas.

Esta decisión tiene una consecuencia fundamental: para un sistema de $n$ qubits, el simulador debe almacenar $2^n$ amplitudes. El crecimiento exponencial es precisamente una de las razones por las que la simulación clásica de sistemas cuánticos grandes resulta costosa. Quantum Coffee acepta esta limitación porque su objetivo es trabajar con circuitos pequeños y pedagógicos. Para la V1 se establece un máximo de 16 qubits.

---

## 3. Desglose por Subtemas

### 3.1 La parte cuántica dentro del compilador

La implementación comienza mucho antes de llegar a las ecuaciones del simulador. Para que una instrucción como:

```text
H(q);
```

pueda ejecutarse correctamente, debe atravesar las mismas etapas que cualquier otra construcción del lenguaje.

El lexer debe reconocer las palabras reservadas cuánticas:

```text
qubit
H
X
Z
CNOT
medir
```

Estas palabras deben producir tokens propios, por ejemplo:

```text
TK_QUBIT
TK_H
TK_X
TK_Z
TK_CNOT
TK_MEDIR
```

La razón para reservarlas es evitar que el parser confunda una compuerta con una llamada ordinaria de función. Sin esta distinción, el texto `H(q);` podría interpretarse simplemente como una llamada a una función llamada `H`. Reservar los términos hace explícito que se trata de primitivas del lenguaje y simplifica la construcción de nodos cuánticos en el AST.

Una vez tokenizada la entrada, el parser utiliza las reglas definidas en la gramática EBNF:

```text
declaracion_qubit      ::= "qubit" identificador ";"

operacion_cuantica     ::= compuerta_unaria
                         | compuerta_binaria

compuerta_unaria       ::= ( "H" | "X" | "Z" )
                           "(" identificador ")"

compuerta_binaria      ::= "CNOT"
                           "(" identificador "," identificador ")"

medicion_cuantica      ::= "medir"
                           "(" identificador ")"
```

El parser solamente determina que la forma sintáctica sea válida. En este momento todavía no necesita saber si el identificador representa realmente un qubit.

Por ejemplo:

```text
grano x = 10;
H(x);
```

puede coincidir con la forma de una compuerta unaria, porque después de `H(` aparece un identificador y luego `)`. El error real es semántico: `x` existe, pero es una variable clásica de tipo `grano`.

Esta separación sigue el principio normal de un compilador: la gramática reconoce la estructura; el análisis semántico decide si esa estructura tiene sentido.

---

### 3.2 Representación del qubit en la tabla de símbolos

Un qubit no debe almacenarse en la tabla de símbolos como si fuera simplemente otro tipo clásico.

Quantum Coffee maneja tipos como:

```text
grano
poso
granoMolido
cafe
taza tipo
```

Estos representan valores clásicos que pueden almacenarse, copiarse, compararse y utilizarse dentro de las expresiones normales del lenguaje.

Un qubit tiene restricciones distintas. Por eso, en la V1 se maneja como una categoría especial de símbolo.

Conceptualmente, una entrada puede verse así:

```text
nombre: q0
categoria: QUBIT
```

mientras una variable ordinaria puede representarse como:

```text
nombre: contador
categoria: VARIABLE
tipo: grano
```

La separación permite que el análisis semántico detecte construcciones inválidas como:

```text
qubit q;
grano x = q;
```

El problema no es que `q` tenga un tipo numérico incorrecto. El problema es más fundamental: un qubit no puede convertirse directamente en un valor clásico. La única operación que cruza esa frontera es la medición:

```text
cafe resultado = medir(q);
```

Por tanto, la tabla de símbolos cumple una función importante en la frontera entre ambas partes del lenguaje. El compilador puede saber que `q` existe y que pertenece a la categoría `QUBIT` sin conocer su amplitud, su probabilidad ni su estado físico. Toda esa información pertenece al simulador.

---

### 3.3 Representación matemática de un qubit

Un qubit no se representa internamente mediante un único bit.

Un bit clásico solo puede encontrarse en uno de dos estados:

$$
0
\qquad\text{o}\qquad
1
$$

En cambio, un qubit puede encontrarse en una combinación lineal de los dos estados de la base computacional:

$$
|\psi\rangle = \alpha|0\rangle + \beta|1\rangle
$$

donde $\alpha$ y $\beta$ son amplitudes complejas.

En forma vectorial:

$$
|\psi\rangle =
\begin{pmatrix}
\alpha \\
\beta
\end{pmatrix}
$$

La diferencia fundamental es que $\alpha$ y $\beta$ no son directamente probabilidades. Las probabilidades se obtienen mediante la Regla de Born:

$$
P(0)=|\alpha|^2
$$

$$
P(1)=|\beta|^2
$$

y todo estado válido debe satisfacer:

$$
|\alpha|^2+|\beta|^2=1
$$

Esta ecuación es la condición de normalización.

Un qubit recién creado comienza en el estado:

$$
|0\rangle
$$

que vectorialmente corresponde a:

$$
|0\rangle =
\begin{pmatrix}
1 \\
0
\end{pmatrix}
$$

Por tanto:

$$
P(0)=|1|^2=1
$$

$$
P(1)=|0|^2=0
$$

Internamente, el simulador comienza representándolo como:

```text
[1, 0]
```

Esto no significa que el simulador almacene simplemente “el qubit vale cero”. El arreglo contiene amplitudes. Esa diferencia se vuelve visible en cuanto se aplica una compuerta como Hadamard, porque el estado deja de corresponder a una sola posición con amplitud 1.

---

### 3.4 Por qué todos los qubits comparten un mismo vector

Una implementación ingenua podría intentar guardar cada qubit de manera independiente:

```text
q0 -> [alpha0, beta0]
q1 -> [alpha1, beta1]
```

Eso funciona mientras los qubits permanezcan separables, pero falla en cuanto aparece entrelazamiento.

Para dos qubits, el estado general no tiene dos amplitudes sino cuatro:

$$
|\psi\rangle =
\alpha_{00}|00\rangle
+\alpha_{01}|01\rangle
+\alpha_{10}|10\rangle
+\alpha_{11}|11\rangle
$$

Por tanto, el simulador debe almacenar:

```text
[
    alpha00,
    alpha01,
    alpha10,
    alpha11
]
```

Para tres qubits aparecen ocho estados:

$$
|000\rangle,
|001\rangle,
|010\rangle,
|011\rangle,
|100\rangle,
|101\rangle,
|110\rangle,
|111\rangle
$$

y el estado general tiene la forma:

$$
|\psi\rangle =
\sum_{i=0}^{7}\alpha_i|i\rangle
$$

En general, para $n$ qubits:

$$
|\psi\rangle =
\sum_{i=0}^{2^n-1}\alpha_i|i\rangle
$$

con la condición:

$$
\sum_{i=0}^{2^n-1}|\alpha_i|^2=1
$$

El vector posee entonces exactamente:

$$
2^n
$$

amplitudes.

La necesidad de un vector global no es una decisión arbitraria. Es la forma mínima de representar estados entrelazados. Por ejemplo, el estado de Bell:

$$
|\Phi^+\rangle =
\frac{|00\rangle+|11\rangle}{\sqrt{2}}
$$

se almacena como:

```text
[
    1/sqrt(2),
    0,
    0,
    1/sqrt(2)
]
```

No existen dos vectores individuales de un qubit cuyo producto tensorial produzca este estado. Esa imposibilidad de separar el sistema en estados individuales es precisamente la propiedad que obliga a mantener un único vector conjunto.

---

### 3.5 Crecimiento exponencial y límite de 16 qubits

El costo del vector global crece exponencialmente.

Para $n$ qubits:

$$
N_{amplitudes}=2^n
$$

Algunos tamaños son:

| Qubits | Amplitudes |
|---:|---:|
| 1 | 2 |
| 2 | 4 |
| 4 | 16 |
| 8 | 256 |
| 10 | 1024 |
| 16 | 65536 |

Si cada amplitud compleja se almacena mediante dos números `double` de 64 bits —parte real e imaginaria— cada amplitud consume aproximadamente 16 bytes.

Para 16 qubits:

$$
65536\times16
=
1048576\text{ bytes}
\approx 1\text{ MiB}
$$

El problema aparece cuando se continúa aumentando $n$. Con 30 qubits:

$$
2^{30}=1073741824
$$

amplitudes, lo que requeriría del orden de 16 GiB solo para el vector, sin contar estructuras auxiliares.

Por eso el límite de 16 qubits es una decisión de ingeniería razonable para Quantum Coffee V1. No representa un límite de la computación cuántica, sino del simulador clásico deliberadamente pequeño que acompaña al lenguaje.

---

### 3.6 Asignación incremental de qubits

El simulador comienza conceptualmente con un sistema de cero qubits:

```text
stateVector = [1]
```

Este vector puede parecer extraño, pero representa el estado vacío con amplitud unitaria. Su utilidad aparece cuando se agrega el primer qubit.

Al ejecutar:

```text
allocateQubit("q0")
```

el nuevo qubit debe comenzar en $|0\rangle$. El estado pasa a:

```text
[1, 0]
```

Después:

```text
allocateQubit("q1")
```

produce:

```text
[1, 0, 0, 0]
```

y un tercer qubit produce:

```text
[1, 0, 0, 0, 0, 0, 0, 0]
```

Cada asignación duplica el tamaño del vector.

La interpretación matemática es un producto tensorial con $|0\rangle$. Si el estado anterior es $|\psi\rangle$, al agregar un nuevo qubit inicialmente en cero se obtiene:

$$
|\psi'\rangle = |0\rangle\otimes|\psi\rangle
$$

bajo la convención de índices escogida por el simulador.

Quantum Coffee utiliza asignación incremental porque mantiene al módulo independiente del compilador. El simulador no necesita recibir previamente el número total de qubits ni depende de un pase adicional que cuente todas las declaraciones.

`allocateQubit(id)` debe comprobar además dos errores propios del módulo: que el identificador no haya sido asignado anteriormente y que el número total de qubits no supere 16. El análisis semántico debería detectar esas situaciones antes de la ejecución, pero el simulador vuelve a verificarlas como defensa en profundidad.

---

### 3.7 Convención LSB y mapeo de identificadores

El simulador mantiene un mapa:

```text
id -> bitIndex
```

La convención de Quantum Coffee es:

> El primer qubit asignado corresponde al bit menos significativo del índice del vector de estado.

Por tanto:

```text
q0 -> bit 0
q1 -> bit 1
q2 -> bit 2
```

Para dos qubits, los índices se interpretan así:

| Índice | Binario | Estado |
|---:|:---:|---|
| 0 | `00` | $|q_1=0,q_0=0\rangle$ |
| 1 | `01` | $|q_1=0,q_0=1\rangle$ |
| 2 | `10` | $|q_1=1,q_0=0\rangle$ |
| 3 | `11` | $|q_1=1,q_0=1\rangle$ |

Esta convención afecta directamente a la implementación de las compuertas y a la lectura de `stateVector()`.

Partiendo de:

```text
[1, 0, 0, 0]
```

la operación:

```text
X(q0)
```

debe producir:

```text
[0, 1, 0, 0]
```

porque el estado es ahora $|01\rangle$.

En cambio:

```text
X(q1)
```

debe producir:

```text
[0, 0, 1, 0]
```

correspondiente a $|10\rangle$.

Fijar esta convención desde el inicio evita uno de los errores más frecuentes al programar simuladores: implementar correctamente las operaciones pero interpretar los índices con un orden de bits distinto en cada parte del código.

---

### 3.8 Máscaras de bits como mecanismo de aplicación de compuertas

Una aproximación matemática directa consistiría en construir una matriz de dimensión $2^n\times2^n$ para cada compuerta y multiplicarla por todo el vector de estado. Aunque esto sería correcto, resulta innecesariamente costoso y complica la implementación.

Quantum Coffee puede aplicar las compuertas directamente sobre las amplitudes afectadas.

Para un qubit cuyo índice interno es $b$, se construye:

```text
mask = 1 << b
```

lo que matemáticamente equivale a:

$$
\text{mask}=2^b
$$

La máscara permite saber si un índice representa un estado donde ese qubit vale 0 o 1.

Si:

```text
(index & mask) == 0
```

el qubit se encuentra en 0 para ese estado base.

Si:

```text
(index & mask) != 0
```

el qubit se encuentra en 1.

Además, para un índice `i` en el que el bit objetivo vale 0, la posición equivalente con ese bit en 1 es:

```text
j = i | mask
```

De esta forma, las compuertas de un solo qubit trabajan sobre pares de amplitudes:

$$
(\alpha_i,\alpha_j)
$$

donde los dos estados representados por $i$ y $j$ son idénticos excepto en el qubit objetivo.

Este mecanismo permite implementar $X$, $H$ y $Z$ sin crear matrices gigantes.

---

### 3.9 Compuerta Pauli-X

La compuerta Pauli-X es el equivalente cuántico de una negación sobre la base computacional.

Su matriz es:

$$
X=
\begin{pmatrix}
0 & 1\\
1 & 0
\end{pmatrix}
$$

y su acción es:

$$
X|0\rangle=|1\rangle
$$

$$
X|1\rangle=|0\rangle
$$

Si se aplica a un estado general:

$$
|\psi\rangle=\alpha|0\rangle+\beta|1\rangle
$$

se obtiene:

$$
X|\psi\rangle=
\beta|0\rangle+\alpha|1\rangle
$$

Es decir, las amplitudes se intercambian.

Para aplicar $X$ al qubit con máscara `mask`, el simulador recorre las posiciones donde el bit está en cero:

```text
para cada i donde (i & mask) == 0:
    j = i | mask
    intercambiar state[i] y state[j]
```

Con dos qubits y `q0` como objetivo:

```text
00 <-> 01
10 <-> 11
```

Por tanto:

```text
state[0] <-> state[1]
state[2] <-> state[3]
```

La operación conserva automáticamente la norma porque solamente permuta amplitudes.

---

### 3.10 Compuerta Hadamard

Hadamard es la compuerta que introduce la primera diferencia visible entre un bit clásico y un qubit.

Su matriz es:

$$
H=
\frac{1}{\sqrt{2}}
\begin{pmatrix}
1 & 1\\
1 & -1
\end{pmatrix}
$$

Sobre los estados base:

$$
H|0\rangle=
\frac{|0\rangle+|1\rangle}{\sqrt{2}}
=
|+\rangle
$$

$$
H|1\rangle=
\frac{|0\rangle-|1\rangle}{\sqrt{2}}
=
|-\rangle
$$

Si un par de amplitudes antes de la operación es:

$$
(a,b)
$$

después de Hadamard pasa a ser:

$$
a'=
\frac{a+b}{\sqrt{2}}
$$

$$
b'=
\frac{a-b}{\sqrt{2}}
$$

La implementación debe guardar primero ambas amplitudes originales antes de escribir los resultados:

```text
a = state[i]
b = state[j]

state[i] = (a + b) / sqrt(2)
state[j] = (a - b) / sqrt(2)
```

Si se actualizara `state[i]` y luego se utilizara ese nuevo valor para calcular `state[j]`, el resultado sería incorrecto.

Partiendo de:

```text
[1, 0]
```

`H(q)` produce aproximadamente:

```text
[0.707106781..., 0.707106781...]
```

Entonces:

$$
P(0)=\left|\frac{1}{\sqrt2}\right|^2=\frac12
$$

$$
P(1)=\left|\frac{1}{\sqrt2}\right|^2=\frac12
$$

Una propiedad especialmente útil para las pruebas es:

$$
H^2=I
$$

por lo que:

$$
H(H(|0\rangle))=|0\rangle
$$

Esto permite verificar no solo el comportamiento probabilístico de Hadamard, sino también la interferencia que produce el signo negativo de la matriz.

---

### 3.11 Compuerta Pauli-Z y fase

La compuerta $Z$ es:

$$
Z=
\begin{pmatrix}
1 & 0\\
0 & -1
\end{pmatrix}
$$

Su comportamiento sobre la base computacional es:

$$
Z|0\rangle=|0\rangle
$$

$$
Z|1\rangle=-|1\rangle
$$

Por tanto, no intercambia las amplitudes. Solamente cambia el signo de las amplitudes asociadas a estados donde el qubit objetivo vale 1.

La implementación es directa:

```text
para cada i donde (i & mask) != 0:
    state[i] = -state[i]
```

A primera vista podría parecer que $Z$ no hace nada útil, porque $|1\rangle$ y $-|1\rangle$ producen la misma probabilidad al medirse:

$$
|-1|^2=1
$$

Sin embargo, el signo representa una **fase relativa**, que puede modificar resultados posteriores mediante interferencia.

Un ejemplo fundamental es:

```text
H(q);
Z(q);
H(q);
```

Partiendo de $|0\rangle$:

$$
|0\rangle
\xrightarrow{H}
\frac{|0\rangle+|1\rangle}{\sqrt2}
$$

después:

$$
\frac{|0\rangle+|1\rangle}{\sqrt2}
\xrightarrow{Z}
\frac{|0\rangle-|1\rangle}{\sqrt2}
$$

y finalmente:

$$
\frac{|0\rangle-|1\rangle}{\sqrt2}
\xrightarrow{H}
|1\rangle
$$

Por eso `H; Z; H` constituye una buena prueba del simulador: demuestra que las amplitudes no se están tratando simplemente como probabilidades.

---

### 3.12 Compuerta CNOT

`CNOT(control, target)` es la primera compuerta de dos qubits de Quantum Coffee.

Su regla es:

> El qubit objetivo se invierte solamente cuando el qubit de control vale 1.

En términos de bits:

$$
(c,t)
\mapsto
(c,c\oplus t)
$$

donde $\oplus$ representa XOR.

Su matriz, en la convención estándar de dos qubits, es:

$$
CNOT=
\begin{pmatrix}
1&0&0&0\\
0&1&0&0\\
0&0&0&1\\
0&0&1&0
\end{pmatrix}
$$

La implementación por máscaras utiliza dos valores:

```text
controlMask = 1 << controlBit
targetMask  = 1 << targetBit
```

El simulador solo necesita procesar posiciones en las que el control vale 1 y el objetivo vale 0:

```text
para cada i:
    si (i & controlMask) != 0
       y (i & targetMask) == 0:

        j = i | targetMask
        intercambiar state[i] y state[j]
```

La condición `target == 0` es importante porque evita intercambiar el mismo par dos veces.

Con `q0` como control y `q1` como objetivo:

```text
|01> -> |11>
```

porque `q0 = 1`.

En cambio:

```text
|10> -> |10>
```

porque `q0 = 0`.

El análisis semántico debe impedir:

```text
CNOT(q, q);
```

porque control y objetivo deben ser qubits distintos. El simulador debería comprobarlo otra vez como validación defensiva.

---

### 3.13 Creación de un estado entrelazado

La combinación de Hadamard y CNOT permite construir el ejemplo mínimo de entrelazamiento.

Consideremos:

```text
qubit q0;
qubit q1;

H(q0);
CNOT(q0, q1);
```

Ambos qubits comienzan en:

$$
|00\rangle
$$

Aplicar Hadamard sobre `q0` produce:

$$
\frac{|00\rangle+|01\rangle}{\sqrt2}
$$

porque, bajo la convención LSB, `q0` es el bit derecho.

Después, `CNOT(q0,q1)` observa el valor de `q0`. La componente $|00\rangle$ tiene control 0 y no cambia. La componente $|01\rangle$ tiene control 1, por lo que se invierte `q1`:

$$
|01\rangle\rightarrow|11\rangle
$$

El estado final es:

$$
|\Phi^+\rangle=
\frac{|00\rangle+|11\rangle}{\sqrt2}
$$

Internamente:

```text
[
    1/sqrt(2),
    0,
    0,
    1/sqrt(2)
]
```

Este caso demuestra por qué el estado debe ser global. Si se midiera `q0`, el resultado sería 0 o 1 con probabilidad $1/2$, pero el resultado de `q1` quedaría determinado por el colapso de todo el vector conjunto.

---

### 3.14 Medición y Regla de Born

La medición es el punto donde la información cuántica vuelve al mundo clásico de Quantum Coffee.

En la sintaxis:

```text
cafe resultado = medir(q);
```

el simulador devuelve internamente un bit:

```text
0
```

o:

```text
1
```

y el runtime realiza la traducción:

```text
0 -> ralo
1 -> fuerte
```

Para medir un qubit cuyo índice de bit es $b$, se utiliza:

```text
mask = 1 << b
```

La probabilidad de obtener 1 no corresponde a una sola posición del vector cuando existen varios qubits. Deben sumarse todas las probabilidades de los estados base donde ese bit sea 1.

Formalmente:

$$
P(1)=
\sum_{i:\,(i\,\&\,mask)\neq0}
|\alpha_i|^2
$$

y:

$$
P(0)=1-P(1)
$$

Por ejemplo, para dos qubits y medición de `q0`:

$$
P(q_0=1)
=
|\alpha_{01}|^2
+
|\alpha_{11}|^2
$$

porque los estados $|01\rangle$ y $|11\rangle$ tienen el bit menos significativo en 1.

Después de calcular las probabilidades, el simulador genera un número pseudoaleatorio:

$$
r\in[0,1)
$$

y compara:

```text
si r < P(1):
    resultado = 1
si no:
    resultado = 0
```

El generador pseudoaleatorio no “crea” el comportamiento cuántico. El estado determina primero la distribución de probabilidades; el RNG solamente selecciona una de las ramas siguiendo esa distribución.

Si:

$$
P(1)=1
$$

el resultado debe ser 1 independientemente del número aleatorio.

Si:

$$
P(1)=0
$$

el resultado debe ser 0.

---

### 3.15 Colapso del vector de estado

Medir no consiste únicamente en devolver un bit. Después de seleccionar el resultado, el vector debe modificarse para representar el estado posterior a la medición.

Supongamos que el resultado fue 0. Todas las amplitudes correspondientes a estados donde el qubit medido vale 1 deben eliminarse:

$$
\alpha_i'=0
\quad\text{si el bit medido de }i\text{ es }1
$$

Si el resultado fue 1 ocurre lo contrario.

Sin embargo, poner amplitudes en cero reduce la norma del vector. Por eso las amplitudes compatibles con el resultado deben renormalizarse.

Si la probabilidad del resultado obtenido es $P(r)$:

$$
\alpha_i'
=
\frac{\alpha_i}{\sqrt{P(r)}}
$$

para los estados compatibles con el resultado.

Después de la renormalización vuelve a cumplirse:

$$
\sum_i|\alpha_i'|^2=1
$$

Este paso es esencial para que una segunda medición del mismo qubit sea coherente.

Por ejemplo:

```text
qubit q;
H(q);

cafe primero = medir(q);
cafe segundo = medir(q);
```

La primera medición puede producir `ralo` o `fuerte`. La segunda debe producir exactamente el mismo resultado si no se aplica ninguna compuerta entre ambas mediciones.

---

### 3.16 Colapso de un estado entrelazado

El estado de Bell permite observar el comportamiento más importante del algoritmo de medición.

Antes de medir:

$$
|\Phi^+\rangle=
\frac{|00\rangle+|11\rangle}{\sqrt2}
$$

Si se mide `q0` y el resultado es 0, se eliminan todas las componentes con `q0=1`. Por tanto desaparece $|11\rangle$ y queda:

$$
|00\rangle
$$

Medir posteriormente `q1` produce necesariamente 0.

Si la primera medición produce 1, desaparece $|00\rangle$ y queda:

$$
|11\rangle
$$

por lo que `q1` produce necesariamente 1.

Lo importante es que el simulador no necesita un algoritmo especial llamado “entrelazamiento”. El comportamiento surge automáticamente porque la medición recorre y colapsa el vector global.

La misma correlación debe obtenerse independientemente del orden de medición. Si se mide primero `q1` y luego `q0`, los dos resultados siguen siendo iguales.

---

### 3.17 Números complejos

Las compuertas de la V1 —$H$, $X$, $Z$ y $CNOT$— podrían implementarse utilizando solo números reales. Sin embargo, el estado cuántico general utiliza amplitudes complejas:

$$
\alpha_i\in\mathbb{C}
$$

Por eso conviene definir desde el inicio una representación compleja.

Una estructura mínima puede almacenar:

```text
Complex
    real
    imaginary
```

y necesita operaciones de suma, resta, multiplicación, cambio de signo, división por escalar y magnitud al cuadrado.

Para:

$$
z=a+bi
$$

su magnitud al cuadrado es:

$$
|z|^2=a^2+b^2
$$

No hace falta calcular la raíz cuadrada para obtener una probabilidad.

Diseñar el simulador con amplitudes complejas desde el inicio evita cambiar toda la representación si en una versión futura se agregan compuertas como $Y$, $S$, $T$ o rotaciones de fase.

---

### 3.18 Precisión numérica y normalización

La matemática cuántica ideal trabaja con valores exactos, pero una implementación utiliza números de coma flotante.

El valor:

$$
\frac{1}{\sqrt2}
$$

no tiene una representación exacta en un `double`.

Por eso, después de:

```text
H(q);
H(q);
```

el estado matemático es exactamente $|0\rangle$, pero internamente podría observarse algo parecido a:

```text
[0.9999999999999998, 0.0000000000000000]
```

En consecuencia, las pruebas no deben comparar amplitudes con igualdad exacta.

Se puede definir:

$$
\varepsilon=10^{-9}
$$

y aceptar una diferencia cuando:

$$
|x_{esperado}-x_{real}|<\varepsilon
$$

La normalización también debe verificarse con tolerancia:

$$
\left|
\sum_i|\alpha_i|^2-1
\right|
<10^{-9}
$$

Un helper de pruebas puede expresar esta propiedad:

```text
assertNormalized():
    total = 0

    para cada amplitud alpha:
        total += absSquared(alpha)

    verificar abs(total - 1) < 1e-9
```

Esta comprobación debería ejecutarse después de cada compuerta y después de cada medición durante las pruebas. Así, si una operación introduce un error, se identifica inmediatamente en vez de descubrirlo varias instrucciones después.

---

### 3.19 Casos deterministas de medición

La medición necesita manejar correctamente los casos extremos.

Si:

$$
P(1)=0
$$

entonces necesariamente:

$$
P(0)=1
$$

y la medición debe devolver 0 sin riesgo de renormalizar usando una probabilidad nula.

Del mismo modo, si:

$$
P(0)=0
$$

entonces:

$$
P(1)=1
$$

y el resultado es 1.

Dos ejemplos básicos son:

```text
qubit q;
cafe resultado = medir(q);
```

que siempre debe producir `ralo`, y:

```text
qubit q;
X(q);
cafe resultado = medir(q);
```

que siempre debe producir `fuerte`.

Estos casos son particularmente útiles porque detectan errores de división por cero en el algoritmo de colapso.

---

### 3.20 Semilla del generador pseudoaleatorio

Los resultados cuánticos pueden ser probabilísticos, pero las pruebas automatizadas deben ser reproducibles.

Por eso el simulador debe permitir configurar la semilla:

```text
setSeed(42)
```

Dos simuladores ejecutados con la misma semilla y la misma secuencia de operaciones deben producir la misma secuencia pseudoaleatoria.

Esto permite construir pruebas confiables sin convertir la semilla en parte de la sintaxis de Quantum Coffee. `setSeed` pertenece a la API interna del simulador, no al lenguaje.

Para una prueba estadística sencilla de Hadamard pueden utilizarse semillas de 0 a 99:

```text
para seed = 0 .. 99:
    crear simulador
    setSeed(seed)
    allocateQubit("q")
    applyH("q")
    medir("q")
```

A lo largo de esas ejecuciones deben aparecer tanto 0 como 1. No se exige exactamente una distribución 50/50, porque el muestreo aleatorio no garantiza conteos idénticos.

---

### 3.21 Interfaz del módulo `QuantumSimulator`

La interfaz mínima propuesta es:

```text
allocateQubit(id)

applyH(id)
applyX(id)
applyZ(id)

applyCNOT(control, target)

measure(id) -> 0 | 1

stateVector()
probabilities()
setSeed(seed)
```

Las seis primeras operaciones forman el núcleo funcional que necesita el runtime.

`stateVector()` y `probabilities()` son herramientas de observación y prueba. No forman parte de la sintaxis del lenguaje y no deberían permitir modificar directamente el estado interno.

Por ejemplo, para Bell:

```text
stateVector()
```

podría devolver conceptualmente:

```text
[
    0.707106781...,
    0,
    0,
    0.707106781...
]
```

mientras:

```text
probabilities()
```

devuelve:

```text
[
    0.5,
    0,
    0,
    0.5
]
```

La diferencia es importante: `stateVector()` conserva fase y amplitud, mientras `probabilities()` pierde la información de fase al calcular $|\alpha_i|^2$.

---

### 3.22 Integración con el runtime

El simulador puede desarrollarse sin que el parser esté terminado.

Durante las pruebas del módulo se puede trabajar directamente:

```text
simulator.allocateQubit("q0");
simulator.allocateQubit("q1");

simulator.applyH("q0");
simulator.applyCNOT("q0", "q1");

a = simulator.measure("q0");
b = simulator.measure("q1");
```

Posteriormente, el runtime conecta los nodos del lenguaje con esa interfaz.

Si el proyecto utiliza un intérprete directo del AST, un nodo conceptual:

```text
QuantumGate(H, "q0")
```

puede ejecutar:

```text
simulator.applyH("q0")
```

Si el equipo introduce una representación intermedia, la misma operación podría convertirse primero en una instrucción:

```text
H q0
```

y el runtime terminaría llamando al mismo método.

Esto es importante para el desarrollo paralelo: la implementación matemática del simulador no depende de que el equipo ya haya decidido si el compilador ejecutará AST directamente o generará una IR.

---

### 3.23 Medición como frontera entre lo cuántico y lo clásico

La medición ocupa una posición especial porque conecta ambos modelos de ejecución.

Antes de medir:

```text
qubit q;
H(q);
```

el estado de `q` no es un valor de tipo `cafe`.

No debería ser válido:

```text
cafe resultado = q;
```

La conversión ocurre únicamente mediante:

```text
cafe resultado = medir(q);
```

Conceptualmente:

```text
QuantumSimulator
      |
      | measure(q)
      v
    0 o 1
      |
      v
Runtime clásico
      |
      +--> 0 = ralo
      |
      +--> 1 = fuerte
```

A partir de ese momento el resultado ya es clásico y puede utilizarse normalmente:

```text
hierve (resultado == fuerte) {
    servir("Se midio 1");
}
```

El simulador no necesita conocer `fuerte` ni `ralo`; esos nombres pertenecen al runtime de Quantum Coffee.

---

### 3.24 Restricciones semánticas de la V1

La gramática define qué programas pueden escribirse sintácticamente, pero varias restricciones deben comprobarse durante el análisis semántico.

Los qubits de la V1 solamente pueden declararse en el nivel superior del bloque de `chorreador`.

Esto es válido:

```text
barista chorreador() {
    grano x = 10;

    qubit q;

    H(q);
}
```

pero esto no:

```text
barista chorreador() {
    hierve (condicion) {
        qubit q;
    }
}
```

y tampoco:

```text
barista otraFuncion() {
    qubit q;
}
```

La restricción simplifica el tiempo de vida de los qubits y evita que la primera implementación tenga que manejar qubits creados dinámicamente dentro de ciclos, recursión o llamadas anidadas.

También deben comprobarse casos como:

```text
H(q);
qubit q;
```

porque el qubit se utiliza antes de declararse.

De igual forma:

```text
grano x = 10;
H(x);
```

debe distinguirse de:

```text
H(q);
```

cuando `q` ni siquiera está declarado. Ambos son errores, pero representan problemas semánticos diferentes.

---

### 3.25 Evaluación lógica y mediciones

`medir(q)` puede utilizarse como parte de una expresión porque la gramática lo incorpora como `factor`.

Por ejemplo:

```text
hierve (medir(q) == fuerte) {
    servir("Se midio 1");
}
```

Esto introduce un detalle importante: medir cambia el estado cuántico. Por tanto, el orden de evaluación de las expresiones clásicas puede cambiar qué mediciones llegan a ejecutarse.

Consideremos:

```text
medir(q0) == fuerte ConAzucar medir(q1) == fuerte
```

Si `ConAzucar` utiliza cortocircuito y el primer operando resulta falso, el segundo podría no evaluarse. En ese caso `q1` no se mide y su estado permanece intacto.

Si el lenguaje evalúa ambos operandos siempre, ambos qubits se miden.

La parte cuántica no debe inventar una regla distinta. La política correcta es:

> La evaluación de `medir` dentro de una expresión sigue exactamente la política de evaluación del operador clásico que contiene esa expresión.

De esta forma el comportamiento cuántico permanece coherente con la semántica general de Quantum Coffee.

---

## 4. Puntos de Confusión y Casos de Esquina

### Un qubit no es un booleano oculto

Una interpretación incorrecta sería almacenar internamente:

```text
q = 0
```

y hacer que `H(q)` simplemente seleccione aleatoriamente 0 o 1.

Eso no reproduce computación cuántica. Si el sistema se redujera a un bit aleatorio, operaciones como:

```text
H(q);
H(q);
```

no podrían recuperar determinísticamente $|0\rangle$.

El simulador debe mantener amplitudes precisamente porque la interferencia depende de ellas.

---

### El RNG no es el simulador

Otra confusión frecuente es pensar que el comportamiento cuántico puede implementarse únicamente mediante `random()`.

El número aleatorio interviene solo después de que el estado haya determinado las probabilidades.

La secuencia correcta es:

```text
estado
  ->
calcular P(0), P(1)
  ->
muestrear
  ->
colapsar
  ->
renormalizar
```

No:

```text
random() -> fingir que el qubit era 0 o 1
```

---

### La probabilidad no conserva la fase

Dos estados pueden tener exactamente las mismas probabilidades y comportarse de forma diferente después de otra compuerta.

Por ejemplo:

$$
|+\rangle=
\frac{|0\rangle+|1\rangle}{\sqrt2}
$$

y:

$$
|-\rangle=
\frac{|0\rangle-|1\rangle}{\sqrt2}
$$

tienen:

$$
P(0)=P(1)=\frac12
$$

pero:

$$
H|+\rangle=|0\rangle
$$

mientras:

$$
H|-\rangle=|1\rangle
$$

Por eso `probabilities()` no puede reemplazar a `stateVector()` como representación interna.

---

### Bell no detecta por sí solo un orden de bits incorrecto

El estado de Bell:

$$
\frac{|00\rangle+|11\rangle}{\sqrt2}
$$

es simétrico respecto al intercambio de los dos qubits. Por eso una implementación podría tener `q0` y `q1` invertidos internamente y aun así pasar una prueba básica de Bell.

Se necesitan pruebas específicas:

```text
X(q0)
```

debe producir amplitud 1 en el índice 1, mientras:

```text
X(q1)
```

debe producir amplitud 1 en el índice 2.

Estas pruebas fijan la convención LSB de manera ejecutable.

---

### Control y objetivo de CNOT no son intercambiables

```text
CNOT(q0, q1)
```

y:

```text
CNOT(q1, q0)
```

son operaciones diferentes.

Una implementación que use incorrectamente la máscara del control como objetivo puede producir resultados plausibles en circuitos simétricos, por lo que deben probarse estados no triviales.

---

### Medir no destruye el qubit

La medición colapsa el estado, pero el qubit continúa existiendo.

Esto es válido:

```text
qubit q;

H(q);

cafe primero = medir(q);

H(q);

cafe segundo = medir(q);
```

No se necesita un estado artificial `MEDIDO` que impida volver a utilizar el qubit.

---

### No comparar `double` con igualdad exacta

Expresiones que matemáticamente producen 1 pueden generar:

```text
0.9999999999999998
```

en coma flotante.

Las pruebas deben utilizar una tolerancia y nunca depender de igualdad exacta para amplitudes o probabilidades calculadas.

---

## 5. Plan de Pruebas del Simulador

La implementación debe validarse inicialmente sin depender del compilador. Todas las pruebas pueden construirse directamente sobre la API de `QuantumSimulator`.

### 5.1 Pauli-X

```text
allocateQubit("q");
applyX("q");
measure("q");
```

Resultado esperado:

```text
1
```

siempre.

Esto comprueba:

$$
X|0\rangle=|1\rangle
$$

---

### 5.2 Hadamard dos veces

```text
allocateQubit("q");
applyH("q");
applyH("q");
```

El vector debe ser aproximadamente:

```text
[1, 0]
```

porque:

$$
H^2=I
$$

Una medición posterior debe devolver 0 con certeza.

---

### 5.3 Interferencia con Z

```text
allocateQubit("q");
applyH("q");
applyZ("q");
applyH("q");
```

Resultado esperado:

$$
|1\rangle
$$

por lo que la medición devuelve 1 con certeza.

Esta prueba comprueba especialmente que el signo de las amplitudes se conserva.

---

### 5.4 Bell y correlación

```text
allocateQubit("q0");
allocateQubit("q1");

applyH("q0");
applyCNOT("q0", "q1");

a = measure("q0");
b = measure("q1");
```

Debe cumplirse:

```text
a == b
```

Los únicos pares válidos son:

```text
0 0
1 1
```

La prueba debe repetirse también midiendo en orden inverso:

```text
b = measure("q1");
a = measure("q0");
```

y la correlación debe conservarse.

---

### 5.5 Doble medición

```text
allocateQubit("q");
applyH("q");

a = measure("q");
b = measure("q");
```

Debe cumplirse:

```text
a == b
```

porque la primera medición colapsó el estado.

---

### 5.6 Orden LSB

Con dos qubits:

```text
applyX("q0")
```

debe producir:

```text
[0, 1, 0, 0]
```

y:

```text
applyX("q1")
```

debe producir:

```text
[0, 0, 1, 0]
```

Estas pruebas validan directamente el mapa `id -> bitIndex`.

---

### 5.7 CNOT con control y objetivo no triviales

Caso 1:

```text
X(q0);
CNOT(q0, q1);
```

debe producir:

$$
|11\rangle
$$

Caso 2:

```text
X(q1);
CNOT(q0, q1);
```

debe permanecer en:

$$
|10\rangle
$$

porque el control `q0` vale 0.

Caso 3:

```text
X(q1);
CNOT(q1, q0);
```

debe producir:

$$
|11\rangle
$$

Estos casos detectan confusiones entre control y objetivo.

---

### 5.8 Normalización como invariante

Después de cada operación debe verificarse:

$$
\left|
\sum_i|\alpha_i|^2-1
\right|
<10^{-9}
$$

Esta prueba no debe ejecutarse únicamente al final del circuito. Debe formar parte de los helpers comunes de la batería para detectar la operación exacta que introduzca un error.

---

### 5.9 Hadamard como prueba probabilística

Para `H(q)` sobre $|0\rangle$:

$$
P(0)=P(1)=\frac12
$$

Con semillas de 0 a 99, deben aparecer ambos resultados al menos una vez.

No se exige exactamente 50 resultados de cada tipo, porque el muestreo puede producir cualquier distribución compatible con la probabilidad.

---

## 6. Ejecución Completa de un Programa

Considérese:

```text
barista chorreador() {
    qubit q0;
    qubit q1;

    H(q0);
    CNOT(q0, q1);

    cafe a = medir(q0);
    cafe b = medir(q1);

    servir(a);
    servir(b);
}
```

Al comenzar `chorreador`, el simulador está vacío:

```text
state = [1]
```

Después de `qubit q0;`:

```text
state = [1, 0]
q0 -> bit 0
```

Después de `qubit q1;`:

```text
state = [1, 0, 0, 0]
q0 -> bit 0
q1 -> bit 1
```

El sistema está en:

$$
|00\rangle
$$

Después de `H(q0)`:

$$
|\psi\rangle=
\frac{|00\rangle+|01\rangle}{\sqrt2}
$$

por lo que:

```text
state = [
    1/sqrt(2),
    1/sqrt(2),
    0,
    0
]
```

Después de `CNOT(q0,q1)`:

$$
|\psi\rangle=
\frac{|00\rangle+|11\rangle}{\sqrt2}
$$

y:

```text
state = [
    1/sqrt(2),
    0,
    0,
    1/sqrt(2)
]
```

Al ejecutar `medir(q0)` existen dos resultados posibles.

Si se obtiene 0, el vector colapsa a:

```text
[1, 0, 0, 0]
```

correspondiente a:

$$
|00\rangle
$$

Entonces `medir(q1)` produce necesariamente 0.

El runtime recibe:

```text
0
0
```

y los convierte en:

```text
ralo
ralo
```

Si la primera medición produce 1, el vector colapsa a:

```text
[0, 0, 0, 1]
```

correspondiente a:

$$
|11\rangle
$$

Entonces la segunda medición también produce 1 y el runtime muestra:

```text
fuerte
fuerte
```

Este ejemplo resume prácticamente toda la arquitectura cuántica V1: asignación, vector global, Hadamard, CNOT, entrelazamiento, medición, colapso y conversión de información cuántica a un tipo clásico.

---

## 7. Para Estudiar

**Pregunta 1.** ¿Por qué Quantum Coffee no puede representar cada qubit con un vector independiente de dos amplitudes cuando existe CNOT?

> Porque CNOT puede crear estados entrelazados. Un estado como $(|00\rangle+|11\rangle)/\sqrt2$ no puede escribirse como producto tensorial de dos estados individuales, por lo que el simulador necesita representar el sistema completo mediante un único vector de $2^n$ amplitudes.

---

**Pregunta 2.** ¿Por qué el simulador necesita amplitudes y no solamente probabilidades?

> Porque la fase relativa entre amplitudes afecta operaciones posteriores mediante interferencia. Los estados $|+\rangle$ y $|-\rangle$ tienen las mismas probabilidades de medición inmediata, pero Hadamard transforma el primero en $|0\rangle$ y el segundo en $|1\rangle$. Una representación basada únicamente en probabilidades perdería esa información.

---

**Pregunta 3.** ¿Qué función cumple la máscara `1 << bitIndex`?

> Identifica la posición binaria que corresponde a un qubit dentro del índice del vector de estado. Permite saber si ese qubit vale 0 o 1 en cada estado base y localizar la amplitud pareja que difiere únicamente en ese bit.

---

**Pregunta 4.** ¿Por qué la medición debe renormalizar el vector?

> Porque al colapsar se eliminan las amplitudes incompatibles con el resultado observado. Las amplitudes restantes tienen una suma de probabilidades igual a la probabilidad del resultado, no necesariamente 1. Dividirlas por $\sqrt{P(resultado)}$ restaura la norma unitaria.

---

**Pregunta 5.** ¿Por qué `medir(q)` devuelve `cafe` y no `qubit`?

> Porque la medición produce información clásica. El simulador devuelve 0 o 1 y el runtime los representa como `ralo` y `fuerte`, que ya pertenecen al sistema booleano clásico de Quantum Coffee.

---

**Pregunta 6.** ¿Qué demuestra la secuencia `H(q); Z(q); H(q);`?

> Demuestra que la fase importa. Después del primer Hadamard el qubit está en $|+\rangle$. Z cambia la fase relativa y produce $|-\rangle$. El segundo Hadamard transforma ese estado en $|1\rangle$. Una simulación que solo almacenara probabilidades no podría reproducir este resultado.

---

**Pregunta 7.** ¿Por qué 16 qubits es un límite razonable para la V1?

> Porque el simulador clásico almacena $2^n$ amplitudes. Con 16 qubits existen 65536 amplitudes, aproximadamente 1 MiB si cada amplitud compleja utiliza dos `double`. Es suficiente para circuitos académicos y evita permitir tamaños que crezcan rápidamente hasta consumir grandes cantidades de memoria.

---

**Pregunta 8.** ¿Qué parte del sistema conoce que 0 significa `ralo` y 1 significa `fuerte`?

> El runtime de Quantum Coffee. El simulador solamente trabaja con estados cuánticos y devuelve el bit 0 o 1 después de la medición. La traducción hacia los valores del lenguaje debe mantenerse fuera del simulador.

---

## 8. Resumen de Responsabilidades

La implementación queda dividida en responsabilidades claras.

El **lexer** reconoce las palabras reservadas cuánticas.

El **parser** construye la estructura sintáctica de declaraciones, compuertas y mediciones.

El **AST** representa esas instrucciones sin realizar todavía la matemática.

El **análisis semántico** comprueba que los identificadores sean qubits válidos, que estén declarados, que CNOT reciba dos qubits distintos y que las declaraciones respeten las restricciones de la V1.

El **runtime** decide cuándo ejecutar cada instrucción y convierte el resultado de `measure` en `ralo` o `fuerte`.

El **QuantumSimulator** mantiene el vector de estado, aplica las compuertas, calcula probabilidades, realiza el muestreo, colapsa y renormaliza.

Esta separación permite desarrollar el simulador de forma independiente mientras el resto del equipo continúa trabajando en el compilador clásico. La integración final no exige que el simulador conozca la sintaxis completa del lenguaje: basta con conectar los nodos o instrucciones cuánticas con una interfaz pequeña y estable.
