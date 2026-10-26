# Estrategias de Despliegue

La manera en que una aplicacion se pone en manos de los usuarios determina el nivel de riesgo, la velocidad de entrega y la capacidad de respuesta ante fallos. No existe una estrategia unica que sea superior en todos los escenarios: cada una equilibra de forma diferente la disponibilidad, el coste de recursos, la complejidad operativa y la rapidez con la que los usuarios reciben cambios. Elegir la estrategia adecuada implica comprender no solo como funciona cada mecanismo, sino tambien en que circunstancias sus compromisos resultan aceptables.

La experiencia acumulada en operaciones de produccion demuestra que los despliegues problematicos rara vez se deben a un error en el codigo desplegado, sino a una decision inadecuada sobre como introducirlo. Un despliegue total e inmediato de una funcionalidad defectuosa puede provocar un corte de servicio generalizado, mientras que una introduccion gradual permite detectar problemas antes de que afecten a toda la base de usuarios.

## Blue-Green

La estrategia blue-green mantiene dos entornos de produccion identicos: uno activo (azul) y otro inactivo (verde). Cuando se realiza un despliegue, el nuevo codigo se despliega al entorno inactivo, se verifica su funcionamiento y se redirige el trafico de golpe desde el entorno anterior al nuevo. El mecanismo subyacente es un interruptor de trafico que apunta a un balanceador de carga o proxy inverso; al cambiar la direccion de destino, todos los usuarios pasan a la nueva version de forma instantanea.

La principal ventaja es la reversibilidad inmediata: si algo falla, basta con redirigir el trafico de vuelta al entorno anterior que sigue exactamente en el estado en que se dejo. Sin embargo, este modelo requiere mantener duplicada toda la infraestructura de produccion durante el despliegue, lo que implica un coste de recursos significativo. Para aplicaciones con bases de datos, la migracion del esquema se convierte en el punto critico, ya que ambas versiones del codigo deben ser compatibles con la version de la base de datos durante la transicion.

> Blue-green es ideal para sistemas donde la disponibilidad es critica y el coste adicional de infraestructura justificable, como servicios financieros o plataformas de comercio electronico con alto volumen de transacciones.

## Canary

La estrategia canary introduce la nueva version gradualmente, dirigiendo inicialmente una fraccion pequena del trafico (por ejemplo, el 5%) hacia ella. Si las metricas de error, latencia y comportamiento de negocio se mantienen dentro de umbrales aceptables, se incrementa progresivamente el porcentaje hasta alcanzar el 100%. El nombre proviene de la practica minera de canario en la mina: si el canario sobrevive, la atmosfera es segura.

Este enfoque permite detectar problemas en produccion con un impacto limitado. Las herramientas de observabilidad (metricas, logs, trazas) se vuelven esenciales para automatizar la decision de continuar o detener el despliegue. La complejidad radica en gestionar simultaneamente dos versiones del codigo en produccion, lo que puede generar incompatibilidades si la nueva version depende de cambios en la base de datos o en APIs que todavia estan sirviendo a la version anterior.

A diferencia de blue-green, canary no requiere duplicar la infraestructura completa, solo el porcentaje adicional de instancias que recibiran trafico de prueba. Esto lo hace mas economico pero tambien mas dificil de automatizar, ya que el criterio de "exito" debe definirse con precision en terminos de metricas observables.

## Rolling Update

Una rolling update realiza la sustitucion progresiva de instancias antiguas por nuevas, manteniendo siempre un numero minimo de replicas funcionando. El orquestador (Kubernetes, por ejemplo) reemplaza un subconjunto de pods, espera a que esten listos y disponibles, y luego continua con el siguiente lote hasta completar la actualizacion. En cualquier momento durante el proceso, el sistema opera con una mezcla de versiones anteriores y nuevas.

Esta estrategia es el compromiso natural entre el coste de blue-green y la complejidad de canary. No requiere infraestructura adicional significativa y permite un despliegue progresivo sin necesidad de un sistema sofisticado de direccionamiento de trafico. Sin embargo, si la nueva version introduce un cambio incompatible a nivel de protocolo o datos, las instancias antiguas y nuevas pueden entrar en conflicto durante la ventana de transicion.

El factor critico en rolling update es la definicion de `maxSurge` y `maxUnavailable`, que controlan cuantas instancias nuevas pueden existir simultaneamente por encima del replicaset deseado y cuantas pueden estar no disponibles durante el despliegue. Una configuracion conservadora garantiza disponibilidad pero ralentiza el proceso; una agresiva lo acelera pero aumenta el riesgo.

## Feature Flags

Los feature flags desacoplan el despliegue del codigo de la activacion de funcionalidades. El codigo nuevo se despliega a produccion con una bandera que lo mantiene desactivado; cuando se desea liberar la funcionalidad, se cambia la configuracion del flag sin necesidad de un nuevo despliegue. Este patron permite separar la tarea tecnica de desplegar de la tarea de negocio de lanzar.

La potencia de los feature flags radica en la capacidad de realizar pruebas A/B, despliegues oscuros (dark launches) y liberaciones graduales sin infraestructura adicional. Tambien facilita la recuperacion ante incidentes: desactivar un flag es instantaneo y no requiere redeploy. Sin embargo, el acumulo de flags sin una politica de limpieza genera deuda tecnica significativa, ya que el codigo con condicionales por flag se vuelve cada vez mas dificil de mantener y testear.

> Un feature flag sin fecha de expiracion es una bomba de complejidad retardada. Cada flag activo multiplica las combinaciones posibles que el sistema debe soportar, aumentando exponencialmente el espacio de pruebas.

## Comparativa y Compromisos

| Estrategia | Velocidad de despliegue | Coste de recursos | Complejidad operativa | Rollback | Riesgo por fallo |
|---|---|---|---|---|---|
| Blue-green | Inmediata | Alto (duplica infra) | Baja | Instantaneo | Nulo |
| Canary | Gradual | Moderado | Alta | Rápido | Limitado |
| Rolling update | Progresiva | Bajo | Moderada | Moderado | Moderado |
| Feature flags | Inmediata | Nulo adicional | Baja-Moderada | Instantaneo | Nulo |

La decision entre estrategias no es binaria; en la practica, las organizaciones combinan varias. Un despliegue canary puede usar feature flags para controlar que funcionalidades se activan, y un rolling update en Kubernetes puede complementarse con un canary manual para cambios criticos. La clave esta en que la infraestructura de despliegue soporte la estrategia elegida y que el equipo tenga la madurez operativa para gestionarla.

