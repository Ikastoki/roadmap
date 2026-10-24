# Coverage tools comparativa

Elegir la herramienta de cobertura adecuada depende del tipo de proyecto, los requisitos del pipeline y la profundidad de analisis deseada. Las opciones principales en el ecosistema Java son JaCoCo, Cobertura, PIT y SonarQube.

## JaCoCo vs Cobertura

Cobertura fue durante anos la herramienta estandar, pero ha quedado practicamente abandonada. Su ultima version estable data de 2015 y no soporta Java moderno ni JUnit 5 de forma nativa. JaCoCo, en cambio, recibe actualizaciones regulares, soporta Java 21 y superiores, y ofrece reportes mas detallados con branch coverage real.

| Caracteristica        | JaCoCo | Cobertura |
|-----------------------|--------|-----------|
| Line coverage         | Si     | Si        |
| Branch coverage       | Si     | Limitado  |
| Java 21+              | Si     | No        |
| JUnit 5               | Si     | No        |
| Reportes HTML         | Si     | Si        |
| XML para SonarQube    | Si     | Si        |
| Mantenimiento activo  | Si     | No        |

JaCoCo es la opcion recomendada para cualquier proyecto nuevo o existente.

## PIT

PIT no es una herramienta de cobertura en sentido estricto, sino de mutation testing. Sin embargo, ofrece una metrica de calidad de tests mucho mas valiosa que el coverage tradicional. Mientras que JaCoCo responde a "que codigo se ejecuta", PIT responde a "que tests detectarian un error real". La combinacion de ambas es la estrategia mas solida.

PIT consume significativamente mas tiempo que JaCoCo. Para un proyecto con 1000 tests, JaCoCo completa el analisis en segundos, mientras que PIT puede tardar minutos u horas dependiendo del numero de mutantes. Por eso se recomienda usar JaCoCo en cada commit y PIT en momentos selectivos del pipeline.

## SonarQube

SonarQube no mide cobertura directamente, sino que consume los reportes XML generados por JaCoCo o Cobertura para integrarlos en su panel de calidad. Ademas del porcentaje global, SonarQube correlaciona la cobertura con la complejidad ciclomatica, el duplicacion de codigo y las vulnerabilidades, ofreciendo una vision holistic.

```properties
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

SonarQube permite definir quality gates que combinan cobertura minima con otras metricas (bugs, code smells, vulnerabilidades). Esto evita el problema de perseguir un numero unico y fomenta una vision equilibrada de la calidad.

## Eleccion recomendada

- **Proyecto nuevo**: JaCoCo + SonarQube en CI. Anadir PIT para modulos criticos.
- **Proyecto legacy**: JaCoCo para establecer una linea base. PIT para medir la efectividad real de los tests de caracterizacion.
- **Proyecto pequeno**: Solo JaCoCo, con verificacion en el build.
- **Proyecto con requisitos regulatorios**: JaCoCo + PIT + SonarQube con quality gates estrictos.

Ninguna herramienta por si sola garantiza la calidad de los tests. La combinacion de cobertura tradicional y mutation testing, junto con revisiones humanas, ofrece la mejor garantia.
