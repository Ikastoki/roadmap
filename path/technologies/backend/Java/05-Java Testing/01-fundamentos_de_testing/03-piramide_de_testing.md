# Piramide de testing

La piramide de testing es un modelo conceptual introducido por Mike Cohn que describe la distribucion ideal de los diferentes tipos de pruebas en un proyecto software. La metafora es arquitectonica: una piramide estable requiere una base ancha y solida sobre la que descansa el resto de la estructura. En testing, esa base son los unit tests, seguidos de los integration tests y, en la cuspide, los E2E tests.

## Base: unit tests

Los unit tests constituyen el cimiento de la piramide. Deben ser la categoria mas numerosa, con decenas o cientos de tests por cada integration o E2E test. Su velocidad de ejecucion (milisegundos por test) permite ejecutar la suite completa en segundos, lo que los convierte en la primera linea de defensa contra regresiones. Son baratos de escribir y mantener porque trabajan con componentes aislados sin infraestructura externa. Un proyecto saludable dedica entre el 60 % y el 80 % de su esfuerzo de testing a esta capa. La cobertura de codigo, aunque no debe ser un objetivo en si mismo, suele ser alta en esta capa simplemente como consecuencia de probar todas las unidades de forma sistematica.

> Una base solida de unit tests permite detectar el 70-80 % de los defectos en segundos, directamente en el entorno del desarrollador, sin necesidad de desplegar ni configurar infraestructura alguna.

## Medio: integration tests

La capa intermedia corresponde a los tests de integracion. Su numero es menor (tipicamente entre un 10 % y un 20 % del total) porque cada uno de ellos abarca la interaccion entre varios componentes y requiere cierto grado de infraestructura: una base de datos, un cliente HTTP, un sistema de mensajeria. Su ejecucion es mas lenta (cientos de milisegundos o segundos), pero cubren un espacio de fallos que los unit tests no pueden abordar. Aqui se verifican los contratos entre modulos, el mapeo objeto-relacional, la serializacion de datos y el comportamiento en presencia de dependencias reales. La clave para mantener esta capa manejable es usar entornos ligeros: bases de datos embebidas (H2), servidores HTTP mockeados (WireMock) o contenedores desechables (Testcontainers).

## Cima: E2E tests

En la cuspide de la piramide se encuentran los tests de extremo a extremo. Son los menos numerosos (5-10 % del total), los mas lentos (segundos o minutos) y los mas costosos de mantener. Cada E2E test ejercita el sistema completo, incluyendo interfaz de usuario, servicios backend, bases de datos y servicios externos. Debido a su fragilidad (un cambio en un selector CSS o en la estructura de una pagina puede romperlos), deben reservarse exclusivamente para los flujos criticos del negocio. Su funcion no es detectar bugs detallados (eso ya lo hacen los unit e integration tests), sino verificar que el sistema completo funciona como un todo cohesivo.

## Anti-patrones en la piramide

El modelo de la piramide se viola con frecuencia, dando lugar a anti-patrones que generan suites de testing lentas, fragiles y dificiles de mantener.

El cono de helado (ice-cream cone) invierte la piramide: pocos unit tests y muchos E2E tests. Esto suele ocurrir en proyectos que adoptan el testing tarde o que dependen excesivamente de herramientas de automatizacion de interfaz de usuario. El resultado es una suite lenta que tarda horas en ejecutarse, tests que fallan intermitentemente (flaky tests) y un mantenimiento costoso que termina desincentivando la escritura de nuevos tests.

El reloj de arena (hourglass) presenta una base solida de unit tests y muchos E2E tests, pero muy pocos integration tests. En esta configuracion, los componentes individuales funcionan correctamente y el sistema completo responde, pero las interacciones entre componentes quedan insuficientemente validadas. Los bugs de integracion aparecen tipicamente en produccion, cuando los datos reales recorren caminos que no se probaron adecuadamente en la interfaz entre modulos.

El trofeo (trophy) es un modelo alternativo propuesto por algunos autores que sugiere priorizar los integration tests sobre los unit tests, argumentando que estos ultimos prueban implementacion en lugar de comportamiento. Aunque menos extendido que la piramide clasica, este enfoque gana traccion en proyectos donde el valor del negocio reside en las interacciones entre servicios, como en arquitecturas de microservicios.
