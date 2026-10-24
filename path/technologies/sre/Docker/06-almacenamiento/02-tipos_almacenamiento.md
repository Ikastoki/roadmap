# Tipos de almacenamiento en Docker

Docker ofrece tres mecanismos de almacenamiento para montar datos desde el exterior en un contenedor: volumenes, bind mounts y tmpfs. Cada uno responde a un caso de uso distinto y presenta diferencias fundamentales en cuanto a gestion, portabilidad, rendimiento y seguridad. La eleccion entre ellos depende de si los datos deben persistir tras la eliminacion del contenedor, si necesitan ser accesibles desde el host y si el requisito de velocidad prima sobre la durabilidad.

## Volumenes

Los volumenes son el mecanismo de almacenamiento recomendado por Docker para produccion. Son objetos gestionados directamente por el demonio de Docker, que los crea, almacena y elimina a traves de su API. Un volumen reside en el sistema de archivos del host bajo `/var/lib/docker/volumes/`, pero esta aislado del resto del sistema: ni siquiera el usuario root del host deberia manipular directamente su contenido, sino que debe hacerlo a traves de comandos de Docker.

```bash
# Crear y usar un volumen
docker volume create mi-volumen
docker run -v mi-volumen:/datos alpine touch /datos/archivo.txt
```

Los volumenes son portables entre sistemas: se pueden exportar, importar y trasladar de un host a otro. Son independientes del ciclo de vida del contenedor, por lo que sobreviven a su eliminacion. Ademas, pueden montarse en multiples contenedores simultaneamente, lo que los hace adecuados para patrones como el sidecar que escribe logs mientras la aplicacion principal los produce. Docker ofrece ademas drivers de volumen que permiten almacenar los datos en sistemas externos como NFS, CIFS, o servicios en la nube (EBS, EFS, Block Storage).

> Un volumen creado sin nombre recibe un identificador hexadecimal y se denomina volumen anonimo. Docker lo elimina automaticamente al eliminar el contenedor con `docker rm -v`, a menos que otro contenedor lo referencie.

## Bind mounts

Un bind mount monta un directorio o archivo concreto del sistema de archivos del host dentro del contenedor. A diferencia de los volumenes, Docker no gestiona el directorio montado: el contenido es el que existe en el host y cualquier modificacion dentro del contenedor se refleja inmediatamente en el host y viceversa. Esta transparencia los hace ideales para desarrollo, donde se puede montar el directorio del codigo fuente y ver los cambios en caliente sin reconstruir la imagen.

```bash
# Montar el directorio actual como bind mount
docker run -v "$(pwd)":/app -w /app node:20 npm run dev
```

El bind mount tiene dos limitaciones importantes. La primera es la portabilidad: la ruta del host es absoluta y depende del sistema donde se ejecuta, por lo que un archivo `docker-compose.yml` que use bind mounts no funcionara en otro equipo a menos que las rutas coincidan. La segunda es la seguridad: un bind mount expone cualquier directorio del host al contenedor, lo que puede ser aprovechado por un proceso comprometido dentro del contenedor para escalar privilegios en el host.

> La sintaxis moderna de Docker recomienda usar la opcion `--mount` en lugar de `-v` para bind mounts, ya que hace explicita la semantica: `--mount type=bind,source="$(pwd)",target=/app`.

## tmpfs

Un montaje tmpfs almacena los datos exclusivamente en la memoria RAM del host, sin escribir nunca en el disco. Cuando el contenedor se detiene, todo el contenido del tmpfs se pierde irreversiblemente. Este mecanismo no sirve para persistencia, sino para datos volatiles que requieren maxima velocidad de acceso o que por seguridad no deben escribirse en disco (credenciales temporales, tokens, claves criptograficas efimeras, cache de sesion).

```bash
# Montar un tmpfs de 64 MB en el contenedor
docker run --tmpfs /cache:rw,noexec,nosuid,size=64m alpine
```

tmpfs solo esta disponible en Linux, ya que depende de la funcionalidad del kernel del mismo nombre. En Windows y macOS no existe equivalente directo, aunque Docker Desktop en macOS utiliza la VM de Linux subyacente para ofrecer un comportamiento similar. Las opciones de montaje permiten controlar el tamano maximo (`size`), los permisos de ejecucion (`noexec`) y el comportamiento del bit setuid (`nosuid`).

| Caracteristica   | Volumenes | Bind mounts | tmpfs |
| ---------------- | --------- | ----------- | ----- |
| Persistencia     | Si        | Si          | No    |
| Gestion Docker   | Completa  | Ninguna     | Parcial |
| Portabilidad     | Alta      | Baja        | Nula  |
| Rendimiento      | Bueno     | Bueno       | Maximo |
| Seguridad        | Alta      | Media       | Alta  |
| Caso de uso      | Produccion | Desarrollo | Datos volatiles |

