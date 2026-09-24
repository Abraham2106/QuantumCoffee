# 1. Tipos de datos
```bash
tipo ::= "grano"           (* int *)
       | "poso"            (* float *)
       | "granoMolido"     (* string *)
       | "taza" tipo       (* array de un tipo *)
       | "cafe"            (* boolean *)
```

# 2. Estructura del Programa
```bash
programa               ::= { declaracion_global | declaracion_funcion } declaracion_chorreador
declaracion_global     ::= "cafeteria" tipo identificador [ "=" expresion ] ";"
declaracion_chorreador ::= "barista" "chorreador" "(" ")" bloque
```

# 3. Funciones
```bash
declaracion_funcion    ::= "barista" identificador "(" [ parametros ] ")" bloque
parametros             ::= parametro { "," parametro }
parametro              ::= tipo identificador
llamada_funcion        ::= identificador "(" [ argumentos ] ")"
argumentos             ::= expresion { "," expresion }
```

# 4. Bloques y Sentencias
```bash
bloque                 ::= "{" { sentencia } "}"

sentencia              ::= declaracion_var_local
                         | declaracion_qubit
                         | asignacion
                         | estructura_condicional
                         | bucle_chorreando
                         | bucle_recolar
                         | llamada_leer
                         | llamada_servir
                         | llamada_funcion ";"
                         | operacion_lista ";"
                         | operacion_cuantica ";"
                         | medicion_cuantica ";"
                         | retorno_degustar

declaracion_var_local  ::= tipo identificador [ "=" expresion ] ";"
asignacion             ::= identificador "=" expresion ";"
llamada_servir         ::= "servir" "(" expresion ")" ";"
llamada_leer           ::= identificador "=" "leer" "(" ")" ";"
retorno_degustar       ::= "degustar" expresion ";"
```

# 5. Estructuras de control
```bash
estructura_condicional ::= "hierve" "(" expresion ")" bloque [ "enfria" bloque ]
bucle_chorreando       ::= "chorreando" "(" expresion ")" bloque
bucle_recolar          ::= "recolar" "(" tipo identificador "en" identificador ")" bloque
```

# 6. Operadores lógicos
```bash
op_or                  ::= "SinAzucar"
op_and                 ::= "ConAzucar"
op_not                 ::= "Amargo"
op_relacional          ::= "==" | "!=" | "<" | "<=" | ">" | ">="
```

# 7. Jerarquía de Expresiones (Precedencia)
```bash
expresion              ::= expresion_or
expresion_or           ::= expresion_and { op_or expresion_and }
expresion_and          ::= expresion_not { op_and expresion_not }
expresion_not          ::= [ op_not ] expresion_relacional
expresion_relacional   ::= expresion_aritmetica [ op_relacional expresion_aritmetica ]
expresion_aritmetica   ::= termino { ( "+" | "-" ) termino }
termino                ::= factor { ( "*" | "/" | "%" ) factor }

factor                 ::= literal
                         | identificador
                         | llamada_funcion
                         | operacion_lista
                         | medicion_cuantica
                         | "(" expresion ")"

operacion_lista        ::= identificador "." ( "insertar" "(" expresion ")"
                                             | "extraer" "(" expresion ")"
                                             | "buscar" "(" expresion ")" )
```

# 8. Literales y Terminales
```bash
literal                ::= numero_entero
                         | numero_decimal
                         | cadena
                         | "fuerte"
                         | "ralo"
                         | "descafeinado"
                         | "SeQuemoElCafe"
                         | arreglo

arreglo                ::= "[" [ expresion { "," expresion } ] "]"

identificador          ::= letra { letra | digito | "_" }
numero_entero          ::= digito { digito }
numero_decimal         ::= digito { digito } "." digito { digito }
cadena                 ::= '"' { caracter_cadena } '"'

letra                  ::= "A".."Z" | "a".."z"
digito                 ::= "0".."9"
caracter_cadena        ::= ? cualquier caracter excepto comillas dobles y salto de linea ?
comentario             ::= "#" { ? cualquier caracter excepto salto de linea ? }
```

# 9. Extensión cuántica
```bash
declaracion_qubit      ::= "qubit" identificador ";"

operacion_cuantica     ::= compuerta_unaria
                         | compuerta_binaria

compuerta_unaria       ::= ( "H" | "X" | "Z" ) "(" identificador ")"

compuerta_binaria      ::= "CNOT" "(" identificador "," identificador ")"

medicion_cuantica      ::= "medir" "(" identificador ")"
```
