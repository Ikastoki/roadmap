# Copias de seguridad

Una copia de seguridad que nunca se ha restaurado es una suposicion, no una garantia. El proposito de un backup no es tener un archivo compress en algun lugar remoto, sino garantizar que los datos pueden recuperarse cuando algo falla: disco defectuoso, error humano, ransomware o desastre fisico. La diferencia entre un incidente molesto y una catastrofe business radica en la existencia y testabilidad de las copias de seguridad.

## La regla 3-2-1

La regla 3-2-1 es un principio fundamental de estrategia de backups que establece tres requisitos basicos: mantener al menos tres copias de los datos, almacenadas en al menos dos medios diferentes, con al menos una copia fuera del sitio fisico. Cada componente de esta regla responde a un riesgo especifico.

Las tres copias protegen contra la perdida de una unica copia. Si solo hay una copia y el disco falla, los datos se pierden. Con tres copias, se necesita que dos fallos simultaneos —o una cadena de fallos— provoquen la perdida total, lo que estadisticamente es mucho menos probable.

Los dos medios diferentes protegen contra fallos sistemicos. Si ambas copias estan en discos del mismo modelo, un defecto de fabricacion puede afectarlas a las dos. Si una copia esta en disco local y otra en cinta, la probabilidad de que ambos medios fallen simultaneamente es minima. En la practica, esto significa combinar disco local con almacenamiento en red, NAS, cinta o nube.

La copia offsite protegne contra desastres fisicos: incendio, inundacion, robo o fallo electrico que afecte al edificio completo. Un backup en la misma habitacion que el servidor no sirve si hay un incendio. La copia remota puede ser un servicio de nube, un disco en otra ubicacion fisica o una replicacion a otro datacenter.

> La regla 3-2-1 es un minimo absoluto. Para datos criticos, se recomienda la variante 3-2-1-1-0: las tres copias, dos medios, una offsite, una copia immutable (que no pueda ser modificada ni eliminada, protegiendo contra ransomware), y cero errores en la verificacion de integridad del backup.

## Herramientas de backup

**rsync** es la herramienta clasica de sincronizacion y backup en Unix. Su fortaleza es la eficiencia en transferencias: utiliza un algoritmo delta que transfiere solo las partes de los archivos que han cambiado, lo que lo hace ideal para backups incrementales sobre redes. `rsync` no gestiona snapshots ni eliminacion automatica de versiones antiguas; es una herramienta de sincronizacion, no de gestion de backups. Su simplicidad es tanto su mayor ventaja como su limitacion: para backups basicos es excelente, pero carece de funcionalidades avanzadas como compresion inteligente o deduplicacion.

**BorgBackup** representa un enfoque mas moderno. Ofrece deduplicacion a nivel de bloque, compresion, encriptacion y gestion de repositorios con politicas de retencion. La deduplicacion significa que si muchos archivos contienen los mismos bloques de datos, solo se almacena una copia de cada bloque, reduciendo drasticamente el espacio necesario. Borg es especialmente eficiente para backups de servidores donde muchos archivos cambian poco entre versiones. Su modelo de repositorio con snapshots permite restaurar el estado del sistema en cualquier punto del tiempo.

**Restic** comparte filosofia con Borg pero esta escrito en Go, lo que lo hace mas portable y con soporte nativo para multiples backends: S3, Azure, Google Cloud, SFTP y mas. Restic tambien ofrece deduplicacion, encriptacion y gestion de snapshots. Su ventaja sobre Borg es la integracion nativa con almacenamiento en nube, lo que facilita la implementacion de la regla 3-2-1 sin necesidad de herramientas adicionales de sincronizacion.

```bash
# Borg: crear un backup con retencion de 7 dias diarios, 4 semanales, 6 mensuales
borg create --stats --progress \
  /mnt/backup::'{now:%Y-%m-%d-%H:%M}' \
  /home /etc /var/www

borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 \
  /mnt/backup
```

> La eleccion entre estas herramientas depende del contexto: `rsync` para sincronizaciones simples, Borg para backups en servidores con almacenamiento local o NAS, Restic cuando el backend principal es nube.

## La importancia de testear la restauracion

El backup mas valioso es aquel que se ha restaurado exitosamente al menos una vez. Sin pruebas de restauracion, un backup es un archivo cuyo contenido y integridad son desconocidos hasta el momento critico en que se necesita. Solo entonces se descubre que el archivo estaba corrupto, que la clave de encriptacion se perdio, que el formato del backup no es compatible con la version actual del software, o que la restauracion tarda doce horas cuando el negocio necesita los datos en una hora.

Las pruebas de restauracion deben ser regulares y documentadas. Un plan basico incluye: restaurar un archivo aleatorio semanalmente para verificar integridad, restaurar un directorio completo mensualmente para verificar el proceso, y realizar una restauracion completa del servidor en un entorno aislado trimestralmente. Cada prueba debe documentar el tiempo necesario, los pasos seguidos y los problemas encontrados.

El concepto de Recovery Point Objective (RPO) y Recovery Time Objective (RTO) guia estas decisiones. RPO define cuantos datos puede permitirse perder el negocio (si el RPO es 24 horas, un backup diario es suficiente; si es 1 hora, se necesitan backups cada hora). RTO define cuanto tiempo puede permanecer el sistema fuera de servicio (si el RTO es 4 horas, la restauracion debe completarse en ese tiempo). Ambos valores determinan no solo la frecuencia de backups, sino tambien la infraestructura necesaria para la restauracion.

