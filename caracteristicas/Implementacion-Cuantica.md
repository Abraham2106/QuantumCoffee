# Implementación cuántica de Quantum Coffee

## 1. Propósito y alcance

La parte cuántica de Quantum Coffee se plantea como una extensión pequeña del lenguaje, que continúa siendo principalmente imperativo y clásico. Su propósito es permitir que un programa declare qubits, aplique compuertas cuánticas y obtenga resultados clásicos mediante medición. La implementación está pensada para un proyecto universitario de Compiladores, con circuitos pequeños que permitan comprender la superposición, la interferencia y el entrelazamiento sin incorporar la complejidad de una plataforma cuántica completa.

La primera versión incluirá las primitivas `qubit`, `H`, `X`, `Z`, `CNOT` y `medir`. La ejecución se realizará en una computadora clásica mediante un simulador propio, llamado `QuantumSimulator`, que mantendrá el estado cuántico y aplicará las operaciones correspondientes. No se requiere hardware cuántico ni integrar frameworks externos como Qiskit o Cirq. El máximo inicial será de 16 qubits, una cantidad suficiente para los ejemplos y las pruebas del proyecto.

Desde el punto de vista del programador, estas operaciones se integran con las instrucciones habituales del lenguaje. Un programa puede declarar dos qubits, aplicar una compuerta Hadamard al primero, relacionarlos mediante CNOT y almacenar las mediciones en variables de tipo `cafe`:

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

Aunque la sintaxis se parece a una llamada de función, las compuertas actúan sobre amplitudes del estado cuántico. El simulador devuelve `0` o `1` al medir, y el runtime de Quantum Coffee convierte esos resultados en `ralo` o `fuerte`, respectivamente. Esta conversión permite utilizar el resultado dentro de las expresiones y estructuras de control clásicas del lenguaje.

## 2. Integración con el compilador

La extensión cuántica debe incorporarse a las mismas etapas que procesan el resto del lenguaje. El lexer reconocerá `qubit`, `H`, `X`, `Z`, `CNOT` y `medir` como palabras reservadas con tokens propios. El parser reconocerá las declaraciones, las compuertas y las mediciones, y construirá los nodos correspondientes en el árbol de sintaxis abstracta, o AST. Reservar estos nombres permite distinguir las primitivas cuánticas de las llamadas a funciones ordinarias.

El parser comprobará la estructura de las instrucciones, mientras que el análisis semántico verificará si sus argumentos son válidos. Por ejemplo, `grano x = 10; H(x);` tiene una forma sintáctica reconocible, pero debe rechazarse porque `x` es una variable clásica. De manera similar, `H(q);` será inválido si `q` no se ha declarado, y `CNOT(q, q);` deberá rechazarse porque el control y el objetivo deben ser qubits diferentes.

En la tabla de símbolos, los qubits se identificarán mediante una categoría propia, `QUBIT`, separada de las variables clásicas. Esta distinción permitirá impedir asignaciones como `grano x = q;` o `cafe resultado = q;`. Un qubit no se utiliza directamente como un valor clásico; la conversión se expresa mediante `medir(q)`, cuyo resultado pertenece al tipo `cafe`. El compilador necesita conocer la declaración y la categoría del símbolo, pero no sus amplitudes ni sus probabilidades, que serán responsabilidad del simulador.

Para simplificar la primera versión, las declaraciones de qubits se permitirán únicamente en el nivel superior del bloque de `chorreador`, fuera de ciclos y bloques anidados. Tampoco podrán declararse dentro de otras funciones. Esta decisión evita tener que resolver desde el inicio la creación repetida de qubits, la recursión y la liberación de recursos cuánticos. Cada qubit deberá declararse antes de utilizarse, y su asignación se conservará durante la ejecución del programa.

## 3. Representación del estado cuántico

Un qubit se representa mediante dos amplitudes, que describen su estado respecto a los valores de la base computacional. Matemáticamente, se escribe como $|\psi\rangle = \alpha|0\rangle + \beta|1\rangle$, donde $\alpha$ y $\beta$ son números complejos. Estas amplitudes no son probabilidades: las probabilidades de medir cero o uno se calculan como $P(0)=|\alpha|^2$ y $P(1)=|\beta|^2$. Para que el estado esté normalizado debe cumplirse $|\alpha|^2+|\beta|^2=1$.

