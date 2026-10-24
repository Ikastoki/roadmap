# Testing en equipo

El testing en equipo no es solo una cuestion tecnica, sino cultural. Las practicas individuales deben alinearse con acuerdos colectivos para que la suite de tests sea consistente, mantenible y valorada por todos los miembros del equipo.

## Code review de tests

Los tests deben revisarse con el mismo rigor que el codigo de produccion. Un code review de tests verifica:

- Que el test cubre el comportamiento descrito en el nombre.
- Que no hay test smells (eager test, mystery guest, excesivo setup).
- Que las aserciones son precisas y verifican el comportamiento, no la implementacion.
- Que el test es rapido y aislado.

```java
// Durante el code review se detectaria que este test es fragil
@Test
void descuento() {
    assertThat(servicio.calcular(100)).isEqualTo(15); // ? por que 15?
}
```

El revisor pediria un nombre mas descriptivo o un @DisplayName que explique el escenario.

## CI/CD

Los tests deben ejecutarse automaticamente en cada commit y en cada push. El pipeline de CI/CD debe incluir:

1. Compilacion del proyecto.
2. Ejecucion de tests unitarios (Surefire).
3. Verificacion de cobertura minima (JaCoCo check).
4. Ejecucion de tests de integracion (Failsafe).
5. (Opcional) Mutation testing en modulos criticos.

```yaml
# GitHub Actions
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
      - run: mvn verify
```

Si el build falla por un test, la prioridad del equipo debe ser repararlo antes de continuar con nuevas funcionalidades. Un pipeline roto que se ignora pierde todo su valor.

## Cobertura minima

El equipo debe acordar un umbral de cobertura minima y documentarlo en el `AGENTS.md` o en el README del proyecto. Este umbral se verifica automaticamente en CI.

El acuerdo debe incluir:

- El porcentaje minimo global (ej: 80 %).
- Las exclusiones aceptadas (codigo generado, DTOs, configuracion).
- Las consecuencias de no alcanzar el umbral (el build falla, pero se puede revisar).
- El proceso para ajustar el umbral cuando sea necesario.

## Propiedad compartida

Los tests no son propiedad de quien los escribio, sino del equipo. Cualquier miembro debe sentirse capacitado y autorizado para modificar, mejorar o eliminar tests cuando sea necesario. La propiedad compartida evita el sindrome de "esto es mio, no lo toques" y fomenta la mejora continua.

Para facilitar la propiedad compartida:

- Los tests se escriben siguiendo las mismas convenciones que el codigo de produccion.
- El codigo de testing se revisa en los mismos code reviews.
- No hay "especialistas en testing"; todos los desarrolladores escriben y mantienen tests.
- Las metricas de testing (cobertura, tiempo de ejecucion, numero de tests) son visibles para todo el equipo.

> Un equipo que no valora sus tests terminara con tests que no valen nada. La cultura de testing se construye con acuerdos, revisiones y, sobre todo, con el ejemplo de los desarrolladores senior.
