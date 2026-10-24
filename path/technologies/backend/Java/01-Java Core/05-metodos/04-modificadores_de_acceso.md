# Modificadores de acceso

Los modificadores de acceso controlan la visibilidad de clases, metodos y campos desde otras partes del programa. Java define cuatro niveles de acceso: `public`, `protected`, `default` (tambien llamado package-private) y `private`. La eleccion del modificador adecuado es fundamental para el encapsulamiento y el diseno de una API estable.

## public

Un elemento declarado como `public` es accesible desde cualquier clase de cualquier paquete. Es el nivel mas permisivo. Se usa para la API publica de una clase: metodos y constantes que forman parte del contrato con los consumidores del componente.

```java
package com.ejemplo.api;

public class ServicioUsuarios {
    public void registrarUsuario(String email) {
        // accesible desde cualquier parte
    }
}
```

```java
package com.ejemplo.cliente;
import com.ejemplo.api.ServicioUsuarios;

public class Cliente {
    public void ejecutar() {
        ServicioUsuarios s = new ServicioUsuarios();
        s.registrarUsuario("test@test.com"); // valido
    }
}
```

> La sobrexposicion de elementos `public` compromete la capacidad de cambiar la implementacion interna sin romper codigo cliente. Por eso el principio de minimo privilegio recomienda usar el nivel mas restrictivo posible.

## protected

El acceso `protected` permite la visibilidad desde el mismo paquete y desde cualquier subclase, incluso si la subclase esta en otro paquete. Es util para metodos que las subclases necesitan heredar o sobrescribir pero que no deben ser parte de la API publica.

```java
package com.ejemplo.base;

public class ClaseBase {
    protected void metodoInterno() {
        // visible para subclases y mismo paquete
    }
}
```

```java
package com.ejemplo.derivado;
import com.ejemplo.base.ClaseBase;

public class ClaseDerivada extends ClaseBase {
    public void test() {
        metodoInterno(); // valido: subclase en otro paquete
    }
}
```

```java
package com.ejemplo.otro;

public class OtraClase {
    public void test() {
        ClaseBase b = new ClaseBase();
        // b.metodoInterno(); // ERROR: no es subclase ni mismo paquete
    }
}
```

## Default (package-private)

Cuando no se especifica ningun modificador, el acceso es default o package-private. El elemento es visible unicamente para clases del mismo paquete. Es el nivel de acceso mas utilizado para implementacion interna que no debe trascender los limites del paquete.

```java
package com.ejemplo.nucleo;

class Utilidades {
    // package-private: solo visible en com.ejemplo.nucleo
    static int calcularHash(String dato) {
        return dato.hashCode();
    }
}
```

```java
package com.ejemplo.nucleo;

public class Procesador {
    public void ejecutar() {
        Utilidades.calcularHash("test"); // valido: mismo paquete
    }
}
```

El acceso default es el nivel natural para clases de soporte interno que no forman parte de la API publica. Al estar limitado al paquete, permite reorganizar la implementacion interna sin afectar a modulos externos.

## private

El modificador `private` restringe el acceso a la propia clase que lo declara. Ninguna otra clase, ni siquiera subclases o clases del mismo paquete, puede acceder a miembros privados.

```java
public class CuentaBancaria {
    private double saldo; // solo accesible desde esta clase

    private void validarSaldo(double cantidad) {
        if (cantidad < 0) {
            throw new IllegalArgumentException("Saldo negativo");
        }
    }

    public void depositar(double cantidad) {
        validarSaldo(cantidad);
        this.saldo += cantidad;
    }
}
```

Los miembros privados son el pilar del encapsulamiento. Permiten ocultar los detalles de implementacion y exponer solo una interfaz controlada mediante metodos publicos.

## Tabla resumen de alcance

| Modificador | Misma clase | Mismo paquete | Subclase otro paquete | Cualquier clase |
|-------------|-------------|---------------|----------------------|------------------|
| `private`     | Si          | No            | No                   | No               |
| `default`     | Si          | Si            | No                   | No               |
| `protected`   | Si          | Si            | Si                   | No               |
| `public`      | Si          | Si            | Si                   | Si               |

## Cuando usar cada uno

La estrategia recomendada es comenzar con el acceso mas restrictivo y solo aumentarlo cuando sea estrictamente necesario.

**`private`** para campos de instancia y metodos auxiliares internos. Casi todos los campos deben ser `private`. La mayoria de los metodos de implementacion deben ser `private`.

**`default`** para clases y metodos que forman parte de la implementacion interna de un modulo pero que necesitan ser compartidos entre varias clases del mismo paquete. Es comun en paquetes que implementan una funcionalidad cohesiva.

**`protected`** para metodos y constructores que las subclases necesitan heredar o sobrescribir pero que no deben exponerse al mundo exterior. Tambien para metodos plantilla (template method pattern).

**`public`** para la API estable que otras clases y paquetes consumiran. Cada elemento `public` es un compromiso de soporte a futuro. Las interfaces y sus metodos son implicitamente `public`.

El uso adecuado de modificadores de acceso no solo previene accesos indebidos, sino que comunica la intencion de diseno: que es parte del contrato, que es extension posible, y que es detalle interno.