$$
|\psi\rangle=\alpha|0\rangle+\beta|1\rangle
=\begin{pmatrix}\alpha\\\beta\end{pmatrix},
\qquad \alpha,\beta\in\mathbb{C}
$$

$$
P(0)=|\alpha|^2,\qquad P(1)=|\beta|^2,
\qquad |\alpha|^2+|\beta|^2=1
$$

Estas probabilidades se obtienen mediante la regla de Born. Para una amplitud compleja $z=a+bi$, el simulador calculará su magnitud al cuadrado como $|z|^2=a^2+b^2$.

Cada qubit nuevo comienza en el estado $|0\rangle$, representado por el vector `[1, 0]`. Sin embargo, cuando existen varios qubits, el simulador debe mantener un único vector para todo el sistema. Con dos qubits se necesitan cuatro amplitudes, correspondientes a los estados $|00\rangle$, $|01\rangle$, $|10\rangle$ y $|11\rangle$. En general, un sistema de $n$ qubits requiere $2^n$ amplitudes. Guardar un vector independiente para cada qubit no permitiría representar correctamente los estados entrelazados.

$$
|\psi\rangle=\sum_{i=0}^{2^n-1}\alpha_i|i\rangle,
\qquad \sum_{i=0}^{2^n-1}|\alpha_i|^2=1
$$

El simulador comenzará con `stateVector = [1]`, que representa el sistema antes de asignar qubits. Cada llamada a `allocateQubit(id)` agregará un qubit en cero y duplicará el tamaño del vector, conservando las amplitudes anteriores en la mitad inferior y colocando ceros en la mitad superior. Si todavía no se han aplicado compuertas, la primera asignación producirá `[1, 0]` y la segunda `[1, 0, 0, 0]`. El módulo también mantendrá un mapa entre cada identificador y su posición interna, rechazando identificadores duplicados y asignaciones que superen el límite permitido.

La asignación se expresa mediante el producto tensorial $|\psi'\rangle=|0\rangle\otimes|\psi\rangle$. Con la convención de índices utilizada, el nuevo qubit se incorpora a la izquierda del estado anterior y comienza en cero.

Quantum Coffee utilizará la convención de que el primer qubit asignado corresponde al bit menos significativo del índice del vector. Así, `q0` ocupará el bit 0 y `q1` el bit 1; los estados de dos qubits se leerán en el orden $|q_1q_0\rangle$. Partiendo de ambos qubits en cero, `X(q0)` deberá producir `[0, 1, 0, 0]`, mientras que `X(q1)` deberá producir `[0, 0, 1, 0]`. Mantener esta convención en todas las operaciones evita confundir los qubits al interpretar el estado.

El límite de 16 qubits responde al crecimiento exponencial del vector. Con esa cantidad se almacenan 65 536 amplitudes, que ocupan aproximadamente 1 MiB si cada número complejo utiliza dos valores `double` de 64 bits, sin contar estructuras auxiliares. Se propone utilizar amplitudes complejas desde el inicio, aunque las compuertas de esta versión pueden operar con valores reales, para permitir futuras extensiones sin sustituir la representación del estado.

## 4. Aplicación de compuertas

Las compuertas se aplicarán directamente sobre las amplitudes, sin construir una matriz completa de tamaño $2^n \times 2^n$. Para localizar las posiciones afectadas se utilizará una máscara de bits, `mask = 1 << bitIndex`. Esta permite identificar los estados base donde el qubit objetivo vale cero o uno y formar pares de posiciones que solo difieren en ese bit. De esta manera, las operaciones pueden implementarse recorriendo el vector de estado.

Para una compuerta de un qubit, cada par de amplitudes se transforma mediante su matriz de tamaño $2\times2$. Si las amplitudes originales son $a$ y $b$, la operación general es:

$$
\begin{pmatrix}a'\\b'\end{pmatrix}
=U\begin{pmatrix}a\\b\end{pmatrix}
$$

