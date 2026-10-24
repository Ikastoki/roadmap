# Modificadores de acceso en POO

Los modificadores de acceso (`public`, `protected`, `default`, `private`) adquieren una dimension adicional cuando se analizan en el contexto de la programacion orientada a objetos, especialmente en relacion con la herencia y el polimorfismo. La visibilidad de los miembros determina que puede heredarse, que puede sobrescribirse y como se accede a los componentes heredados desde distintas posiciones en la jerarquia.

## public en el contexto de herencia

Un miembro `public` en una superclase es accesible desde cualquier clase, incluyendo subclases en cualquier paquete. Los metodos publicos constituyen la API que las subclases pueden sobrescribir.

```java
package com.ejemplo.base;

public class Vehiculo {
    public void acelerar() {
        System.out.println("Acelerando...");
    }
}
```

```java
package com.ejemplo.derivado;
import com.ejemplo.base.Vehiculo;

public class Coche extends Vehiculo {
    @Override
    public void acelerar() {
        System.out.println("Coche acelerando...");
    }
}

public class Taller {
    public void probar(Vehiculo v) {
        v.acelerar(); // accesible: public
    }
}
```

Un metodo `public` sobrescrito debe seguir siendo `public` (no puede reducir la visibilidad). La regla es que la visibilidad solo puede ampliarse, nunca restringirse.

## protected en el contexto de herencia

`protected` es el modificador disenado especificamente para la herencia. Permite que las subclases (incluso en otros paquetes) accedan a miembros que no forman parte de la API publica.

```java
package com.ejemplo.base;

public class Animal {
    protected String especie;
    protected void respirar() {
        System.out.println("Respirando...");
    }
}
```

```java
package com.ejemplo.derivado;
import com.ejemplo.base.Animal;

public class Perro extends Animal {
    public void mostrar() {
        System.out.println(especie); // accesible: protected
        respirar();                  // accesible: protected
    }

    public void test(Animal a) {
        // a.especie;     // ERROR: no accesible via referencia de superclase
        // a.respirar();  // ERROR: solo accesible via this o herencia directa
    }
}
```

Un matiz importante: una subclase en otro paquete solo puede acceder a miembros `protected` heredados a traves de la herencia (mediante `this` o directamente), no a traves de referencias de la superclase. Esto evita accesos indirectos desde el mismo paquete de la subclase a objetos de la superclase.

## Default (package-private) en herencia

El acceso default no permite el acceso desde subclases en otros paquetes. Es el nivel adecuado para metodos y campos que deben compartirse entre clases del mismo paquete pero que no forman parte del contrato de herencia.

```java
package com.ejemplo.base;

public class Base {
    String mensaje = "Hola"; // package-private
    void mostrar() {         // package-private
        System.out.println(mensaje);
    }
}
```

```java
package com.ejemplo.base;
// Mismo paquete

public class Derivada extends Base {
    public void test() {
        System.out.println(mensaje); // accesible: mismo paquete
        mostrar();                   // accesible: mismo paquete
    }
}
```

```java
package com.ejemplo.otro;
import com.ejemplo.base.Base;

public class Lejos extends Base {
    public void test() {
        // System.out.println(mensaje); // ERROR: default, otro paquete
        // mostrar();                   // ERROR: default, otro paquete
    }
}
```

## private en el contexto de herencia

Los miembros `private` no son visibles para las subclases. Sin embargo, existen en el objeto y ocupan memoria. La subclase no puede acceder a ellos directamente, pero puede hacerlo indirectamente a traves de metodos publicos o protected de la superclase.

```java
public class Cuenta {
    private double saldo; // no accesible desde subclases

    public double getSaldo() {
        return saldo;
    }

    protected void setSaldo(double saldo) {
        this.saldo = saldo;
    }
}

public class CuentaPremium extends Cuenta {
    public void aplicarBonus() {
        // saldo += 100; // ERROR: saldo es private
        setSaldo(getSaldo() + 100); // correcto: via metodos heredados
    }
}
```

El hecho de que los campos privados no sean accesibles desde subclases refuerza el encapsulamiento: la implementacion interna de la superclase puede cambiar sin afectar a las subclases, siempre que los metodos publicos/protegidos mantengan su contrato.

## Visibilidad entre clases del mismo paquete vs subclases de otro paquete

La tabla completa de visibilidad en funcion de la ubicacion del invocante:

| Modificador  | Misma clase | Mismo paquete | Subclase otro paquete | Mundo exterior |
|--------------|-------------|---------------|----------------------|-----------------|
| `private`      | Si          | No            | No                   | No              |
| `default`      | Si          | Si            | No                   | No              |
| `protected`    | Si          | Si            | Si                   | No              |
| `public`       | Si          | Si            | Si                   | Si              |

**Mismo paquete:** `default` y `protected` son equivalentes en visibilidad. Ambos permiten acceso a cualquier clase del paquete. La diferencia solo se manifiesta cuando interviene una subclase de otro paquete.

**Subclase en otro paquete:** `protected` permite acceso via herencia (a traves de `this`), pero no via referencia de la superclase. `default` no permite acceso en absoluto.

```java
package com.ejemplo.base;

public class Base {
    protected int protegido;
    int paquete;
}
```

```java
package com.ejemplo.derivado;
import com.ejemplo.base.Base;

public class Hija extends Base {
    public void test() {
        protegido = 1;   // OK: via herencia
        // paquete = 1;  // ERROR: default no visible
    }

    public void test2(Base b) {
        // b.protegido = 1; // ERROR: no accesible via referencia
    }

    public void test3(Hija h) {
        h.protegido = 1;   // OK: misma clase concreta
    }
}
```

Esta distincion sutil tiene implicaciones practicas al disenar jerarquias. `protected` debe usarse con cuidado, ya que cualquier subclase en cualquier paquete puede acceder al miembro, lo que puede comprometer el encapsulamiento si no se controla quien extiende la clase.
