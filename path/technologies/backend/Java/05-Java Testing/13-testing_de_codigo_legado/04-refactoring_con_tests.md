# Refactoring con tests

El refactoring es el proceso de modificar el codigo interno sin alterar su comportamiento externo. Los tests son el habilitador critico del refactoring: sin ellos, no hay forma de saber si el comportamiento se ha preservado.

## Red-Green-Refactor en legacy

El ciclo Red-Green-Refactor clasico de TDD tambien se aplica al codigo legacy, pero con una variante importante: el "Red" inicial no es un test que falla por falta de implementacion, sino un characterization test que captura el comportamiento actual.

1. **Red**: escribir un characterization test (o grupo de tests) que refleje el comportamiento actual del metodo a refactorizar.
2. **Green**: ejecutar el test contra el codigo actual y verificar que pasa.
3. **Refactor**: modificar el codigo interno manteniendo la interfaz publica.
4. **Green**: ejecutar los tests de nuevo. Si fallan, el refactoring ha alterado el comportamiento.

```java
// Paso 1: characterization test
@Test
void refactorCalculoImpuesto() {
    double resultado = factura.calcularImpuesto(100.0, "VIP");
    assertThat(resultado).isEqualTo(15.0); // comportamiento actual
}

// Paso 3: refactorizar el codigo interno
// ...sin cambiar la firma del metodo...

// Paso 4: el test debe seguir pasando
```

## Small steps

La clave del refactoring con tests son los pasos pequenos. Cada cambio debe ser minimo y reversible. Si el test falla, se sabe exactamente que cambio lo provoco.

```java
// Paso 1: extraer una constante
void calcular() {
    final double IVA = 0.21; // solo esto cambia
    total = subtotal * 0.21;
}

// Paso 2: extraer metodo
void calcular() {
    total = aplicarIva(subtotal);
}

private double aplicarIva(double valor) {
    return valor * 0.21;
}
```

Cada paso va seguido de la ejecucion de los tests. Si todos pasan, se avanza al siguiente. Si alguno falla, se revierte el cambio y se intenta de otra forma.

## Strangler Fig

El patron Strangler Fig (higuera estranguladora) se usa para reemplazar gradualmente un sistema legacy. En lugar de reescribir todo de golpe, se va envolviendo el codigo legacy con nuevo codigo hasta que el legacy queda obsoleto.

A nivel de codigo, el patron se aplica creando una nueva implementacion de una interfaz existente, y luego redirigiendo las llamadas progresivamente.

```java
// Interfaz existente
interface RepositorioUsuarios {
    Usuario buscar(String id);
}

// Implementacion legacy (no testeable)
class RepositorioLegacy implements RepositorioUsuarios { ... }

// Nueva implementacion (testeable)
class RepositorioNuevo implements RepositorioUsuarios { ... }

// Redirigir llamadas progresivamente
class RepositorioFacade implements RepositorioUsuarios {
    private final RepositorioUsuarios legacy = new RepositorioLegacy();
    private final RepositorioUsuarios nuevo = new RepositorioNuevo();

    @Override
    public Usuario buscar(String id) {
        if (caracteristicaActiva(id)) {
            return nuevo.buscar(id);
        }
        return legacy.buscar(id);
    }
}
```

## Parallel change

El patron parallel change (o expand-contract) permite modificar una interfaz sin romper los consumidores existentes. Se anade el nuevo metodo, se migran los consumidores, y se elimina el antiguo.

```java
// Fase 1: expandir (anadir nuevo metodo)
interface Repositorio {
    Usuario buscarPorId(String id);           // antiguo
    Usuario buscar(UsuarioCriteria criteria);  // nuevo
}

// Fase 2: migrar consumidores al nuevo metodo
// Fase 3: contraer (eliminar metodo antiguo)
interface Repositorio {
    Usuario buscar(UsuarioCriteria criteria);
}
```

Cada fase va acompanada de tests que verifican que el comportamiento no cambia. El parallel change es especialmente util en APIs publicas donde no se controlan todos los consumidores.

> El refactoring sin tests no es refactoring, es reescritura. Sin la red de seguridad, no se puede distinguir entre un cambio seguro y uno que introduce errores.
