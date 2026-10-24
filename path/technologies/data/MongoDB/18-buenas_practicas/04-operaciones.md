# Operaciones

La operacion de MongoDB en produccion requiere una vigilancia constante, planes de contingencia probados y una comprension profunda de la capacidad del sistema. Una base de datos no configurada ni monitorizada es una bomba de tiempo: el fallo no es cuestion de si ocurrira, sino de cuando.

## Monitoreo continuo

Sin monitoreo, la primera noticia de un problema suele ser una alerta de los usuarios. El monitoreo proactivo mide metricas clave, establece umbrales y notifica antes de que el incidente afecte al servicio.

Las metricas fundamentales en MongoDB incluyen:

- **Operaciones por segundo**: lecturas, escrituras, actualizaciones, eliminaciones. Picos anomalos pueden indicar un ataque o un query ineficiente.
- **Cache de WiredTiger**: `wt cache` tracked bytes y `wt cache` pages read into cache. Una cache que se llena constantemente indica que la RAM es insuficiente.
- **Tiempo de operacion (latencia)**: percentiles p50, p95 y p99 de lecturas y escrituras. Incrementos sostenidos senalan problemas de indice, disco o contención.
- **Concurrencia**: `currentQueue` de lecturas y escrituras. Colas persistentes indican cuellos de botella en disco o bloqueos por escritura.
- **Estado del replica set**: `replSetGetStatus` muestra el lag de replicacion. Un lag creciente puede llevar a perdida de datos en un failover.

Prometheus combinado con el exporter de MongoDB (`mongodb_exporter`) y Grafana para visualizacion constituye la pila de monitoreo auto-gestionada mas extendida:

```yaml
# Prometheus config
scrape_configs:
   - job_name: 'mongodb'
     static_configs:
        - targets: ['localhost:9216']
```

Para equipos que prefieren una solucion gestionada, Atlas incluye monitoreo integrado con metricas historicas, alertas configurables y dashboards predefinidos. Independientemente de la herramienta, las alertas deben configurarse para notificar por PagerDuty, Slack o correo electronico.

> No monitorizar solo MongoDB. El monitoreo debe abarcar tambien el sistema operativo (CPU, RAM, E/S de disco) y la red (latencia, paquetes perdidos). Un problema de rendimiento en MongoDB suele tener su origen en el hardware subyacente.

## Backup probado

Un backup que no se ha probado no es un backup, es un deseo. La restauracion periodica en un entorno de pruebas es la unica forma de garantizar que el proceso funciona y que se cumple el objetivo de tiempo de recuperacion (RTO) y el objetivo de punto de recuperacion (RPO).

Las estrategias de backup recomendadas son:

- **mongodump/mongorestore**: herramienta nativa que exporta datos en formato BSON. Adecuada para bases de datos pequenas o medianas. No es atomica a nivel de base de datos: los cambios durante el dump pueden producir inconsistencias.

```bash
mongodump --uri="mongodb://usuario:clave@localhost:27017/mi_db" \
   --out=/backups/$(date +%Y%m%d_%H%M%S)
```

- **Backup de archivos (file system snapshot)**: copia de los archivos de datos de MongoDB a nivel de sistema de archivos, usando LVM, EBS snapshots o similares. Es atomico, rapido y no afecta al rendimiento si se realiza sobre un volumen separado.

- **Backup desde un secundario**: ejecutar el backup contra un nodo secundario del replica set elimina el impacto en el primario, pero el secundario debe estar sincronizado. Si el lag de replicacion es alto, el backup puede reflejar un estado desactualizado.

- **Atlas Backup**: en MongoDB Atlas, los backups continuos (con PITR habilitado) permiten restaurar a cualquier punto en los ultimos 7-30 dias segun el plan contratado, con minimo esfuerzo operativo.

```bash
# Restaurar un backup de mongodump
mongorestore --uri="mongodb://usuario:clave@localhost:27017" \
   /backups/20250725_120000
```

> Probar la restauracion al menos una vez al mes en un entorno no productivo. Documentar el procedimiento y los tiempos reales de restauracion.

## Plan de recuperacion ante desastres (DR)

Un plan de recuperacion ante desastres define como restaurar el servicio cuando un evento catastrofico inutiliza el datacenter principal: incendio, inundacion, fallo electrico prolongado o ciberataque.

La estrategia mas habitual en MongoDB es la replica set multi-AZ o multi-region. Un replica set con miembros distribuidos en tres zonas de disponibilidad (AZ) diferentes soporta la perdida de una zona completa sin perder el quorum ni la capacidad de elegir un nuevo primario.

Para escenarios que requieren resiliencia entre regiones geograficas, se configura un cluster con un replica set en la region primaria y un conjunto de replicas en una region secundaria. La aplicacion escribe en la region primaria y los datos se replican asincronamente a la secundaria.

Los parametros clave del plan de DR son:

- **RTO** (Recovery Time Objective): tiempo maximo aceptable desde la declaracion del desastre hasta la restauracion del servicio. Un RTO de 4 horas implica que el proceso de recuperacion no debe exceder ese limite.
- **RPO** (Recovery Point Objective): cantidad maxima de datos que se puede perder. Un RPO de 5 minutos significa que se tolera perder hasta 5 minutos de escrituras.
- **Estrategia active-passive**: el cluster secundario permanece inactivo hasta que se activa manualmente. Es mas economica pero el tiempo de conmutacion depende de la intervencion humana.
- **Estrategia active-active**: ambas regiones sirven trafico. Requiere resolver conflictos de escritura y es significativamente mas compleja.

El plan de DR debe documentarse, automatizarse en la medida de lo posible y probarse al menos dos veces al ano. Una prueba de DR consiste en simular un fallo del primario, verificar que el failover se produce correctamente, operar desde la region secundaria y luego revertir.

## Capacity planning

El capacity planning responde a la pregunta: cuanto hardware necesita el sistema actual y cuanto necesitara en los proximos 6-12 meses. Hacer esta estimacion evita dos problemas: quedarse sin recursos en un pico de trafico o pagar por capacidad ociosa.

Las variables a considerar son:

- **Tasa de crecimiento de datos**: cuantos GB nuevos ingresan al dia. Multiplicando por 365 se obtiene la proyeccion anual, a la que hay que sumar el overhead de indices (tipicamente 1.5x a 3x el tamano de los datos).
- **Operaciones por segundo**: pico historico y crecimiento esperado. El servidor debe soportar el pico con un margen de seguridad del 30-50 %.
- **Working set**: conjunto de datos e indices que caben en RAM. Si el working set excede la RAM disponible, MongoDB recurre al disco constantemente y el rendimiento se desploma. La regla general es que los indices mas los datos de acceso frecuente deben caber en RAM.
- **Throughput de disco**: las operaciones que no pueden servirse desde cache (escrituras, lecturas de datos frios) dependen de la capacidad de IOPS y rendimiento secuencial del almacenamiento. Discos SSD NVMe son practicamente obligatorios en produccion.

```javascript
// Estimar el tamano de indices
db.coleccion.aggregate([
   { $collStats: { storageStats: { } } }
])
```

Con estos datos, se dimensiona el servidor o cluster:

| Recurso | Ajuste tipico |
|---------|--------------|
| RAM | Suficiente para contener el working set + 20-30 % |
| vCPU | 4-16 segun concurrencia y operaciones por segundo |
| Disco | SSD NVMe, 3x el tamano de datos estimado |
| Red | 10 Gbps para clusters de mas de 3 nodos |

> El capacity planning no es un ejercicio unico. Debe revisarse trimestralmente y ajustarse segun la evolucion real del trafico y el volumen de datos.