Aquí, $U$ representa la compuerta aplicada y los valores con prima son las amplitudes resultantes. Las matrices siguientes describen esta transformación, aunque el código puede calcularla directamente sin construirlas.

La compuerta `X` intercambia las amplitudes de cada par. En un qubit aislado transforma $|0\rangle$ en $|1\rangle$ y $|1\rangle$ en $|0\rangle$. Su implementación consiste en recorrer las posiciones donde el bit objetivo es cero e intercambiarlas con sus correspondientes posiciones donde vale uno. Procesar cada par una sola vez evita deshacer el intercambio durante el mismo recorrido.

$$
X=\begin{pmatrix}0&1\\1&0\end{pmatrix},
\qquad X\begin{pmatrix}a\\b\end{pmatrix}
=\begin{pmatrix}b\\a\end{pmatrix}
$$

$$
X|0\rangle=|1\rangle,\qquad X|1\rangle=|0\rangle
$$

La compuerta `H`, o Hadamard, permite producir superposición. Si las amplitudes de un par son $a$ y $b$, los nuevos valores se calculan como $(a+b)/\sqrt{2}$ y $(a-b)/\sqrt{2}$. Ambas amplitudes originales deben guardarse antes de escribir los resultados. Aplicada a un qubit en cero, Hadamard produce `[1/sqrt(2), 1/sqrt(2)]`, por lo que una medición puede devolver cero o uno con igual probabilidad. Aplicar Hadamard dos veces devuelve el estado inicial, una propiedad útil para comprobar que el simulador conserva las amplitudes y representa la interferencia.

$$
H=\frac{1}{\sqrt{2}}\begin{pmatrix}1&1\\1&-1\end{pmatrix},
\qquad
H\begin{pmatrix}a\\b\end{pmatrix}
=\frac{1}{\sqrt{2}}\begin{pmatrix}a+b\\a-b\end{pmatrix}
$$

$$
H|0\rangle=\frac{|0\rangle+|1\rangle}{\sqrt{2}},
\qquad
H|1\rangle=\frac{|0\rangle-|1\rangle}{\sqrt{2}},
\qquad H^2=I
$$

El símbolo $I$ representa la identidad, es decir, una operación que deja el estado sin cambios. El signo negativo de Hadamard permite que las amplitudes se cancelen al volver a aplicar la compuerta.

La compuerta `Z` cambia el signo de las amplitudes asociadas a estados donde el qubit objetivo vale uno. Este cambio no modifica por sí solo las probabilidades de una medición en la base computacional, pero puede alterar los resultados de operaciones posteriores. Por ejemplo, la secuencia `H(q); Z(q); H(q);`, aplicada sobre un qubit inicialmente en cero, lo lleva al estado uno. Este comportamiento muestra por qué el simulador debe conservar las amplitudes y sus fases, en lugar de almacenar únicamente probabilidades.

$$
Z=\begin{pmatrix}1&0\\0&-1\end{pmatrix},
\qquad
Z\begin{pmatrix}a\\b\end{pmatrix}
=\begin{pmatrix}a\\-b\end{pmatrix}
$$

$$
|0\rangle\xrightarrow{H}
\frac{|0\rangle+|1\rangle}{\sqrt{2}}
\xrightarrow{Z}
\frac{|0\rangle-|1\rangle}{\sqrt{2}}
\xrightarrow{H}|1\rangle
$$

La compuerta `CNOT(control, objetivo)` invierte el qubit objetivo en las componentes del estado donde el control vale uno. Su implementación utiliza una máscara para cada qubit e intercambia las amplitudes correspondientes a posiciones con control en uno y objetivo en cero. El orden de los argumentos es importante: `CNOT(q0, q1)` y `CNOT(q1, q0)` pueden producir resultados distintos. La operación transforma el estado de manera coherente y no realiza una medición del control.

Si $c$ representa el control y $t$ el objetivo, su acción sobre la base computacional se expresa como $|c,t\rangle\mapsto|c,t\oplus c\rangle$, donde $\oplus$ es la operación XOR. Para `CNOT(q0, q1)`, usando el orden del simulador $|q_1q_0\rangle$ y la base ordenada $|00\rangle,|01\rangle,|10\rangle,|11\rangle$, la matriz es:

