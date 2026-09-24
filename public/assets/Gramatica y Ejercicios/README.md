# 1. Tipos de datos
tipo ::= "grano"           (* int *)
       | "poso"            (* float *)
       | "granoMolido"     (* string *)
       | "taza" tipo       (* array de un tipo *)
       | "cafe"            (* boolean *)
       | "fuerte"          (* true *)
       | "ralo"            (* false *)
       | "descafeinado"    (* null *)
       | "SeQuemoElCafe"   (* error *)

# 2. Estructura del Programa
programa               ::= { declaracion_global | declaracion_funcion } declaracion_chorreador
declaracion_global     ::= "cafeteria" tipo identificador [ "=" expresion ] ";"
declaracion_chorreador ::= "chorreador" "(" ")" bloque

# 3. Funciones
declaracion_funcion    ::= "barista" identificador "(" [ parametros ] ")" bloque
parametros             ::= parametro { "," parametro }
parametro              ::= tipo identificador
llamada_funcion        ::= identificador "(" [ argumentos ] ")"
argumentos             ::= expresion { "," expresion }

# 4. Bloques y Sentencias
bloque                 ::= "{" { sentencia } "}"

sentencia              ::= declaracion_var_local
                         | asignacion
                         | estructura_condicional
                         | bucle_chorreando
                         | bucle_recolar
                         | llamada_servir
                         | llamada_funcion ";"
                         | operacion_lista ";"
                         | retorno_degustar

declaracion_var_local  ::= tipo identificador [ "=" expresion ] ";"
asignacion             ::= identificador "=" expresion ";"
llamada_servir         ::= "servir" "(" expresion ")" ";"
retorno_degustar       ::= "degustar" ( expresion | "descafeinado" | "SeQuemoElCafe" ) ";"

# 5. Estructuras de control
estructura_condicional ::= "hierve" "(" expresion ")" bloque [ "enfria" bloque ]
bucle_chorreando       ::= "chorreando" "(" expresion ")" bloque
bucle_recolar          ::= "recolar" "(" tipo identificador "en" identificador ")" bloque

# 6. Operadores lógicos
op_or                  ::= "SinAzucar"
op_and                 ::= "ConAzucar"
op_not                 ::= "Amargo"
op_relacional          ::= "==" | "!=" | "<" | "<=" | ">" | ">="

# 7. Jerarquía de Expresiones (Precedencia)
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
                         | "(" expresion ")"

operacion_lista        ::= identificador "." ( "insertar(" expresion ")" 
                                             | "extraer(" expresion ")" 
                                             | "buscar(" expresion ")" )

# 8. Literales y Terminales
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


# Ejercicios

# 1. Ejercicios de numeros

```bash
barista chorreador() {
    grano sacos_bodega = 15 ;
    poso peso_por_saco = 45.5 ;
    poso peso_total = sacos_bodega * peso_por_saco ;
    
    servir(peso_total) ;
}
```
# 2. Ejercicio de textos
```bash
barista chorreador() {
    granoMolido cliente = "Don Juan" ;
    granoMolido pedido = "Capuchino doble" ;
    
    servir("Orden lista para: " + cliente) ;
    servir("Bebida: " + pedido) ;
}
}
```
# 3. Ejercicio de control de flujo
```bash
# Uso de repeticion (chorreando) y bifurcacion (hierve/enfria)
barista chorreador() {
    grano temperatura = 20 ;
    
    chorreando (temperatura < 90) {
        temperatura = temperatura + 10 ;
    }
    
    hierve (temperatura >= 90) {
        servir("Agua al punto ideal para chorrear") ;
    } enfria {
        servir("Falta fuego, el agua esta fria") ;
    }
}
```

# 4. Ejercicio de listas
```bash
# Manejo de arreglos e invocacion de metodos nativos
barista chorreador() {
    taza granoMolido menu = ["Espresso", "Americano"] ;
    
    menu.insertar("Latte") ;
    cafe hay_latte = menu.buscar("Latte") ;
    granoMolido servido = menu.extraer(0) ;
    
    servir("Bebida servida al cliente:") ;
    servir(servido) ;
    
    servir("Restante en el menu:") ;
    recolar (granoMolido bebida en menu) {
        servir(bebida) ;
    }
}
```

# 5. Ejercicio integrado
```bash
# Uniendo tipos, funciones, booleanos, ciclos y condicionales complejos
barista auditarInventario(taza grano inventario) {
    grano total_granos = 0 ;
    cafe falta_cafe = ralo ;

    recolar (grano cantidad en inventario) {
        hierve (cantidad <= 5) {
            falta_cafe = fuerte ;
        }
        total_granos = total_granos + cantidad ;
    }

    hierve (falta_cafe == fuerte ConAzucar total_granos < 50) {
        servir("Alerta critica: Hay que tostar mas cafe urgente") ;
        degustar ralo ;
    } enfria {
        servir("Inventario de la cafeteria saludable") ;
        degustar fuerte ;
    }
}

barista chorreador() {
    taza grano bodega = [10, 4, 20, 15] ;
    cafe estado_tienda = auditarInventario(bodega) ;
}
```