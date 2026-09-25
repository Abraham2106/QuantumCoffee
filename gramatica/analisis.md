## 3. Análisis del lenguaje

### 3.1 Elementos sólidos

Jerarquía de expresiones: La prioridad de los operadores se define directamente en la gramática siguiendo este orden:

`expresion_or > expresion_and > expresion_not > expresion_relacional > expresion_aritmetica > termino > factor`

En cada nivel se usan repeticiones `{ ... }` en vez de recursión izquierda. Esto facilita convertir cada producción en una función dentro de un analizador descendente recursivo. Además, no hace falta utilizar una tabla externa para definir la prioridad de los operadores. Por ejemplo, en `2 + 3 * 4`, primero se resuelve `3 * 4` y después se suma 2, porque `*` se encuentra en un nivel más profundo que `+`.

Los bloques con llaves evitan el problema del "else colgante": En lenguajes como C puede existir confusión sobre a cuál `if` pertenece un `else` cuando hay varias condiciones seguidas. En Quantum Coffee, `hierve` y `enfria` siempre utilizan bloques con llaves, por lo que es más fácil ver qué instrucciones pertenecen a cada condición.

```text
hierve (a) {
    hierve (b) { servir("a y b"); }
} enfria {
    servir("no a");
}
```

La asignación como una sentencia: El símbolo `=` solamente se utiliza al declarar una variable o al hacer una asignación. Dentro de una condición se utiliza `==`. Esto evita errores comunes como escribir una asignación dentro de una condición por accidente.

```text
hierve (temperatura = 90) { ... }   # rechazado: "=" no pertenece a expresion
```

El alcance de las variables: Las variables globales utilizan la palabra `cafeteria`, mientras que las variables locales no necesitan un prefijo. Gracias a esto, al leer el código se puede reconocer con mayor facilidad si una variable es global o local. También ayuda a evitar que se creen variables globales por accidente. Además, `chorreador` se declara utilizando `barista`, igual que las demás funciones, manteniendo una forma similar para declarar los subprogramas.

Tipos básicos y listas con un tipo definido: El lenguaje incluye cadenas, booleanos, nulo y error como valores propios. También permite usar `taza` varias veces, por ejemplo `taza taza grano`, lo que permite representar estructuras como matrices. Como las listas mantienen un mismo tipo de datos y los parámetros indican el tipo que reciben, una declaración como `taza grano inventario` permite entender mejor qué clase de información espera una función.

Temática: Las palabras reservadas siguen la temática de una cafetería. Por ejemplo, `barista` se utiliza para las funciones, `degustar` para retornar un valor, `servir` para mostrar una salida, `chorreando` y `recolar` para los ciclos, y `hierve` junto con `enfria` para las condiciones. Al mantener esta misma idea a lo largo del lenguaje, es más sencillo relacionar cada palabra con lo que hace una vez que se entiende la temática general.

### 3.2 Elementos débiles

Las funciones no dicen directamente qué tipo de dato devuelven: En el Ejercicio 5, por ejemplo, `auditarInventario` devuelve un `cafe`, pero eso no aparece en la firma:

```text
barista auditarInventario(taza grano inventario) { ... degustar fuerte ; }
```

Entonces, para saber qué devuelve la función, toca revisar el cuerpo y ver qué aparece en los `degustar`. Esto también hace que el analizador semántico tenga que averiguarlo por su cuenta. Además, tampoco existe una forma de indicar claramente que una función no devuelve nada.

Las listas se quedan un poco cortas en operaciones: Por ahora solo permiten `insertar`, `extraer` y `buscar`. No se puede entrar directamente a una posición ni preguntar cuántos elementos tiene la lista. Por ejemplo:

```text
granoMolido primero = menu[0] ; # no existe: no se puede acceder por posición
grano n = menu.largo() ;        # no existe: no hay una operación para obtener el tamaño
```

Si se quiere ver un elemento sin sacarlo de la lista, básicamente hay que hacerlo dentro de `recolar`. Tampoco se puede cambiar directamente lo que está guardado en una posición. Y `buscar` solamente responde si el elemento está o no, pero no dice en qué posición aparece.

Los números también tienen algunas limitaciones: `numero_entero` está formado solo por dígitos y `factor` no acepta un `-` directamente delante de un número. Entonces algo como `grano deuda = -5 ;` no funciona y habría que escribir `0 - 5`. Tampoco se pueden escribir valores como `.5` ni usar notación científica, así que por ese lado el formato de números es bastante básico.

El control de flujo es bastante sencillo: No existe algo como `else if`, así que cuando hay varias condiciones toca ir metiendo un `hierve` dentro del `enfria` anterior:

```text
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

También hacen falta algunas instrucciones comunes como `for` con contador, `break` y `continue`. Si se quiere repetir algo una cantidad específica de veces, hay que crear una variable auxiliar y resolverlo usando `chorreando`, como pasa en el Ejercicio 3. Funciona, pero claramente se vuelve un poco más largo de lo necesario.

Las cadenas y la salida: `caracter_cadena` no permite usar comillas dobles dentro del texto ni saltos de línea, y tampoco hay secuencias de escape. Entonces algo como `servir("Dijo "hola"") ;` no sería válido. Además, `servir` recibe una sola expresión, por lo que si se quieren mostrar varias cosas hay que unirlas usando `+`. Tampoco queda definido qué debería pasar al mezclar tipos distintos, por ejemplo un `granoMolido` con un `poso`. Otro detalle curioso es que los identificadores solo aceptan letras ASCII, así que no se podrían usar `ñ` ni tildes, lo cual se siente un poco raro para un lenguaje que está pensado en español.

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
