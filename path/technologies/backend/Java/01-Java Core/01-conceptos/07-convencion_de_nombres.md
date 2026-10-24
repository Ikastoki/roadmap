# Convencion de nombres

Java establece convenciones de nomenclatura que, aunque no son obligatorias desde el punto de vista del compilador, constituyen un estandar ampliamente adoptado en la industria. Seguir estas convenciones mejora la legibilidad, mantiene la consistencia entre proyectos y facilita el trabajo en equipo.

## Clases y tipos

Los nombres de clases, interfaces, enums y registros utilizan PascalCase (tambien llamado UpperCamelCase): cada palabra comienza con mayuscula y el resto en minuscula.

```java
public class GestorDeConexiones { }     // clase
public interface Repositorio { }        // interfaz
public enum EstadoDelPedido { }         // enumeracion
public record DatosDelCliente { }       // registro (Java 16+)
public exception class ConexionFallida { }  // excepcion
```

Las clases que implementan interfaces suelen agregar sufijos descriptivos: `ArrayList` implementa `List`, `HashMap` implementa `Map`. Las excepciones se nombran con el sufijo `Exception`. Las clases abstractas pueden usar el prefijo `Abstract` o el sufijo `Base`.

Los nombres deben ser sustantivos o frases nominales, no verbos. Una clase representa un concepto, no una accion.

## Metodos

Los metodos utilizan camelCase: la primera palabra en minuscula y las siguientes con la primera letra en mayuscula. Los nombres deben ser verbos o frases verbales que describan la accion que realizan.

```java
public String obtenerNombre() { }
public void establecerEdad(int edad) { }
public boolean estaActivo() { }
public int calcularTotal() { }
public void enviarCorreoElectronico(String destino) { }
```

Los metodos que retornan un valor boolean suelen usar prefijos como `is`, `has`, `can`, `should`. Los metodos getter y setter se nombran con los prefijos `get` y `set` seguidos del nombre del campo (sin el prefijo `get`/`set` para boolean se usa `is` en lugar de `get`).

## Variables

Las variables locales y los campos de instancia utilizan camelCase, igual que los metodos pero sin ser verbos.

```java
int contadorDeElementos;
String nombreDelUsuario;
boolean hayResultadosDisponibles;
List<Map<String, Object>> listaDeRegistros;
```

Los nombres deben ser descriptivos y significativos. Se penalizan los nombres de una sola letra salvo en contextos donde su significado es obvio, como iteradores de bucle (`i`, `j`, `k`) o lambda expressions simples.

## Constantes

Las constantes de clase (campos `static final`) se escriben en UPPER_SNAKE_CASE: palabras separadas por guiones bajos, todas en mayuscula.

```java
public static final int MAXIMO_NUMERO_DE_INTENTOS = 3;
public static final String CODIGO_DE_PAIS_POR_DEFECTO = "ES";
public static final double IMPUESTO_SOBRE_VENTAS = 0.21;
```

> Las constantes de instancia (campos `final` sin `static`) no siguen esta convencion y mantienen el estilo camelCase normal, ya que su valor varia entre instancias.

## Paquetes

Los nombres de paquetes utilizan todo en minuscula, sin guiones ni subrayados. La convencion es usar nombres de dominio invertidos para evitar colisiones entre organizaciones.

```java
package com.empresa.modulo.submodulo;
package org.proyecto.utilidades;
package es.gobierno.agencia.datos;
```

Los nombres de paquetes deben ser singulares (no `com.empresa.modulos`, sino `com.empresa.modulo`). Se evitan los numeros al inicio y las palabras compuestas se escriben pegadas, no con separadores.

## Enums y sus constantes

Las constantes dentro de una enumeracion se escriben en UPPER_SNAKE_CASE, igual que las constantes de clase, ya que tecnicamente son constantes estaticas finales generadas por la JVM.

```java
public enum DiaDeLaSemana {
    LUNES,
    MARTES,
    MIERCOLES,
    JUEVES,
    VIERNES,
    SABADO,
    DOMINGO
}
```

Cuando una enumeracion tiene campos y metodos, cada constante puede configurarse de forma diferente a traves del constructor.

## Reglas formales de identificadores

Un identificador en Java debe cumplir las siguientes reglas sintacticas: debe comenzar con una letra (A-Z o a-z), el caracter de subrayado (`_`) o el signo de dolar (`$`). Los caracteres subsiguientes pueden ser letras, digitos (0-9), subrayados o dolares. No puede ser una palabra reservada del lenguaje.

```java
// Identificadores validos
String nombre;
int _contador;
double $precio;
String nombreDeUsuario123;

// Identificadores invalidos
int 123nombre;    // comienza con digito
String mi-nombre; // contiene guion
void class();     // es palabra reservada
```

> El uso del signo de dolar en identificadores esta reservado por convencion para los nombres generados internamente por el compilador y herramientas de bytecode. Los programadores no deben utilizarlo en codigo fuente.
