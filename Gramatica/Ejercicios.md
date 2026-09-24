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