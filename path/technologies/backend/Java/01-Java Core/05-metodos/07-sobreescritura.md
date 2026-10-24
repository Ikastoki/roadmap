# Sobreescritura

La sobrescritura (overriding) permite a una subclase proporcionar una implementacion especifica de un metodo que ya esta definido en su superclase. Es el mecanismo fundamental del polimorfismo dinamico en Java: la decision de que metodo ejecutar se toma en tiempo de ejecucion, no en compilacion.

## Misma firma en subclase

Para sobrescribir un metodo, la subclase debe declarar un metodo con exactamente la misma firma: mismo nombre, misma lista de parametros (en tipo y orden), y mismo tipo de retorno o un tipo covariante.

```java
public class Animal {
    public void emitirSonido() {
        System.out.println("Sonido generico");
    }
}

public class Perro extends Animal {
    @Override
    public void emitirSonido() {
        System.out.println("Guau guau");
    }
}

public class Gato extends Animal {
    @Override
    public void emitirSonido() {
        System.out.println("Miau");
    }
}
```

La sobrescritura solo aplica a metodos de instancia. Los metodos estaticos se ocultan (hide), no se sobrescriben. Los metodos privados no son visibles en la subclase y por tanto no pueden sobrescribirse.

## La anotacion @Override

La anotacion `@Override` le indica al compilador que el metodo anotado intenta sobrescribir un metodo de la superclase. Si el metodo no sobrescribe efectivamente ningun metodo (por error en la firma, por ejemplo), el compilador produce un error.

```java
public class Base {
    public void mostrar() { }
}

public class Derivada extends Base {
    @Override
    public void mostrar() { } // correcto

    @Override
    public void mostrar(int x) { } // ERROR: no sobrescribe nada
}
```

> Siempre debe usarse `@Override` al sobrescribir metodos. No es obligatorio, pero es una practica universalmente aceptada porque atrapa errores tipograficos sutiles, como escribir `equals(Object o)` cuando se pretendia sobrescribir `equals(Object o)`.

## Resolucion en tiempo de ejecucion (dynamic dispatch)

El polimorfismo dinamico funciona mediante tablas de metodos virtuales (vtable). Cuando se invoca un metodo sobrescrito sobre una referencia, la JVM consulta la tabla de la clase real del objeto (no la del tipo de la referencia) para determinar que implementacion ejecutar.

```java
Animal mascota = new Perro();
mascota.emitirSonido(); // "Guau guau" (en runtime)
```

El tipo de la variable `mascota` es `Animal`, pero el objeto es `Perro`. El compilador verifica que `Animal` tiene el metodo `emitirSonido()`. En runtime, la JVM busca la implementacion en la clase `Perro` y ejecuta esa version.

```java
void probarAnimal(Animal a) {
    a.emitirSonido(); // ejecuta la implementacion de la clase real
}

probarAnimal(new Perro()); // Guau guau
probarAnimal(new Gato());  // Miau
probarAnimal(new Animal());// Sonido generico
```

## Reglas de la sobrescritura

El compilador impone varias reglas para garantizar que la sobrescritura sea coherente con el contrato de la superclase.

**Retorno covariante.** El tipo de retorno en la subclase puede ser el mismo o un subtipo (covariante) del tipo de retorno declarado en la superclase.

```java
public class Base {
    public Number obtenerValor() { return 0; }
}

public class Derivada extends Base {
    @Override
    public Integer obtenerValor() { return 42; } // Integer extiende Number
}
```

**No menos visibilidad.** El metodo sobrescrito no puede ser menos accesible que el metodo original. Un metodo `protected` en la superclase puede volverse `public` en la subclase, pero no `private` o `default`.

```java
public class Base {
    protected void metodo() { }
}

public class Derivada extends Base {
    @Override
    public void metodo() { } // valido: mas visible

    // @Override
    // private void metodo() { } // ERROR: menos visible
}
```

**No mas excepciones.** El metodo sobrescrito no puede declarar excepciones comprobadas (checked) nuevas o mas amplias que las declaradas en el metodo original. Puede declarar menos o ninguna.

```java
import java.io.IOException;
import java.sql.SQLException;

public class Base {
    public void procesar() throws IOException { }
}

public class Derivada extends Base {
    @Override
    public void procesar() { } // valido: sin excepciones

    // @Override
    // public void procesar() throws SQLException { } // ERROR: nueva checked exception
}
```

**Metodos final no se sobrescriben.** Un metodo declarado como `final` en la superclase no puede ser sobrescrito.

**Metodos estaticos no se sobrescriben.** Si una subclase declara un metodo estatico con la misma firma que uno en la superclase, este se oculta (hide), no se sobrescribe. La resolucion se hace en compile-time segun el tipo de la referencia, no en runtime.

```java
public class Base {
    public static void saludo() {
        System.out.println("Hola desde Base");
    }
}

public class Derivada extends Base {
    public static void saludo() {
        System.out.println("Hola desde Derivada");
    }
}

Base.saludo();    // "Hola desde Base"
Derivada.saludo();// "Hola desde Derivada"
Base b = new Derivada();
b.saludo();       // "Hola desde Base" (resolucion estatica!)
```
