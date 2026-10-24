# Que es TDD

El Desarrollo Guiado por Pruebas, conocido por sus siglas en ingles TDD (Test-Driven Development), invierte el orden natural del desarrollo de software: en lugar de escribir primero la logica de negocio y despues los tests que la validan, TDD propone escribir el test antes que el codigo de produccion. Este cambio, aparentemente trivial, transforma radicalmente la forma en que se disena, estructura y verifica el software, desplazando el foco desde la implementacion hacia el comportamiento esperado.

## Red-Green-Refactor

El ciclo fundamental de TDD se compone de tres fases que se repiten de forma continua:

- **Red**: Se escribe un test que falla. El test describe una funcionalidad que aun no existe. Es crucial que el test falle para confirmar que estamos probando algo que realmente no esta implementado.
- **Green**: Se escribe el minimo codigo de produccion necesario para que el test pase. En esta fase no importa la calidad del codigo; la prioridad es hacer que el test verde.
- **Refactor**: Se mejora el codigo de produccion (y eventualmente el test) sin anadir nueva funcionalidad. Se eliminan duplicaciones, se mejoran nombres, se aplican patrones de diseno.

```java
// Red: el test falla porque la clase no existe
@Test
void shouldReturnTrueWhenValueIsPositive() {
    assertTrue(MathUtils.isPositive(5));
}

// Green: codigo minimo para que el test pase
public class MathUtils {
    public static boolean isPositive(int value) {
        return true; // minimo indispensable
    }
}

// Refactor: se mejora la implementacion
public class MathUtils {
    public static boolean isPositive(int value) {
        return value > 0;
    }
}
```

## Por que TDD

TDD no es una tecnica de testing, sino una tecnica de diseno. Los beneficios trascienden la mera verificacion de codigo y afectan a la arquitectura del proyecto.

### Diseno emergente

Al escribir el test primero, el diseno emerge de forma natural guiado por el uso que el codigo cliente hace de la API. Esto conduce a interfaces minimas, cohesivas y con baja suposicion sobre el funcionamiento interno. Se evita la sobreingenieria porque solo se escribe el codigo estrictamente necesario para satisfacer el test actual.

### Confianza y red de seguridad

Un conjunto de tests que cubre todo el comportamiento del sistema proporciona una red de seguridad que permite modificar el codigo con confianza. Cualquier cambio que rompa una funcionalidad existente se detecta en segundos. Esto hace que el refactoring sea una actividad segura y frecuente, no un evento excepcional.

### Documentacion viviente

Los tests en TDD constituyen una especificacion ejecutable del sistema. Cualquier desarrollador puede leer los tests para comprender que debe hacer una unidad de codigo y cual es su comportamiento esperado en cada escenario. A diferencia de la documentacion tradicional, los tests nunca quedan desactualizados porque si fallan, el equipo lo sabe de inmediato.

> Un test que no falla cuando debe es un test que miente. Por eso la fase Red es tan importante como la fase Green.

## TDD vs Testing tradicional

| Aspecto | Testing tradicional | TDD |
|---------|-------------------|-----|
| Orden | Codigo primero, test despues | Test primero, codigo despues |
| Proposito | Verificar codigo existente | Guiar el diseno del codigo |
| Cobertura | Depende de la disciplina del equipo | Naturalmente alta |
| Acoplamiento | Tests pueden acoplarse a implementacion | Tests desacoplados por diseno |
| Refactoring | Arriesgado sin tests | Seguro por la red de tests |

El testing tradicional verifica que el codigo funciona, pero no influye en como se escribe ese codigo. TDD, en cambio, condiciona cada decision de implementacion desde el primer momento, forzando un diseno modular, desacoplado y testeable.

> La diferencia fundamental no es el test en si, sino el momento en que se escribe. Escribir el test antes cambia la perspectiva de "como verifico esto" a "que comportamiento necesito".
