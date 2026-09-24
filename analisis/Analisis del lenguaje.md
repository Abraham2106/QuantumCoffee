## 3. Análisis del lenguaje

### 3.1 Elementos sólidos

**Una jerarquía de expresiones explícita y sin recursión izquierda:** La precedencia se codifica en la estructura de la gramática: `expresion_or` → `expresion_and` → `expresion_not` → `expresion_relacional` → `expresion_aritmetica` → `termino` → `factor`. Cada nivel usa repeticiones `{ ... }` en lugar de recursión izquierda, así que cada producción se convierte casi directamente en una función de un analizador descendente recursivo. La precedencia no depende de tablas externas. Por ejemplo, `2 + 3 * 4` se agrupa como `2 + (3 * 4)` porque `*` vive en término, un nivel más profundo que `+`.

**Los bloques con llaves obligatorias eliminan el "else colgante":** En C, `if (a) if (b) x(); else y();` obliga al programador a saber a cuál `if` pertenece el `else`. En Quantum Coffee `hierve` y `enfria` exigen `bloque`, por lo que la intención siempre queda escrita:

```
hierve (a) {
    hierve (b) { servir("a y b") ; }
} enfria {
    servir("no a") ;
}
```

**La asignación es una sentencia, no una expresión:** `=` solo aparece en `asignacion` y en las declaraciones; dentro de una condición solo existe `==`. Por eso el clásico error de C queda fuera del lenguaje:

```
hierve (temperatura = 90) { ... }   # rechazado: "=" no forma parte de expresion
```

**Ámbito visible desde la declaración:** Las variables globales llevan la palabra `cafeteria`, y las locales no llevan prefijo. Quien lee el código distingue el alcance de una variable sin buscar dónde se declaró, y no se crean globales por accidente. Además, `chorreador` se declara con `barista` igual que cualquier función, así que hay una sola sintaxis para todo subprograma.

**Tipos suficientes y listas tipadas:** El lenguaje trae cadenas, booleanos, nulo y error como valores nativos. `taza tipo` es recursivo, así que `taza taza grano` permite matrices. Como las listas son homogéneas y los parámetros llevan tipo (`taza grano inventario`), la firma de una función ya documenta lo que espera recibir.

**Identidad temática coherente:** La metáfora se aplica de forma sistemática: `barista` (quien prepara) es la función, `degustar` es el retorno, `servir` es la salida, `chorreando` y `recolar` son los ciclos, y `hierve`/`enfria` es la bifurcación. Esto reduce la carga de aprendizaje, porque una vez entendida la metáfora la mayoría de las palabras reservadas se deducen (ver 3.3).

### 3.2 Elementos débiles

**Las funciones no declaran tipo de retorno:** En el Ejercicio 5, `auditarInventario` devuelve un `cafe`, pero nada en su firma lo dice:

```
barista auditarInventario(taza grano inventario) { ... degustar fuerte ; }
```

Quien la llama solo puede saber qué devuelve leyendo el cuerpo, y el analizador semántico debe inferirlo a partir de los `degustar`. Tampoco hay forma de marcar una función que no devuelve nada.

**Las listas no se pueden indexar ni medir:** Las únicas operaciones son `insertar`, `extraer` y `buscar`:

```
granoMolido primero = menu[0] ;   # no existe: factor no admite indexacion
grano n = menu.largo() ;          # no existe: no hay operacion de longitud
```

Leer un elemento sin sacarlo de la lista solo es posible dentro de `recolar`, y no hay forma de modificar una posición. Además, `buscar` devuelve un booleano, así que no dice dónde está el elemento.

**Los números son limitados:** `numero_entero` es solo `digito { digito }` y `factor` no admite un `-` unario, por lo que `grano deuda = -5 ;` no es válido y hay que escribir `0 - 5`. Tampoco existen `.5` ni notación científica.

**El control de flujo es mínimo:** No hay `else if`, así que una cadena de casos debe anidarse dentro de `enfria`:

```
hierve (temperatura < 50) {
    servir("Frio") ;
} enfria {
    hierve (temperatura < 90) {
        servir("Tibio") ;
    } enfria {
        servir("Listo") ;
    }
}
```

Tampoco hay `for` con contador, `break` ni `continue`. Repetir N veces exige una variable auxiliar y `chorreando`, como en el Ejercicio 3.

**Las cadenas y la salida son rígidas.** `caracter_cadena` excluye las comillas dobles y el salto de línea, y no hay secuencias de escape, así que `servir("Dijo \"hola\"") ;` no es válido. `servir` recibe una sola expresión, por lo que todo se arma con `+`, y no se especifica qué pasa al concatenar un `granoMolido` con un `poso`. Además, los identificadores admiten solo letras ASCII, lo que impide usar `ñ` o tildes en un lenguaje pensado en español.

**El tipo de error no tiene mecanismo asociado:** `SeQuemoElCafe` existe como valor, pero nada en la gramática permite generarlo, capturarlo o tratarlo distinto de otro valor. No hay `try`/`catch` ni una comprobación equivalente.

