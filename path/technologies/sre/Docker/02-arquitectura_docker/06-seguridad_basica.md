# Seguridad basica en Docker

La seguridad en Docker no es un aspecto que se pueda anadir a posteriori; debe integrarse desde el diseno de las imagenes, la configuracion del demonio y la operacion de los contenedores. El modelo de seguridad de los contenedores es inherentemente diferente al de las maquinas virtuales: al compartir el kernel del anfitrion, una vulnerabilidad de escalado de privilegios en el kernel puede comprometer no solo un contenedor, sino todo el sistema. Comprender los mecanismos de seguridad disponibles y aplicarlos correctamente es esencial para operar Docker de forma segura, tanto en desarrollo como en produccion.

## Ejecucion como usuario no root

La regla de seguridad mas importante en Docker es no ejecutar procesos como root dentro del contenedor. Si bien el proceso dentro del contenedor se ejecuta como root en su namespace de usuario, en el anfitrion ese mismo proceso tiene los mismos privilegios que el usuario que ejecuta Docker.

```dockerfile
# Mal: el proceso se ejecuta como root
FROM node:20
COPY . /app
CMD ["node", "server.js"]

# Bien: crear y usar un usuario no privilegiado
FROM node:20
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
COPY . /app
CMD ["node", "server.js"]
```

```bash
# Verificar que el contenedor no se ejecuta como root
docker run --rm -u 1000:1000 alpine whoami
# 1000 (usuario no root)
```

Si el proceso dentro del contenedor se ejecuta como root y un atacante consigue escapar del contenedor, los procesos resultantes en el anfitrion tambien seran root. Usar un usuario no privilegiado dentro del contenedor limita el dano potencial.

## Linux capabilities

Las capabilities de Linux dividen los privilegios de root en unidades atomicas. Docker elimina por defecto todas las capabilities excepto las minimas necesarias para el funcionamiento basico. Esto limita lo que un proceso root dentro del contenedor puede hacer realmente.

```bash
# Ver las capabilities por defecto de un contenedor
docker run --rm alpine cat /proc/1/status | grep Cap
# CapInh: 00000000a80425fb
# CapPrm: 00000000a80425fb
# CapEff: 00000000a80425fb
# CapBnd: 00000000a80425fb
# CapAmb: 0000000000000000

# Decodificar las capabilities
capsh --decode=00000000a80425fb
# 0x00000000a80425fb = cap_chown,cap_dac_override,cap_fowner,
# cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,
# cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,
# cap_audit_write,cap_setfcap
```

Para la mayoria de las aplicaciones, las capabilities por defecto son suficientes. Si una aplicacion necesita privilegios adicionales, deben anadirse explicitamente en lugar de ejecutar el contenedor con `--privileged`.

```bash
# Anadir capabilities especificas (recomendado)
docker run --cap-add=NET_ADMIN --cap-add=SYS_TIME nginx

# Evitar: --privileged otorga todas las capabilities y acceso a dispositivos
docker run --privileged nginx  # No recomendado
```

El flag `--privileged` desactiva casi todos los mecanismos de seguridad de Docker. Solo debe usarse en casos excepcionales y perfectamente justificados.

## Perfiles de Seccomp

Seccomp (Secure Computing Mode) filtra las llamadas al sistema que un proceso puede realizar. Docker aplica un perfil por defecto que permite las llamadas esenciales (alrededor de 300 de las mas de 400 disponibles en Linux) y bloquea las peligrosas.

```bash
# Verificar que seccomp esta activo
docker info | grep seccomp
# seccomp: enabled

# Ejecutar con un perfil seccomp personalizado
docker run --security-opt seccomp=/path/to/profile.json nginx

# Ejecutar sin seccomp (solo para depuracion)
docker run --security-opt seccomp=unconfined nginx
```

