# Scripts Shell para Automatizacion de Despliegue

La administracion de servidores comienza con tareas repetitivas: instalar dependencias, crear usuarios, copiar archivos de configuracion, iniciar servicios. Un administrador que ejecuta estos pasos manualmente en cada servidor esta invirtiendo tiempo en algo que un script podria hacer en segundos. Los scripts shell, escritos en bash, son la herramienta mas inmediata para automatizar estas tareas. No requieren infraestructura adicional, no necesitan agentes instalados en los servidores objetivo y pueden ejecutarse desde cualquier maquina con acceso SSH. Sin embargo, un script mal disenado puede causar tantos problemas como resolver: si cada ejecucion crea duplicados, sobreescribe configuraciones validas o falla sin mensajes claros, el administrador termina perdiendo mas tiempo depurando el script que ejecutandolo manualmente. La clave esta en escribir scripts que sean **idempotentes**: ejecutables multiples veces con el mismo resultado predecible.

## Por que la idempotencia importa

Un script que crea un usuario con `useradd` fallara en la segunda ejecucion porque el usuario ya existe. Un script que anexa una linea a `/etc/fstab` duplicara esa linea en cada ejecucion. Estos problemas son criticos en entornos de produccion donde los scripts se ejecutan periodicamente o donde un despliegue puede reiniciarse tras un fallo parcial. La idempotencia garantiza que ejecutar el script una vez o cien veces produce exactamente el mismo estado del sistema. Esto no es un concepto abstracto: es la diferencia entre un despliegue robusto y uno que requiere intervencion manual constante.

## Estrategias para lograr idempotencia

### Comprobaciones previas antes de cada accion

El patron mas basico es verificar el estado actual antes de actuar. En bash, esto se traduce en comprobar si un usuario existe antes de crearlo, si un paquete esta instalado antes de instalarlo, o si un archivo ya tiene el contenido deseado antes de sobreescribirlo.

```bash
#!/bin/bash
USUARIO="deploy"
if ! id "$USUARIO" &>/dev/null; then
    useradd -m -s /bin/bash "$USUARIO"
    echo "Usuario $USUARIO creado"
else
    echo "Usuario $USUARIO ya existe, omitiendo"
fi
```

El operador `id` devuelve exito si el usuario existe. La redireccion `&>/dev/null` suprime la salida, y el bloque `else` informa sin generar error. Este patron se replica para cualquier recurso del sistema: paquetes, archivos, servicios, puertos.

### Uso de comandos que verifican estado automaticamente

Herramientas como `ansible` implementan idempotencia internamente, pero en bash puro se pueden usar comandos con esta propiedad natural. Por ejemplo, `grep -q` comprueba si una linea existe en un archivo sin modificarlo:

```bash
grep -q "PermitRootLogin no" /etc/ssh/sshd_config || \
    sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```

El operador `||` ejecuta la segunda parte solo si `grep` fallo (la linea no existe o no coincide). Esto evita duplicar entradas en configuraciones.

### Templates con diff antes de aplicar

En lugar de sobreescribir archivos de configuracion directamente, se puede generar una version temporal y compararla con la existente. Si son identicos, no se realiza ningun cambio. Si difieren, se aplica la nueva version y se notifica el cambio.

```bash
TMPFILE=$(mktemp)
envsubst < config.template > "$TMPFILE"
if ! diff -q "$TMPFILE" /etc/app/config.conf &>/dev/null; then
    cp "$TMPFILE" /etc/app/config.conf
    systemctl reload app
    echo "Configuracion actualizada"
fi
rm -f "$TMPFILE"
```

`envsubst` sustituye variables de entorno en el template, `diff -q` compara sin mostrar diferencias (solo exito/fallo), y el servicio solo se recarga si la configuracion cambio realmente.

### Control de errores con set

