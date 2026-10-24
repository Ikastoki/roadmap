# Tipos de testing

El testing de software se clasifica en categorias que atienden a diferentes niveles de abstraccion, alcance y objetivos. Comprender cada tipo y saber cuando aplicarlo es fundamental para disenar una estrategia de testing equilibrada que maximice la deteccion de defectos sin duplicar esfuerzos ni generar costes innecesarios.

## Unit testing

Las pruebas unitarias verifican el comportamiento de la unidad mas pequeña de codigo de forma aislada: un metodo, una funcion o una clase. El objetivo es confirmar que esa unidad produce la salida correcta para un conjunto dado de entradas, incluyendo casos borde, valores nulos y condiciones de error. Para lograr el aislamiento, las dependencias externas (bases de datos, servicios HTTP, sistemas de ficheros) se reemplazan por test doubles. Los unit tests constituyen la base de la piramide de testing y deben ejecutarse en milisegundos, permitiendo cientos o miles de ejecuciones en cada commit. Un unit test bien escrito es deterministico, rapido y prueba una sola responsabilidad.

> Un unit test que interactua con la base de datos o la red no es un unit test: es un test de integracion disfrazado. La prueba unitaria debe ejecutarse en completo aislamiento para ser fiable y rapida.

## Integration testing

Los tests de integracion verifican que multiples componentes funcionan correctamente cuando se combinan. A diferencia de los unit tests, aqui las dependencias reales (o versiones simplificadas pero reales) participan en la prueba: una base de datos en memoria, un servidor HTTP embebido, un sistema de colas. El objetivo es detectar problemas de comunicacion, formato de datos, contratos y estados compartidos que no aparecen en aislamiento. Los integration tests son inherentemente mas lentos y fragiles que los unit tests, pero cubren un espacio de fallos que estos no pueden alcanzar. La clave esta en limitar su alcance: probar la interaccion entre dos o tres componentes, no el sistema completo.

## E2E testing

Las pruebas de extremo a extremo (end-to-end, E2E) validan el sistema completo desde la perspectiva del usuario final. Simulan flujos completos: abrir un navegador, autenticarse, navegar, introducir datos, enviar formularios y verificar resultados en la interfaz. Herramientas como Selenium, Cypress o Playwright permiten automatizar estos escenarios. Los E2E tests son los mas lentos (segundos o minutos), fragiles (cambios en la interfaz los rompen) y costosos de mantener. Por esta razon deben reservarse para los flujos criticos del negocio (happy paths) y ejecutarse con moderacion, tipicamente antes de liberar una version.

## Smoke testing

Los smoke tests (pruebas de humo) son un subconjunto minimo de pruebas que verifican que las funcionalidades mas basicas del sistema funcionan despues de un despliegue o build. El termino proviene de la electronica: si al encender un equipo salia humo, no tenia sentido seguir probando. En software, un smoke test confirma que la aplicacion arranca, responde a peticiones y ejecuta las operaciones fundamentales antes de invertir tiempo en pruebas mas profundas. Suelen ser un puñado de tests E2E o de integración que se ejecutan como primer paso en el pipeline de CI/CD.

## Regression testing

Las pruebas de regresion verifican que los cambios recientes en el codigo no han roto funcionalidad existente. Cualquier test que haya pasado antes y falle despues de un cambio indica una regresion. La suite de regression esta compuesta tipicamente por unit tests (la mayoria), integration tests y algunos E2E tests. Automatizar las pruebas de regresion es el argumento mas solido para invertir en testing: sin una red de seguridad automatizada, cada cambio requiere volver a probar manualmente todo el sistema, lo que resulta inviable a medida que el producto crece. Un alto porcentaje de cobertura de codigo en unit tests reduce drásticamente el riesgo de regresiones no detectadas.

## Exploratory testing

El testing exploratorio es una aproximacion no guionizada donde el probador disena y ejecuta pruebas en tiempo real, usando su conocimiento del dominio y su creatividad para descubrir comportamientos inesperados. No sigue casos de prueba predefinidos, sino que explora rutas de ejecucion, combinaciones de entrada y estados del sistema que los tests automatizados no cubren. Es especialmente valioso en fases tempranas del desarrollo, cuando los requisitos son inestables, y como complemento a la automatizacion para encontrar vulnerabilidades de seguridad o problemas de usabilidad. A diferencia del testing manual estructurado, el exploratory testing se basa en la experiencia y el criterio del probador para adaptar dinamicamente la estrategia de prueba.
