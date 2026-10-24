# Michael Feathers y el codigo legado

Michael Feathers, en su libro "Working Effectively with Legacy Code", establece una de las definiciones mas influyentes de codigo legacy: codigo sin tests. Bajo esta premisa, cualquier codigo, independientemente de su antiguedad, calidad o tecnologia, es legacy si carece de tests que lo respalden.

## Definicion legacy code

Feathers sostiene que el codigo legacy no se define por su edad o por estar escrito en un lenguaje obsoleto, sino por la ausencia de una red de seguridad que permita modificarlo con confianza. Sin tests, cualquier cambio es una apuesta: no se sabe si la modificacion rompe algo hasta que se despliega en produccion.

```
Codigo legacy = codigo sin tests
```

Esta definicion es liberadora porque cambia el enfoque de "reemplazar codigo antiguo" a "poner codigo bajo test". No importa si el codigo tiene 20 anos o fue escrito ayer; si no tiene tests, es legacy.

## Dependency Injection (DI)

La falta de testabilidad en codigo legacy suele deberse a dependencias duras: el codigo crea sus propias dependencias dentro del constructor o del metodo, impidiendo que los tests puedan reemplazarlas por dobles.

```java
// Codigo legacy: dependencia dura
class Procesador {
    private final BaseDatos bd = new BaseDatos(); // no se puede mockear

    void ejecutar() {
        bd.guardar(datos);
    }
}

// Refactor: dependency injection
class Procesador {
    private final BaseDatos bd;

    Procesador(BaseDatos bd) { // inyectable
        this.bd = bd;
    }
}
```

Feathers recomienda introducir DI de forma gradual, extrayendo la creacion de dependencias a constructores o metodos factory, sin modificar la logica de negocio.

## Seam model

Un seam es un lugar donde se puede alterar el comportamiento del programa sin modificar el codigo en ese lugar. Feathers distingue dos tipos principales:

- **Preprocessing seam**: en lenguajes con preprocesador (C, C++), se pueden cambiar definiciones antes de compilar.
- **Object seam**: en Java, los seams son metodos que se pueden sobrescribir en una subclase de test.

```java
// Codigo original
class ReporteService {
    DatabaseConnection getConnection() {
        return new DatabaseConnection("produccion");
    }

    void generar() {
        Connection conn = getConnection(); // seam
        // ...
    }
}

// Test: sobrescribir el seam
class ReporteServiceTest extends ReporteService {
    @Override
    DatabaseConnection getConnection() {
        return mock(DatabaseConnection.class); // conexion de prueba
    }
}
```

Identificar seams en codigo legacy es el paso mas importante para poder testearlo. Cada seam ofrece un punto de entrada para los tests sin modificar la estructura existente.

## Sprout techniques

Las tecnicas de sprout (brote) permiten anadir nueva funcionalidad a codigo legacy sin modificar el codigo existente:

- **Sprout method**: extraer la nueva logica a un metodo nuevo y llamarlo desde el original.
- **Sprout class**: crear una clase nueva para la funcionalidad e inyectarla.

```java
// Antes: anadir logica directamente en metodo legacy
void procesarPedido(Pedido p) {
    // 200 lineas existentes
    if (p.esInternacional()) { // nuevo codigo mezclado
        p.calcularArancel();
    }
    // mas codigo existente
}

// Despues: sprout method
void procesarPedido(Pedido p) {
    // 200 lineas existentes
    aplicarArancelSiAplica(p); // solo una linea nueva
    // mas codigo existente
}

private void aplicarArancelSiAplica(Pedido p) {
    if (p.esInternacional()) {
        p.calcularArancel();
    }
}
```

El metodo extraido es testeable de forma independiente, mientras que el metodo original permanece intacto pero ahora invoca codigo que tiene tests.

> Feathers no propone reescribir el codigo legacy. Propone envolverlo, extraer partes y anadir tests gradualmente hasta que el codigo legacy quede completamente cubierto y se pueda refactorizar o eliminar con seguridad.
