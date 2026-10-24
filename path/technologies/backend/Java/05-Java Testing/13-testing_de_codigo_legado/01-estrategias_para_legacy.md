# Estrategias para legacy code

El codigo legacy se caracteriza por ser dificil de testear, con acoplamientos fuertes, dependencias ocultas y diseno que no contempla la testabilidad. Modificar este codigo sin tests es peligroso; anadir tests sin refactorizar es dificil. Las estrategias de Feathers, descritas en "Working Effectively with Legacy Code", proporcionan un conjunto de tecnicas para salir de este circulo vicioso.

## Characterization tests

Los characterization tests (o tests de caracterizacion) son la primera linea de defensa. En lugar de escribir tests que verifiquen el comportamiento deseado (que a menudo se desconoce), se escribe un test que capture el comportamiento actual. El test registra lo que el sistema hace ahora, no lo que deberia hacer.

```java
@Test
void comportamientoActualDeFacturacion() {
    Factura factura = servicio.generarFactura("C-001", 100.0);
    assertThat(factura.getTotal()).isEqualTo(121.0); // IVA 21%
    assertThat(factura.getEstado()).isEqualTo(Estado.PENDIENTE);
}
```

Si el test falla al cambiar el codigo, se sabe que el comportamiento se ha modificado. Esto no garantiza que el comportamiento sea correcto, pero si que es el que se esperaba conservar.

## Sprout method

Cuando hay que anadir una funcionalidad a una clase no testeable, se extrae el nuevo codigo a un metodo nuevo (sprout method) que puede testearse de forma independiente. El metodo original invoca al nuevo, y los tests se escriben contra el metodo nuevo.

```java
// Codigo original (no testeable)
void procesarPedido(Pedido pedido) {
    // 100 lineas de logica no testeable
    // ...
    double impuesto = pedido.getTotal() * 0.21;
    pedido.setImpuesto(impuesto);
    // ...
}

// Refactor: extraer logica a metodo testeable
double calcularImpuesto(Pedido pedido) {
    return pedido.getTotal() * 0.21;
}

void procesarPedido(Pedido pedido) {
    // ... codigo existente ...
    double impuesto = calcularImpuesto(pedido);
    pedido.setImpuesto(impuesto);
    // ...
}
```

El test se escribe contra `calcularImpuesto`, que ahora es testeable.

## Sprout class

Si la nueva funcionalidad es suficientemente compleja como para justificar una clase propia, se crea una nueva clase que contenga la logica y se inyecta en la clase original. Esto es preferible a anadir responsabilidades a una clase ya sobrecargada.

```java
public class CalculadorImpuesto {
    public double calcular(double total, String tipoCliente) {
        // logica testeable
    }
}

public class ProcesadorPedido {
    private final CalculadorImpuesto calculador;

    public ProcesadorPedido(CalculadorImpuesto calculador) {
        this.calculador = calculador;
    }
}
```

## Wrap method

Cuando se necesita anadir comportamiento antes o despues de un metodo existente, se crea un nuevo metodo con el nombre original y se renombra el antiguo. El nuevo metodo invoca al antiguo y anade la funcionalidad adicional.

```java
// Original
void guardarCliente(Cliente cliente) { ... }

// Wrap method
void guardarCliente(Cliente cliente) {
    logger.info("Guardando cliente: {}", cliente.getId());
    guardarClienteImpl(cliente);
}

private void guardarClienteImpl(Cliente cliente) { ... }
```

## Expose static method

Cuando un metodo usa dependencias estaticas o no accesibles, se puede exponer una version estatica parcial que acepte los parametros necesarios.

```java
// Original
void procesar() {
    Configuracion config = Configuracion.getInstance();
    // ...
}

// Refactor
static void procesarConConfig(Configuracion config) {
    // logica testeable
}
```

## Seams

Un seam es un punto donde se puede alterar el comportamiento sin modificar el codigo fuente. En Java, los seams incluyen:

- Sobrescritura de metodos en tests (extender la clase y sobrescribir un metodo protegido).
- Inyeccion de dependencias mediante setters o constructores.
- Factories que se pueden reemplazar en tests.
- Parametrizacion de metodos para aceptar dependencias.

```java
// Seam: sobrescritura de metodo protegido
class ProcesadorTestable extends Procesador {
    @Override
    protected ServicioExterno crearServicio() {
        return mock(ServicioExterno.class);
    }
}
```

Cada una de estas tecnicas permite ganar testabilidad sin una reescritura completa. El objetivo no es lograr tests perfectos, sino establecer una base que permita refactorizar con seguridad.