$$
\mathrm{CNOT}_{q_0\rightarrow q_1}=
\begin{pmatrix}
1&0&0&0\\
0&0&0&1\\
0&0&1&0\\
0&1&0&0
\end{pmatrix}
$$

Esta matriz intercambia las componentes $|01\rangle$ y $|11\rangle$, mientras deja intactas $|00\rangle$ y $|10\rangle$. Especificar el orden de los qubits es necesario porque la disposición de la matriz depende de cuál actúa como control.

La combinación `H(q0); CNOT(q0, q1);`, partiendo de dos qubits en cero, produce el estado de Bell $(|00\rangle+|11\rangle)/\sqrt{2}$, representado por `[1/sqrt(2), 0, 0, 1/sqrt(2)]`. En este estado, las mediciones de ambos qubits están correlacionadas: se obtiene el par cero-cero o uno-uno, cada uno con probabilidad de un medio. No hace falta una estructura adicional para indicar que están entrelazados, porque esa relación ya está contenida en el vector global.

$$
|00\rangle\xrightarrow{H(q_0)}
\frac{|00\rangle+|01\rangle}{\sqrt{2}}
\xrightarrow{\mathrm{CNOT}(q_0,q_1)}
\frac{|00\rangle+|11\rangle}{\sqrt{2}}
=|\Phi^+\rangle
$$

$$
|\Phi^+\rangle=\frac{1}{\sqrt{2}}
\begin{pmatrix}1\\0\\0\\1\end{pmatrix}
$$

## 5. Medición y resultados clásicos

La medición convierte el resultado de una operación cuántica en un valor que el programa puede utilizar. Para medir un qubit, el simulador calculará primero la probabilidad de obtener uno, sumando las magnitudes al cuadrado de todas las amplitudes cuyos estados base tengan ese qubit en uno. La probabilidad de obtener cero será el complemento. Con dos qubits, por ejemplo, la probabilidad de medir `q0` en uno se obtiene sumando las probabilidades de $|01\rangle$ y $|11\rangle$.

Si $b(i)$ indica el valor del bit correspondiente al qubit medido en el índice $i$, el cálculo es:

$$
P(1)=\sum_{i:\,b(i)=1}|\alpha_i|^2,
\qquad P(0)=1-P(1)
$$

$$
P(q_0=1)=|\alpha_{01}|^2+|\alpha_{11}|^2
$$

Después de calcular la distribución, el simulador utilizará un generador pseudoaleatorio para seleccionar el resultado. Si genera un número $u\in[0,1)$, devolverá uno cuando $u<P(1)$ y cero en el caso contrario. La aleatoriedad interviene en esta selección; las probabilidades dependen del estado construido por las compuertas. Por eso, aplicar Hadamard no puede implementarse simplemente asignando un bit aleatorio al qubit.

La medición también debe actualizar el vector. Las amplitudes incompatibles con el resultado se ponen en cero y las restantes se dividen entre la raíz cuadrada de la probabilidad del resultado obtenido. Este proceso de colapso y renormalización permite que el estado siga siendo válido. Los casos con probabilidad cero o uno deben tratarse correctamente para evitar seleccionar un resultado imposible o dividir entre cero.

Si el resultado obtenido es $r\in\{0,1\}$, las amplitudes posteriores a la medición se calculan así:

$$
\alpha_i'=\begin{cases}
\dfrac{\alpha_i}{\sqrt{P(r)}},&\text{si }b(i)=r,\\[6pt]
0,&\text{si }b(i)\ne r.
\end{cases}
$$

El divisor utiliza la probabilidad del resultado seleccionado y garantiza que las amplitudes restantes vuelvan a cumplir la condición de normalización.

Una segunda medición del mismo qubit deberá producir el mismo resultado si no se aplica ninguna compuerta entre ambas. Esto también explica la correlación del estado de Bell: al medir el primer qubit, el sistema queda en $|00\rangle$ o en $|11\rangle$, y la medición del segundo coincide con la primera. Medir no elimina el qubit; después de la medición pueden aplicarse nuevas compuertas y volver a medirlo.

