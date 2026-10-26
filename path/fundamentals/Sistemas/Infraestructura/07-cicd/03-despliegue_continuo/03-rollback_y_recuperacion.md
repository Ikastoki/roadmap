# Rollback y Recuperacion

La capacidad de revertir un despliegue no es un feature secundario del proceso de despliegue continuo, sino la condicion fundamental que lo hace seguro. Sin mecanismos de rollback, cada despliegue es un viaje sin retorno: si algo sale mal, la unica opcion es reparar el codigo defectuoso, testear la correccion y volver a desplegar, un proceso que puede tomar horas durante las cuales el sistema opera en un estado degradado. Con un rollback efectivo, la recuperacion ante un despliegue fallido se reduce a minutos o segundos, lo que transforma radicalmente el perfil de riesgo de la entrega continua.

La distincion entre despliegue continuo (continuous deployment) y despliegue manual es, en gran medida, la distincion entre sistemas con rollback automatico y sistemas sin el. La confianza para desplegar frecuentemente nace de la certeza de que, si algo falla, el sistema puede restaurar su estado anterior sin intervencion humana compleja.

## Rollback Automatico vs Manual

Un rollback automatico se activa cuando el sistema detecta que el nuevo despliegue no cumple criterios de salud predefinidos. Por ejemplo, si despues de un despliegue la tasa de errores HTTP 5xx supera un umbral, o si la latencia percentil 99 se dispara, el sistema invierte automaticamente el despliegue sin esperar intervencion humana. Este mecanismo requiere observabilidad integrada en el pipeline: metricas en tiempo real, comparadores contra baseline y un sistema de decision que pueda distinguir entre una anomalia transitoria y un fallo persistente.

El rollback manual ocurre cuando un operador decide, basandose en observaciones o alertas, que el despliegue debe revertirse. Este enfoque es mas lento pero mas flexible, ya que un humano puede evaluar contextos que un sistema automatico no puede capturar (por ejemplo, un pico de errores que coincide con un evento externo previsible). La desventaja es que el tiempo de respuesta depende de la disponibilidad del operador y de la claridad de la informacion presentada en los dashboards.

> Un rollback automatico sin umbrales bien calibrados es tan peligroso como no tenerlo: falsos positivos revierten despliegues validos, falsos negativos permiten que fallos persistan. La calibracion de estos umbrales es un proceso iterativo que requiere experiencia operativa.

En la practica, las organizaciones maduras implementan ambos mecanismos en capas: una primera capa automatica que detiene el despliegue o revierte si las metricas criticas se desvian, y una segunda capa manual que permite al equipo de operaciones tomar decisiones de reversal en situaciones mas complejas.

## Rollback en Blue-Green

La estrategia blue-green ofrece el rollback mas simple y robusto disponible. Dado que el entorno anterior permanece intacto y operativo durante el periodo de validacion del nuevo despliegue, revertir consiste simplemente en redirigir el trafico de vuelta al entorno anterior. No hay necesidad de reinstalar software, restaurar bases de datos o reiniciar servicios: el entorno anterior esta exactamente en el estado en que se dejo, esperando recibir trafico.

Esta inmediatez tiene un coste: mantener el entorno anterior operativo implica resources duplicados durante todo el periodo de coexistencia. Ademas, si el despliegue incluyo una migracion de base de datos, el rollback se complica significativamente, ya que la base de datos ahora esta en el nuevo esquema y el codigo antiguo podria no ser compatible con el. Por esta razon, las migraciones de base de datos en entornos blue-green deben disenarse para ser backward-compatible: agregar columnas en lugar de renombrarlas, crear tablas nuevas en lugar de modificar existentes.

## Rollback en Canary

En una despliegue canary, el rollback consiste en detener el envio de trafico a la version canary y dirigir todo el trafico a la version estable. Dado que la canary solo recibe un porcentaje reducido del trafico total, el impacto del fallo ya esta limitado por diseno. El rollback puede ser instantaneo si se implementa como un cambio en la configuracion del balanceador de carga o del service mesh.

La complejidad surge cuando la canary ha estado ejecutandose durante un periodo prolongado y los usuarios que la usaban han generado datos en formatos o esquemas que la version anterior no maneja. En sistemas con estado, el rollback no es solo una cuestion de redirigir trafico, sino de garantizar la compatibilidad de los datos producidos durante la ventana canary.

## Rollback en Rolling Update

Las rolling updates presentan el escenario mas complejo para el rollback, ya que durante el despliegue coexisten instancias de ambas versiones. Si se detecta un problema a mitad de la actualizacion, el orquestador puede detener el proceso y mantener las instancias que aun no fueron reemplazadas. Sin embargo, las instancias que ya fueron actualizadas con la nueva version estan ejecutando codigo potencialmente defectuoso y deben ser revertidas.

Kubernetes, por ejemplo, almacena los ReplicaSets anteriores y puede escalarlos de nuevo mientras reduce los de la version nueva. El tiempo de recuperacion depende de cuantas instancias fueron reemplazadas y de la velocidad de creacion de nuevas instancias. En un cluster con muchos nodos, este proceso puede tomar varios minutos, tiempo durante el cual parte del trafico sigue siendo servido por la version defectuosa.

## Preparacion para el Rollback

La preparacion para el rollback comienza antes del despliegue. Antes de cada despliegue, el sistema debe garantizar que el estado anterior es recuperable: backups de configuraciones, snapshots de bases de datos, registros de las versiones previas de los artefactos y las configuraciones que los acompanaban. Un despliegue sin este backup preventivo es como saltar sin paracaidas.

Las migraciones de base de datos merecen atencion especial. Toda migracion debe disenarse con la posibilidad de revertir en mente. Esto significa evitar operaciones destructivas (eliminar columnas, borrar datos) en una sola migracion, prefiriendo un enfoque de multiple pasos: primero agregar la nueva estructura, despues migrar los datos, y finalmente eliminar la antigua en una migracion posterior una vez que la version nueva esta estable.

Los pipeline de despliegue deben incluir pasos explicitos de validacion post-despliegue: smoke tests, verificacion de metricas basicas, comprobacion de conectividad con servicios dependientes. Estos checks actuan como la primera linea de defensa antes de que un despliegue defectuoso alcance a todos los usuarios. Si fallan, el pipeline puede ejecutar el rollback automaticamente sin intervencion humana.

## Rollback como Cultura

La capacidad de rollback influye profundamente en la cultura de ingenieria de un equipo. Cuando el rollback es rapido, seguro y frecuentemente practicado, los ingenieros se sienten comodos desplegando cambios con confianza, sabiendo que cualquier error es reversible. Cuando el rollback es lento, arriesgado o inexistente, los equipos naturalmente se vuelven conservadores: reducen la frecuencia de despliegues, aumentan las capas de aprobacion manual y retrasan la entrega.

Esta dinamica crea un circulo virtuoso o vicioso. Equipos que despliegan frecuentemente acumulan experiencia rapida en rollback, lo que les permite desplegar con mas confianza. Equipos que rara vez despliegan nunca aprenden a manejar rollback en situaciones reales, lo que les hace desplegar con mas miedo. La practica regular de despliegues (incluso de cambios triviales) es la forma mas efectiva de construir esta capacidad organica.

