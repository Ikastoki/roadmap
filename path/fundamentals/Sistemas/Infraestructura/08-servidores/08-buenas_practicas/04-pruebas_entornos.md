# Pruebas y Entornos

La infraestructura, al igual que el codigo, requiere ser probada antes de llegar a produccion. Un cambio en una regla de firewall, una actualizacion de sistema operativo o una reconfiguracion de red puede tener consecuencias que no se anticipan si no se validan previamente. La cultura de probar los cambios de infraestructura en entornos controlados antes de aplicarlos en produccion es lo que separa a los equipos reactivos, que corrigen fallos en produccion, de los equipos proactivos, que previenen que los fallos lleguen a produccion.

## Entornos progresivos

El modelo de entornos progresivos establece una cadena de validacion donde los cambios se mueven secuencialmente desde un entorno de bajo riesgo hacia produccion, ganando confianza en cada etapa. Cada entorno replica las caracteristicas del siguiente en la cadena, con la excepcion de la escala y la sensibilidad de los datos que alberga.

### Desarrollo

El entorno de desarrollo es donde los ingenieros trabajan activamente en nuevas configuraciones, scripts de despliegue o cambios de infraestructura. Este entorno es intencionalmente inestable: se permite experimentar, romper y reconstruir sin consecuencias sobre usuarios reales. Los datos que contiene son sinteticos o anonimizados, y la configuracion puede variar significativamente de produccion en terminos de escala, pero no en terminos de arquitectura.

El valor del entorno de desarrollo radica en la velocidad de iteracion. Un ingeniero puede probar un cambio de configuracion en minutos, observar su comportamiento y refinarlo antes de promoverlo al siguiente entorno. Sin este espacio de prueba, los cambios se probarian directamente en produccion, con el riesgo que eso conlleva.

### Staging

El entorno de staging es una replica lo mas fiel posible de produccion, tanto en configuracion como en datos (utilizando datos anonimizados o sinteticos representativos). Su proposito es validar que un cambio funciona correctamente en un contexto que simula las condiciones reales de operacion antes de desplegarlo en produccion.

Staging es donde se descubren los problemas que el entorno de desarrollo no puede revelar: incompatibilidades entre servicios, problemas de rendimiento bajo carga real, conflictos de configuracion entre componentes que en desarrollo se ejecutan de forma aislada. Un cambio que funciona perfectamente en desarrollo puede fallar dramaticamente en staging si no se han considerado todas las dependencias del sistema.

> Un entorno de staging que difiere significativamente de produccion es peor que no tener staging, porque genera una falsa sensacion de seguridad. Si staging no replica las condiciones reales, los exitos en staging no garantizan nada en produccion.

### Produccion

Produccion es el entorno que da servicio a usuarios reales y cuyos datos tienen valor economico o legal. Los cambios que llegan a produccion deben haber sido validados exhaustivamente en los entornos anteriores. El proceso de promocion de un cambio a produccion debe ser un evento controlado, documentado y reversible.

La reversibilidad es especialmente importante en produccion. Si un cambio produce un efecto no deseado, el equipo debe poder volver al estado anterior de forma rapida y limpia. Esto implica que los despliegues deben ser atomicos en la medida de lo posible: o el cambio se aplica completamente o no se aplica nada, evitando estados intermedios inconsistentes.

## Validacion de cambios

### Pruebas automatizadas de infraestructura

Igual que el codigo tiene pruebas unitarias y de integracion, la infraestructura debe tener pruebas que validen que el estado del sistema es el esperado. Herramientas como InSpec, Goss o Testinfra permiten definir propiedades esperadas del sistema (que servicios estan activos, que puertos estan abiertos, que usuarios existen) y verificarlas automaticamente despues de cada despliegue.

Estas pruebas se ejecutan como parte del pipeline de despliegue y actuan como una barrera de calidad. Si un cambio de configuracion rompe una propiedad esperada del sistema, la prueba falla y el despliegue se detiene antes de afectar a produccion. Este enfoque convierte las implicitamente esperadas en explicitamente verificadas.

### Pruebas de regresion

Las pruebas de regresion validan que un cambio no ha alterado el comportamiento de componentes que no estaban siendo modificados. En infraestructura, es comun que una modificacion en un servicio tenga efectos colaterales en servicios dependientes. Por ejemplo, cambiar la configuracion de DNS de un servicio puede afectar a los clientes que se conectan a el, incluso si el cambio se consideraba local.

Las pruebas de regresion deben ejecutarse periodicamente, no solo despues de cambios, porque algunos problemas emergen gradualmente: certificados que se vencen, discos que se llenan, conexiones que degradan su rendimiento. La ejecucion periodica de pruebas de validacion permite detectar estos problemas antes de que provoquen incidentes.

## Ingenieria del caos basica

La ingenieria del caos es la disciplina de inyectar fallos controlados en un sistema para validar su capacidad de resistencia y recuperacion. En lugar de esperar a que un componente falle de forma inesperada en produccion, se provocan fallos de forma intencionada en entornos controlados para observar como responde el sistema y detectar debilidades antes de que se manifiesten en condiciones reales.

### Simulacion de fallos de red

Una de las practicas mas comunes es simular cortes de red entre componentes. Si un servidor de bases de datos pierde conectividad con la aplicacion, que ocurre? La aplicacion tiene mecanismos de reintento? Los datos se pierden o se bufferizan? El sistema se recupera automaticamente cuando la conectividad se restaura? Estas preguntas solo pueden responderse probando activamente el fallo, no analizando la configuracion.

### Simulacion de fallos de servicios

Eliminar o detener intencionadamente un servicio para observar como reacciona el sistema dependiente permite validar mecanismos de tolerancia a fallos como circuit breakers, timeouts y fallbacks. Si un servicio de cache cae, la aplicacion debe degradarse de forma elegante en lugar de fallar completamente. Este comportamiento solo se puede verificar sometiendo el sistema a esas condiciones exactas.

### Principio de seguridad en pruebas de caos

Las pruebas de caos deben ejecutarse siempre en entornos controlados y con mecanismos de rollback preparados. El objetivo es descubrir debilidades, no provocar incidentes reales. Antes de inyectar un fallo, el equipo debe tener claro que observar, como detener la prueba si las consecuencias son mayores de lo esperado y que acciones correctivas tomar en funcion de los resultados obtenidos.

