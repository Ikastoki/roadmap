# Almacenamiento en la nube

El almacenamiento en la nube introduce una distincion fundamental que no existia de la misma forma en servidores fisicos: la separacion entre almacenamiento efimero (vinculado a la vida de la instancia) y almacenamiento persistente (independiente de ella). Esta separacion no es solo una caracteristica tecnica, sino un diseno arquitectonico que refleja como los proveedores optimizan costes, rendimiento y durabilidad. Comprender esta distincion es critico porque la decision de donde存放 los datos determina que ocurre cuando la instancia se detiene, falla o se elimina.

## Instance Store (almacenamiento efimero)

El instance store es almacenamiento fisico directamente conectado al servidor host donde se ejecuta la instancia. Ofrece la latencia mas baja posible porque los datos se leen y escriben directamente en los discos del host, sin pasar por la red. Sin embargo, los datos en el instance store estan vinculados a la vida util de la instancia: si la instancia se detiene, se reinicia o falla, todos los datos se pierden de forma permanente. No hay posibilidad de recuperacion.

El instance store es util para caches, buffers temporales y datos regenerables que no necesitan persistencia. Un servidor de cache como Redis puede utilizar el instance store para sus datos porque la informacion puede reconstruirse desde la base de datos de origen. Utilizar el instance store para datos criticos — como bases de datos de produccion o logs — es un error de diseno que garantiza perdida de datos ante cualquier evento de reinicio.

> El instance store funciona como la memoria RAM de un servidor fisico: es rapida, volatil y util para datos temporales. Confundirlo con almacenamiento persistente es el equivalente a guardar un documento importante solo en la RAM de un ordenador sin guardarlo en disco.

## Discos persistentes (EBS, Azure Disk)

Los discos persistentes son volumenes de almacenamiento independientes que se asocian logicamente a las instancias pero se almacenan fisicamente en la infraestructura de almacenamiento del proveedor, separada de los servidores de compute. Esto significa que un disco persistente puede desasociarse de una instancia y reasociarse a otra, manteniendo todos los datos intactos. Si la instancia falla o se elimina, el disco persistente continua existiendo y puede montarse en una nueva instancia en minutos.

AWS denomina a estos volumenes EBS (Elastic Block Store), Azure los llama Managed Disks y GCP Persistent Disks. Todos ofrecen diferentes tipos optimizados para distintos patrones de uso: discos magneticos (HDD) para cargas de trabajo secuenciales de bajo coste, discos SSD genericos para aplicaciones interactivas, y discos SSD provisionados con alto rendimiento (como io2 o UltraDisk) para bases de datos transaccionales con requisitos estrictos de IOPS.

## Backups y snapshots

Los snapshots son copias completas de un disco en un momento dado, almacenadas en el almacenamiento duradero del proveedor (tipicamente S3 en AWS). Crear un snapshot de un disco EBS genera una copia incremental: solo se almacenan los bloques que han cambiado desde el ultimo snapshot, lo que reduce tanto el tiempo de creacion como el coste de almacenamiento. Los snapshots son la base de la estrategia de backup en la nube y permiten restaurar discos completos a un punto en el tiempo especifico.

> Los snapshots deben automatizarse. Depender de que alguien recuerde crear snapshots manualmente es tan arriesgado como no hacer backups en un servidor fisico. Los proveedores ofrecen servicios como AWS Backup o Azure Backup que programan snapshots automaticamente segun politicas definidas por el cliente.

La durabilidad de los snapshots en la nube es extremadamente alta: AWS guaranteea un 99.999999999% (once nines) de durabilidad de objetos en S3, lo que equivale a una probabilidad insignificante de perdida de datos. Sin embargo, la durabilidad del snapshot no protege contra borrado accidental o malicioso, por lo que es recomendable activar protecciones como Object Lock o versionado en el bucket de almacenamiento.