#### Inconsistencias en la gramática actual

1. **Las palabras reservadas no están enumeradas.** La gramática dice que no pueden usarse como identificadores, pero no las lista, y varios terminales cumplen también la regla de `identificador`. `barista chorreador() { }` deriva de `declaracion_funcion` y de `declaracion_chorreador`, y `x = leer() ;` deriva de `llamada_leer` y de `asignacion`. Con la extensión cuántica pasa lo mismo: `H(a) ;` deriva de `llamada_funcion` y de `compuerta_unaria`, y `medir(a)` de `llamada_funcion` y de `medicion_cuantica`. Además, si `H`, `X` y `Z` son reservadas, dejan de estar disponibles como nombres de variables de una letra.
2. **Varias alternativas comparten prefijo.** `declaracion_funcion` y `declaracion_chorreador` empiezan con `barista`, y en `sentencia` y `factor` varias alternativas empiezan con `identificador` (asignación, llamada, operación de lista, y en `sentencia` también `llamada_leer`). Un analizador LL(1) necesita factorizar por la izquierda o mirar más de un token para decidir entre ellas.

### 3.3 Elementos chistosos

El humor del lenguaje viene de que cada construcción tiene una lectura literal de cafetería, y esa lectura casi siempre tiene sentido técnico.

**Los tipos siguen la lógica del grano:** `grano` es un entero porque el grano se cuenta de uno en uno, y `taza` es un arreglo porque es lo que contiene cosas. `descafeinado` es el nulo ideal: sigue siendo café, pero le falta justo lo que lo hace café, igual que un `null` es un valor sin contenido.

**El humor es local:** `chorreador` es el colador tradicional de café costarricense, y con eso el punto de entrada es literalmente el aparato por donde pasa todo el programa. `ralo` es como se le dice al café aguado, y por eso es el `false` de un booleano llamado `cafe`, cuyo `true` es `fuerte`. `SeQuemoElCafe` es el error, y lo gracioso es que, como en la vida real, una vez quemado no hay forma de deshacerlo: es un valor que la gramática no permite capturar (ver 3.2).

**El programa se lee como una escena de cafeteria:** El Ejercicio 3 recorre un ciclo `chorreando (temperatura < 90)` hasta que el agua llega al punto y luego pregunta con `hierve`. El Ejercicio 5 escribe `hierve (falta_cafe == fuerte ConAzucar total_granos < 50)` y responde con "Alerta critica: Hay que tostar mas cafe urgente". Un condicional sobre inventarios se ve como una emergencia de cafetería.

**El único intruso es `leer`:** Es la única palabra reservada fuera de la metáfora.

### 3.4 Comparación con otros lenguajes

*Tabla 1 – Decisiones puntuales frente a C y Java*

| Aspecto | C | Java | Quantum Coffee |
|---|---|---|---|
| Punto de entrada | `int main()` en cualquier posición | `main` dentro de una clase | `barista chorreador()`, sin parámetros ni retorno, siempre al final |
| Tipo de retorno | Antes del nombre | Antes del nombre | No se declara |
| Booleanos | `int` (o `stdbool.h`) | `boolean` | `cafe` con `fuerte`/`ralo` |
| Nulo y errores | `NULL`, `errno` | `null`, excepciones | `descafeinado`, `SeQuemoElCafe` (valores sin captura) |
| Asignación en condición | Permitida (`if (x = 5)`) | Solo con `boolean` | Imposible (es sentencia) |
| Else-if | `else if` | `else if` | No existe, se anida en `enfria` |
| Recorrer una lista | `for` con índice | `for (T x : lista)` | `recolar (T x en lista)` |
| Variables globales | Nivel de archivo, sin marca | `static` en una clase | `cafeteria` |

**Listas y ciclos:** `taza` se comporta como el `ArrayList` de Java, donde `insertar` corresponde a `add`, `extraer(i)` a `remove(i)` y `buscar` a `contains`. Pero le faltan `get` y `size`, que son lo que hace usable a `ArrayList`. `recolar` mezcla dos estilos: la palabra `en` de Python (`for x in lista`) con el tipo explícito de Java (`for (String x : lista)`).

**Nulo y error como valores:** `descafeinado` se parece al `null` de Java, y si puede asignarse a cualquier tipo (la especificación no lo aclara) hereda los mismos problemas de referencias nulas. `SeQuemoElCafe` se acerca al tipo `error` de Go, que también es un valor, pero aquí no hay retorno múltiple ni forma de comprobarlo, que es lo que hace útil el patrón en Go.

**Ensamblador:**. El ciclo del Ejercicio 3 se reduce a una comparación y dos saltos:

```asm
        mov  eax, 20        ; temperatura = 20
ciclo:  cmp  eax, 90        ; temperatura < 90 ?
        jge  fin
        add  eax, 10        ; temperatura = temperatura + 10
        jmp  ciclo
fin:
```

En ASM no existen bloques, tipos ni precedencia. Todo eso lo aporta el lenguaje, y por eso la gramática debe ser más precisa en un lenguaje de alto nivel.
