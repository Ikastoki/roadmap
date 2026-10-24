# POO

La programacion orientada a objetos (POO) es un paradigma de programacion que organiza el software en torno a objetos en lugar de procedimientos o funciones sueltas. Mientras que la programacion procedural estructura el codigo como una serie de instrucciones secuenciales que operan sobre datos, la POO encapsula tanto los datos como el comportamiento dentro de entidades autonomas llamadas objetos. Este enfoque permite modelar sistemas del mundo real de forma mas natural, ya que cada objeto representa una entidad con estado propio y capacidad de actuar de manera independiente.

Java es un lenguaje puramente orientado a objetos: todo el codigo reside dentro de clases, incluso el punto de entrada `main` es un metodo estatico de una clase. Esta decision de diseno convierte a Java en un lenguaje donde la POO no es opcional sino la forma fundamental de escribir codigo.

## Los cuatro principios fundamentales

La POO se sustenta en cuatro pilares que, trabajando en conjunto, permiten construir sistemas modulares, reutilizables y faciles de mantener.

### Encapsulacion

La encapsulacion consiste en ocultar los detalles internos de un objeto y exponer unicamente lo necesario a traves de una interfaz publica. Un objeto controla su propio estado interno, impidiendo que otras partes del codigo modifiquen sus campos directamente. En lugar de acceder a un campo `saldo` de una cuenta bancaria, se invoca un metodo `retirar(monto)` que valida internamente si la operacion es posible.

Este principio reduce el acoplamiento entre componentes: si la representacion interna de un objeto cambia, solo su propio codigo necesita modificarse, mientras que los clientes que interactuan con el permanecen intactos.

### Abstraccion

La abstraccion consiste en modelar un objeto mostrando unicamente las propiedades y comportamientos relevantes para el contexto, ocultando la complejidad innecesaria. Un conductor de coche interactua con el volante, los pedales y el cambio, sin necesidad de comprender el funcionamiento interno del motor de combustion.

En Java, la abstraccion se implementa mediante clases abstractas e interfaces, que definen contratos sin especificar la implementacion completa. Esto permite trabajar a nivel de concepto: un `Pago` puede ser abstracto, mientras que `PagoConTarjeta` y `PagoEnEfectivo` concretizan ese concepto.

### Herencia

La herencia permite crear nuevas clases basadas en clases existentes, reutilizando codigo y estableciendo relaciones jerarquicas de tipo. Una subclase hereda los campos y metodos de su superclase, pudiendo añadir nuevos comportamientos o modificar los existentes.

```java
public class Animal {
    protected String nombre;

    public void respirar() {
        System.out.println("Respirando...");
    }
}

public class Perro extends Animal {
    public void ladrar() {
        System.out.println("Guau!");
    }
}
```

La herencia modela relaciones "es-un": un `Perro` es un `Animal`. Esto permite polimorfismo y reutilizacion de codigo comun.

### Polimorfismo

El polimorfismo permite que un mismo metodo se comporte de manera diferente segun el tipo de objeto que lo invoque. Existen dos formas principales: polimorfismo de sobrecarga (varios metodos con el mismo nombre pero distintos parametros) y polimorfismo de sobrescritura (una subclase redefine el comportamiento de un metodo de su superclase).

```java
public class Figura {
    public double calcularArea() {
        return 0;
    }
}

public class Circulo extends Figura {
    private double radio;

    @Override
    public double calcularArea() {
        return Math.PI * radio * radio;
    }
}

public class Rectangulo extends Figura {
    private double ancho, alto;

    @Override
    public double calcularArea() {
        return ancho * alto;
    }
}
```

Un `Figura` puede referenciar tanto un `Circulo` como un `Rectangulo`, y al invocar `calcularArea()` se ejecutara la implementacion correspondiente al tipo real del objeto.

## Relacion entre los principios

Los cuatro principios no operan de forma aislada. La encapsulacion protege el estado del objeto; la abstraccion define que partes de ese estado son visibles; la herencia permite reutilizar y especializar comportamientos; el polimorfismo hace que esos comportamientos se adapten dinamicamente al tipo concreto. Juntos, forman un sistema donde la complejidad se gestiona por capas.

## POO vs programacion procedural

| Aspecto | Procedural | POO |
|---|---|---|
| Organizacion | Funciones y datos separados | Objetos que encapsulan datos y comportamiento |
| Reutilizacion | Copia de funciones | Herencia y composicion |
| Mantenimiento | Cambios afectan multiples archivos | Cambios localizados en la clase |
| Modelado | Basado en algoritmos | Basado en entidades del mundo real |
| Seguridad de datos | Datos expuestos globalmente | Acceso controlado via metodos |

La programacion procedural resulta adecuada para scripts simples y tareas lineales, pero a medida que el sistema crece, la falta de estructura genera codigo fragil, dificil de depurar y propenso a efectos colaterales. La POO mitiga estos problemas al imponer reglas claras de interaccion entre componentes.

## Clases vs objetos

Una **clase** es la definicion, el plano o molde que describe que campos tiene un objeto y que comportamientos puede ejecutar. Es una entidad estatica que existe en el codigo fuente.

Un **objeto** es una instancia concreta de una clase, creada en tiempo de ejecucion con valores reales en sus campos. Si la clase es el plano de una casa, el objeto es la casa construida.

```java
// Definicion de clase
public class Coche {
    String marca;
    int anio;

    void arrancar() {
        System.out.println("El coche arranca.");
    }
}

// Creacion de objetos (instanciacion)
Coche c1 = new Coche();  // Objeto 1
c1.marca = "Toyota";
c1.anio = 2022;

Coche c2 = new Coche();  // Objeto 2
c2.marca = "Honda";
c2.anio = 2020;
```

`c1` y `c2` son dos objetos distintos de la misma clase `Coche`. Cada uno tiene su propio estado (`marca` y `anio` con valores diferentes) pero comparten el mismo comportamiento (`arrancar`). El operador `new` reserva memoria en el heap para el objeto y devuelve una referencia que permite acceder a el.

Una clase puede existir sin que nunca se cree un objeto de ella, pero un objeto siempre requiere una clase que lo defina. La relacion es unidireccional: la clase no conoce a sus objetos, pero los objetos conocen su clase.
