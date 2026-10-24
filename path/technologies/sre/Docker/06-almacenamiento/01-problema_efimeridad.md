# El problema de la efimeridad

Un contenedor, por diseño, es una entidad efimera. Cuando se crea, Docker anade una capa de escritura sobre las capas de solo lectura de la imagen, y todos los cambios que el proceso dentro del contenedor realiza (archivos creados, bases de datos modificadas, logs generados) se almacenan exclusivamente en esa capa. Al eliminar el contenedor con `docker rm`, esa capa de escritura se descarta por completo y los datos desaparecen sin posibilidad de recuperacion. Este comportamiento no es un defecto, sino una consecuencia deliberada del modelo de inmutabilidad que hace a los contenedores tan predecibles y faciles de replicar: cualquier instancia de una imagen produce exactamente el mismo punto de partida.

El problema se manifiesta de forma evidente cuando un contenedor ejecuta una base de datos. Supongase un contenedor de PostgreSQL que lleva semanas en produccion acumulando registros. Si el contenedor se detiene y se elimina (por una actualizacion, un fallo del nodo o una reconfiguracion), toda la informacion almacenada se pierde a menos que se haya utilizado algun mecanismo de persistencia externo. Lo mismo ocurre con aplicaciones que almacenan archivos subidos por usuarios, generan reportes, mantienen colas de mensajes o escriben logs de auditoria.

```bash
# Demostracion del problema: los datos desaparecen al eliminar el contenedor
docker run --name prueba-efimera alpine sh -c "echo 'datos importantes' > /tmp/datos.txt && cat /tmp/datos.txt"
docker rm prueba-efimera
# Si se vuelve a ejecutar otro contenedor, /tmp/datos.txt no existe
```

La capa de escritura del contenedor presenta ademas problemas de rendimiento y aislamiento. Al estar acoplada al ciclo de vida del contenedor, no puede ser compartida por multiples contenedores. Su rendimiento de E/S es inferior al de los volumenes gestionados por Docker, especialmente en operaciones de escritura sincrona. Y al residir en el sistema de archivos del host dentro del directorio de Docker (`/var/lib/docker/`), no es facilmente accesible para tareas de backup o inspeccion directa.

> La efimeridad no es inherentemente negativa. De hecho, es una ventaja para cargas de trabajo stateless como servidores web o workers de procesamiento por lotes, donde cada instancia es intercambiable y cualquier estado se delega a un servicio externo (base de datos, cache, almacenamiento de objetos).

La solucion al problema de la efimeridad consiste en separar el estado del computo: el codigo y las dependencias residen en la imagen (inmutable), mientras que los datos persistentes se almacenan fuera del contenedor mediante volumenes, bind mounts o tmpfs. Esta separacion no solo preserva los datos ante la eliminacion del contenedor, sino que permite que diferentes contenedores accedan simultaneamente a los mismos datos, que estos se respalden de forma independiente y que el almacenamiento se escale por separado de la capacidad de computo.

