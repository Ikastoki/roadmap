# Glosario

Java define un conjunto de modificadores y terminologia que determinan el comportamiento de clases, metodos, campos y paquetes. Comprender estos conceptos es fundamental para interpretar cualquier codigo Java, ya que influyen en la visibilidad, el ciclo de vida, la mutabilidad y el diseño general del programa.

## Modificadores de acceso

Los modificadores de acceso controlan la visibilidad y accesibilidad de clases, metodos y campos. Java establece cuatro niveles de encapsulacion, organizados de mayor a menor visibilidad:

**public** otorga acceso total: cualquier clase, en cualquier paquete, puede acceder al miembro marcado con este modificador. Es el nivel menos restrictivo y se usa cuando se quiere exponer una API publica sin limitaciones.

**protected** permite acceso dentro del mismo paquete y ademas a las subclases de otros paquetes. Este modificador es especialmente relevante en contextos de herencia, donde se quiere que las clases hijas puedan acceder a miembros del padre sin hacerlos publicos a toda la aplicacion.

**default** (sin modificador explícito, tambien llamado "package-private") limita el acceso exclusivamente al paquete donde se encuentra declarado. Cualquier clase del mismo paquete puede acceder, pero clases de otros paquetes no.

**private** restringe el acceso exclusivamente a la misma clase. Es el modificador mas restrictivo y la base del principio de encapsulacion en POO.

```java
public class Ejemplo {
    public String campoPublico = "acceso total";
    protected String campoProtegido = "paquete y subclases";
    String campoDefault = "solo paquete";  // modificador por defecto
    private String campoPrivado = "solo esta clase";
}
```

> La regla fundamental es que una clase solo puede hacer referencia a un miembro si tiene acceso directo a el o si existe una cadena de herencia que lo permita. El compilador verifica estas restricciones en tiempo de compilacion.

## Modificadores de miembros

Ademas de los modificadores de acceso, Java ofrece modificadores que alteran el comportamiento intrinseco de los miembros:

**static** indica que un miembro pertenece a la clase misma y no a ninguna instancia particular. Los campos estaticos comparten un unico valor para todas las instancias de la clase. Los metodos estaticos pueden invocarse sin crear un objeto, accediendo directamente a traves del nombre de la clase. Los bloques de inicializacion estatica se ejecutan una unica vez cuando se carga la clase.

**final** impide la modificacion posterior. En campos, una vez asignado un valor, nunca puede cambiarse. En metodos, impide que las subclases lo sobreescriban (override). En clases, impide que sea extendida con herencia. La combinacion `static final` crea verdaderas constantes de clase cuyo valor se determina en tiempo de compilacion.

```java
public class EjemploFinal {
    static final double PI = 3.14159265;  // constante de clase
    final String id;                        // constante de instancia

    public EjemploFinal(String id) {
        this.id = id;  // solo puede asignarse en el constructor
    }

    final void metodoInmutable() {  // no puede sobreescribirse
        // ...
    }
}
```

**abstract** se aplica tanto a clases como a metodos. Una clase abstracta no puede instanciarse directamente; sirve como plantilla que las subclases deben completar. Un metodo abstracto carece de implementacion en la clase que lo declara, obligando a cada subclase concreta a proporcionar su propia version.

**transient** indica que un campo no debe serializarse. Cuando un objeto se convierte a un stream de bytes (por ejemplo, para guardarlo en disco o enviarlo por red), los campos marcados como transient se ignoran y mantienen su valor por defecto en el objeto deserializado.

**volatile** garantiza que un campo sea leido siempre desde la memoria principal y no desde la cache del nucleo del procesador. Es esencial en programacion concurrente, donde multiples hilos pueden acceder al mismo campo simultaneamente. Sin volatile, un hilo podria leer un valor desactualizado porque su nucleo de CPU mantiene una copia local sin reflejar las escrituras de otros hilos.

## Modificadores de clase

Las clases admiten un conjunto reducido de modificadores. Pueden ser `public` (accesible desde cualquier lugar), `abstract` (no instanciable, requiere subclase concreta) o `final` (no extensible). Las clases internas (inner classes) pueden ademas ser `static`, lo que elimina la referencia implicita a la instancia de la clase contenedora.

> Las clases no pueden ser `private` ni `protected` como miembros de otro nivel, excepto cuando se trata de clases internas (inner classes), donde estos modificadores tienen sentido particular.

## Terminologia esencial

**Clase:** Plantilla que define el comportamiento (metodos) y el estado (campos) de un tipo de objeto. Es la unidad basica de modelado en Java.

**Objeto:** Instancia concreta de una clase, creada en el heap con el operador `new`. Cada objeto tiene su propio estado independiente.

**Interfaz:** Contrato que define un conjunto de metodos que una clase debe implementar. Desde Java 8 puede incluir metodos por defecto (con cuerpo) y metodos estaticos.

**Enumeracion (enum):** Tipo especial que define un conjunto finito de constantes nombradas. Internamente es una clase que extiende `java.lang.Enum` y puede incluir campos, constructores y metodos.

**Paquete:** Agrupacion logica de clases y interfaces que proporciona namespace y encapsulacion a nivel de paquete.

**Herencia:** Mecanismo por el cual una subclase adquiere los campos y metodos de su superclase. Java soporta herencia simple de clases (un solo padre) e implementacion multiple de interfaces.

**Polimorfismo:** Capacidad de tratar objetos de diferentes tipos como si fueran del tipo de su clase padre, invocando la implementacion correcta en tiempo de ejecucion (dynamic binding).