Un perfil seccomp personalizado permite bloquear llamadas al sistema especificas. Por ejemplo, una aplicacion web que nunca necesita ejecutar `mount` o `reboot` puede tener un perfil restrictivo que bloquee estas llamadas.

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["mount", "umount2", "reboot", "kexec_load"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

## AppArmor y SELinux

AppArmor (en distribuciones Debian/Ubuntu) y SELinux (en distribuciones Red Hat/CentOS/Fedora) son modulos de seguridad del kernel de Linux que implementan control de acceso obligatorio (MAC). Docker puede integrarse con estos sistemas para anadir una capa adicional de seguridad.

```bash
# Verificar si AppArmor esta activo
sudo aa-status

# Docker aplica un perfil AppArmor por defecto
docker run --rm alpine cat /proc/1/attr/current
# docker-default (enforce)

# Ejecutar con un perfil AppArmor personalizado
docker run --security-opt apparmor=mi-perfil nginx

# Ejecutar sin AppArmor (no recomendado)
docker run --security-opt apparmor=unconfined nginx
```

El perfil AppArmor por defecto de Docker restringe operaciones como escritura en /proc, montaje de sistemas de archivos y acceso a dispositivos. Un perfil personalizado puede ajustar estos permisos segun las necesidades de la aplicacion.

## No root mode en el demonio

Docker ofrece el modo "rootless", donde el demonio se ejecuta sin privilegios de root. En este modo, tanto el demonio como los contenedores se ejecutan en el espacio de usuario del usuario que inicio el demonio.

```bash
# Instalar el modo rootless
sudo apt-get install -y docker-ce-rootless-extras

# Iniciar el demonio rootless
dockerd-rootless-setuptool.sh install
systemctl --user start docker

# En modo rootless, los contenedores se ejecutan sin privilegios
# El mapeo de puertos <1024 no funciona (requiere privilegios)
# Se usa slirp4netns para la red
```

El modo rootless reduce significativamente el impacto de una vulnerabilidad en Docker, ya que incluso si un atacante compromete el demonio, solo obtiene privilegios de usuario, no de root. Sin embargo, tiene limitaciones: no soporta el mapeo de puertos inferiores a 1024 y tiene restricciones en la configuracion de red.

## Lectura del sistema de archivos

Para contenedores que no necesitan escribir en su sistema de archivos, se puede montar el sistema de archivos como solo lectura. Esto evita que un atacante modifique archivos del sistema dentro del contenedor.

```bash
# Montar el sistema de archivos del contenedor como solo lectura
docker run --read-only nginx

# Si la aplicacion necesita escribir en directorios especificos,
# montar volumenes temporales para esos directorios
docker run --read-only \
  -v /tmp \
  -v /var/run \
  nginx

# Verificar que el sistema de archivos es solo lectura
docker run --rm --read-only alpine touch /test
# touch: /test: Read-only file system
```

La combinacion de `--read-only` con volumenes para directorios de escritura especificos es una practica de seguridad recomendada que limita la superficie de ataque.

## Limitar recursos como medida de seguridad

Los limites de recursos no solo evitan que un contenedor degrade el rendimiento de otros, sino que tambien mitigan ataques de denegacion de servicio. Un contenedor comprometido que intente consumir toda la memoria o CPU sera detenido por los cgroups.

```bash
# Limitar recursos para mitigar ataques DoS
docker run -d \
  --memory="256m" \
  --memory-swap="512m" \
  --cpus="0.5" \
  --pids-limit=100 \
  --restart=on-failure:5 \
  nginx
```

Las opciones de limitacion relevantes para seguridad incluyen:
- `--pids-limit`: Limita el numero de procesos, evitando ataques de fork bomb
- `--memory`: Limita la memoria disponible
- `--cpus`: Limita el uso de CPU
- `--ulimit nofile=1024:1024`: Limita el numero de archivos abiertos

## No exponer el socket de Docker

La regla de seguridad mas importante en la operacion de Docker es no exponer el socket de Docker `/var/run/docker.sock` dentro de los contenedores. El socket otorga control total sobre el demonio Docker.

```bash
# Esto es EXTREMADAMENTE PELIGROSO:
docker run -v /var/run/docker.sock:/var/run/docker.sock alpine

# Dentro del contenedor, cualquiera puede ejecutar comandos Docker
# con los mismos privilegios que el usuario del anfitrion
apk add --no-cache docker-cli
docker ps  # Funciona! Ve los contenedores del anfitrion
docker run --privileged alpine  # Puede ejecutar contenedores privilegiados
```

Si es absolutamente necesario que un contenedor interactue con el demonio Docker (por ejemplo, una herramienta de CI/CD), se debe limitar el acceso mediante autorizacion TLS y, preferiblemente, usando la API de Docker con tokens temporales en lugar del socket.

> La seguridad en Docker es un proceso continuo, no una configuracion unica. Requiere mantenerse actualizado con las vulnerabilidades del kernel y de Docker, revisar periodicamente las configuraciones de seguridad y aplicar el principio de minimo privilegio. Un contenedor debe tener solo los permisos estrictamente necesarios para su funcion, ni uno mas.