Un script sin control de errores puede dejar el sistema en un estado intermedio: paquetes parcialmente instalados, usuarios sin grupos asignados, servicios reiniciados con configuracion incompleta. La directiva `set -euo pipefail` convierte cualquier error en una terminacion inmediata del script.

```bash
#!/bin/bash
set -euo pipefail
```

- `set -e`: termina el script si cualquier comando retorna un codigo distinto de cero.
- `set -u`: trata variables no definidas como errores (evita bugs silenciosos).
- `set -o pipefail`: propaga el codigo de error a traves de tuberias, de modo que `cmd1 | cmd2` falla si falla `cmd1`.

## Patrones comunes en scripts de despliegue

### Instalacion condicional de paquetes

```bash
PKGManager=$(command -v apt-get || command -v yum || command -v dnf)
if [ -z "$PKGManager" ]; then
    echo "Gestor de paquetes no soportado" >&2
    exit 1
fi

install_pkg() {
    local pkg="$1"
    if ! rpm -q "$pkg" &>/dev/null && ! dpkg -l "$pkg" &>/dev/null 2>&1; then
        $PKGManager install -y "$pkg"
    fi
}
```

Esta funcion abstracte el gestor de paquetes y verifica si el paquete ya esta instalado antes de intentar instalarlo. El script funciona en distribuciones Debian y Red Hat sin modificar.

### Creacion de estructura de directorios

```bash
create_dirs() {
    local dirs=("/etc/app" "/var/log/app" "/var/lib/app")
    for dir in "${dirs[@]}"; do
        mkdir -p "$dir"           # -p no falla si ya existe
        chown appuser:appuser "$dir"
        chmod 755 "$dir"
    done
}
```

`mkdir -p` es idempotente por diseno: crea el directorio completo si no existe, o no hace nada si ya existe. Los permisos se aplican siempre, lo cual es seguro porque son valores fijos.

### Gestion de servicios

```bash
enable_service() {
    local svc="$1"
    if systemctl is-enabled --quiet "$svc" 2>/dev/null; then
        echo "Servicio $svc ya habilitado"
    else
        systemctl enable "$svc"
    fi
    if ! systemctl is-active --quiet "$svc"; then
        systemctl start "$svc"
    fi
}
```

La funcion verifica el estado actual del servicio antes de actuar. Un servicio ya habilitado y en ejecucion no recibe comandos redundantes.

## Estructura recomendada de un script de despliegue

Un script de produccion deberia seguir una estructura clara: definir variables al inicio, verificar prerequisitos, ejecutar pasos en orden logico, y reportar el resultado final. Los logs deben ser claros y timestamps para facilitar la depuracion.

```bash
#!/bin/bash
set -euo pipefail

LOGFILE="/var/log/deploy-$(date +%Y%m%d-%H%M%S).log"
exec > >(tee -a "$LOGFILE") 2>&1

echo "=== Inicio de despliegue: $(date) ==="

# Verificaciones previas
[[ $EUID -ne 0 ]] && { echo "Ejecutar como root" >&2; exit 1; }

# Pasos de despliegue
install_dependencies
create_user
deploy_application
configure_service

echo "=== Despliegue completado: $(date) ==="
```

La redireccion con `tee` duplica la salida tanto a terminal como a archivo de log. La verificacion del usuario root al inicio evita fallos silenciosos por falta de permisos.

## Limitaciones de los scripts shell

Los scripts shell son adecuados para automatizacion simple en uno o pocos servidores. Cuando la infraestructura crece a docenas o cientos de servidores, surgen problemas que bash no resuelve nativamente: ejecucion paralela, gestion de inventario, dependencias entre tareas, validacion de estado post-despliegue. En esos casos, herramientas de gestion de configuracion como Ansible o Puppet ofrecen modelos declarativos donde se describe el estado deseado del sistema y la herramienta calcula los pasos necesarios para alcanzarlo. Los scripts shell siguen siendo utiles como complemento, pero no como solucion unica en entornos complejos.