El simulador devolverá únicamente el bit obtenido. El runtime será responsable de convertir `0` en `ralo` y `1` en `fuerte`, y de entregar ese valor a la expresión que contiene `medir(q)`. Si la medición aparece dentro de una expresión lógica, deberá respetar el orden de evaluación y las reglas de cortocircuito del lenguaje. Una medición que no llegue a evaluarse no debe ejecutarse ni provocar un colapso adicional.

## 6. Interfaz e integración del simulador

El módulo `QuantumSimulator` ofrecerá los métodos `allocateQubit(id)`, `applyH(id)`, `applyX(id)`, `applyZ(id)`, `applyCNOT(control, target)` y `measure(id)`. Estos métodos cubren las operaciones que necesita el runtime para ejecutar las primitivas de la primera versión. El módulo mantendrá internamente el vector de estado, el mapa de identificadores y el generador pseudoaleatorio, sin depender de los nombres de los tipos clásicos ni de la sintaxis completa de Quantum Coffee.

Para facilitar las pruebas también se proponen `stateVector()`, `probabilities()` y `setSeed(seed)`. Los dos primeros permitirán consultar el estado y su distribución de probabilidades sin modificar directamente los datos internos. La semilla permitirá reproducir una secuencia de mediciones durante las pruebas. Estas funciones serán parte de la API interna del simulador y no requieren nuevas instrucciones en el lenguaje.

La integración consistirá en conectar cada nodo cuántico del AST con el método correspondiente. Una declaración llamará a `allocateQubit`, una compuerta Hadamard llamará a `applyH` y una medición llamará a `measure`, seguida de la conversión al tipo `cafe`. Si el compilador utiliza una representación intermedia, las mismas operaciones podrán pasar por ella antes de ejecutarse. Esta separación permite desarrollar y comprobar el simulador mientras se completa el procesamiento del lenguaje.

## 7. Validación inicial

Las primeras pruebas se realizarán directamente sobre `QuantumSimulator`. Un qubit recién asignado deberá medirse como cero, mientras que después de aplicar `X` deberá medirse como uno. La secuencia `H; H` deberá recuperar el estado inicial, y `H; Z; H` deberá transformar cero en uno. Estas comprobaciones permiten verificar la asignación, el intercambio de amplitudes y la interferencia antes de integrar el módulo con el compilador.

Con dos qubits se comprobará que el estado de Bell produzca resultados iguales al medir ambos, independientemente del orden de medición. También se verificará que medir dos veces el mismo qubit sin operaciones intermedias conserve el resultado. La convención de índices y el orden de los argumentos de CNOT necesitarán pruebas propias, porque un ejemplo simétrico como Bell puede ocultar una inversión de posiciones o una confusión entre control y objetivo.

Las amplitudes y probabilidades calculadas se compararán con una tolerancia, por ejemplo $10^{-9}$, debido al uso de números de coma flotante. Después de las compuertas y las mediciones se verificará que la suma de las magnitudes al cuadrado siga siendo aproximadamente uno. Para Hadamard sobre cero se comprobarán probabilidades cercanas a un medio y se realizarán mediciones sobre estados preparados de nuevo, utilizando una semilla controlada. No se exigirá obtener exactamente la misma cantidad de ceros y unos en una muestra finita.

La comprobación numérica del estado puede expresarse como:

$$
\left|\sum_{i=0}^{2^n-1}|\alpha_i|^2-1\right|<10^{-9}
$$

Finalmente, se comprobará que el compilador rechace variables clásicas usadas como qubits, identificadores no declarados, declaraciones fuera de los lugares permitidos y llamadas a CNOT con argumentos iguales. El simulador también verificará sus argumentos y el límite de asignación. Una vez superadas estas pruebas, el programa de ejemplo permitirá comprobar la ejecución completa: las declaraciones crean los qubits, las compuertas preparan el estado entrelazado y las mediciones producen `ralo, ralo` o `fuerte, fuerte` mediante la conversión del runtime.
