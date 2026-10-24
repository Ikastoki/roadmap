# Limitaciones de la arquitectura de capas

Ningun patron arquitectonico es universalmente aplicable, y la arquitectura de capas no es una excepcion. Su adopcion conlleva costes que en determinados contextos pueden superar a los beneficios. Conocer estas limitaciones es esencial para decidir cuando aplicarla y, sobre todo, cuando evitarla o combinarla con otros patrones.

## Overhead de llamadas

Cada operacion de negocio atraviesa todas las capas: el controlador invoca al servicio, el servicio invoca al repositorio, y la respuesta recorre el camino inverso. En aplicaciones con una profundidad de tres o cuatro capas, este encadenamiento introduce una sobrecarga en terminos de rendimiento: cada salto de capa implica una llamada a metodo, una transicion de contexto (aunque sea dentro del mismo hilo) y, a menudo, la creacion de objetos de transferencia de datos (DTO) que traducen informacion entre capas.

En sistemas de alto rendimiento o baja latencia --procesamiento de transacciones financieras, streaming en tiempo real, juegos multijugador-- esta sobrecarga, aunque pequena por llamada, se acumula hasta volverse significativa bajo carga elevada. La solucion clasica es colapsar capas en los caminos criticos o utilizar modelos alternativos como la arquitectura hexagonal o CQRS, que reducen el numero de saltos.

> En aplicaciones empresariales tipicas, el overhead de capas es irrelevante comparado con el tiempo de acceso a base de datos o a servicios externos. Solo conviene preocuparse cuando el tiempo de CPU interno domina el perfil de rendimiento.

## Acoplamiento inevitable

Aunque la arquitectura de capas promueve el desacoplamiento, introduce una forma sutil de acoplamiento: todas las capas dependen de las abstracciones de las capas inferiores. Un cambio en la firma de un metodo de repositorio obliga a modificar el servicio que lo llama y, potencialmente, el controlador. Este efecto domino contradice parcialmente la promesa de aislamiento total.

El problema se agrava cuando las capas inferiores filtran detalles de implementacion hacia arriba. Por ejemplo, si un repositorio lanza una excepcion de tipo `SQLException`, la capa de negocio se ve forzada a conocer detalles de persistencia, rompiendo la separacion de responsabilidades. La mitigacion tipica en Java es envolver las excepciones tecnicas en excepciones de dominio (`DataAccessException` es el ejemplo canonico en Spring), pero esto anade mas codigo de infraestructura.

## Complejidad en aplicaciones simples

Para una aplicacion de pocas funcionalidades --un CRUD basico, un script de migracion, un microservicio con una unica responsabilidad-- la arquitectura de capas introduce una complejidad accidental desproporcionada. Crear cuatro paquetes, definir interfaces para cada capa, implementar DTOs y configurar la inyeccion de dependencias multiplica el esfuerzo inicial sin aportar beneficios tangibles cuando el ambito es reducido.

En estos casos, el principio de YAGNI (You Aint Gonna Need It) aconseja comenzar con una estructura plana y evolucionar hacia capas solo cuando la complejidad lo justifique. Martin Fowler lo denomina "arquitectura estrangulada": empezar simple y estratificar gradualmente a medida que el sistema crece. Muchos frameworks Java modernos, como Spring Boot, facilitan este enfoque permitiendo anadir capas sin reestructurar el proyecto completo.

> La decision de usar capas no es binaria. Un microservicio de 200 lineas probablemente no necesita capas; una aplicacion corporativa de 200.000 lineas probablemente no puede prescindir de ellas.
