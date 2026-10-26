# Hardening del sistema operativo

El *hardening* es el proceso de reducir la superficie de ataque de un sistema operativo eliminando servicios innecesarios, aplicando parches de seguridad, configurando restricciones de acceso y endureciendo las configuraciones por defecto. El objetivo es que el sistema exponga el menor numero de vectores de ataque posible, manteniendo solo las funcionalidades estrictamente necesarias para su rol.

Un sistema sin endurecer es como un edificio con todas las puertas y ventanas abiertas: la meta no es construir muros inexpugnables, sino eliminar las oportunidades obvias de entrada.

## Eliminacion de servicios innecesarios

Todo servicio que se ejecuta en un sistema es un potencial vector de ataque. El principio de minimo privilegio implica ejecutar solo los servicios estrictamente necesarios:

```bash
# Listar servicios activos
systemctl list-units --type=service --state=running

# Deshabilitar un servicio
sudo systemctl disable --now cups  # impresion (si no se usa)

# Deshabilitar servicios de red innecesarios
sudo systemctl disable --now avahi-daemon  # descubrimiento mDNS
sudo systemctl disable --now rpcbind  # RPC (si no se usa NFS)
```

En un servidor web, por ejemplo, no deberian ejecutarse servicios de escritorio, impresion, o comparticion de archivos. Cada servicio eliminado reduce la superficie de ataque.

## Actualizaciones y parches

Mantener el sistema actualizado es una de las medidas mas efectivas. Los parches de seguridad corrigen vulnerabilidades conocidas que los atacantes explotan activamente:

```bash
# Debian/Ubuntu
sudo apt update && sudo apt upgrade -y

# RHEL/CentOS/Fedora
sudo dnf update -y

# Verificar actualizaciones de seguridad pendientes
sudo apt list --upgradable | grep security
```

Para servidores en produccion, es recomendable automatizar las actualizaciones de seguridad con `unattended-upgrades` (Debian/Ubuntu) o `dnf-automatic` (RHEL/Fedora) y reiniciar periodicamente para aplicar actualizaciones del kernel.

## Configuracion del firewall

Un firewall filtra el trafico de red entrante y saliente. En Linux, `iptables`, `nftables` o `ufw` son las herramientas mas comunes:

```bash
# UFW - interfaz simplificada
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# Ver reglas activas
sudo ufw status verbose
```

La estrategia recomendada es *deny all, allow some*: bloquear todo el trafico entrante por defecto y abrir solo los puertos estrictamente necesarios para el servicio que el servidor proporciona.

## SELinux y AppArmor

Ambos son sistemas de control de acceso obligatorio (*MAC*) que restringen las acciones que pueden realizar los procesos, incluso si son ejecutados como root:

**SELinux** (predeterminado en RHEL/CentOS/Fedora):

```bash
# Ver estado de SELinux
getenforce
sestatus

# Modos: Enforcing (aplica restricciones), Permissive (solo registra), Disabled
sudo setenforce 1  # Activar modo Enforcing
```

**AppArmor** (predeterminado en Ubuntu/Debian):

```bash
# Ver estado
sudo aa-status

# Generar perfil para una aplicacion
sudo aa-genprof /usr/sbin/nginx

# Poner perfil en modo enforce
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

## Proteccion de la pila y ASLR

Linux implementa varias protecciones de memoria a nivel de kernel:

```bash
# Verificar estado de las protecciones
cat /proc/sys/kernel/randomize_va_space  # ASLR (2 = completo)

# Proteccion de pila (Stack Protector)
cat /proc/sys/kernel/suid_dumpable  # 0 = deshabilitado, 1 = habilitado
```

- **ASLR (Address Space Layout Randomization):** aleatoriza la ubicacion de la pila, monticulo y bibliotecas en cada ejecucion, dificultando ataques que dependen de direcciones conocidas.
- **Stack Canary:** coloca un valor canario antes del retorno de funcion; si se sobrescribe por un buffer overflow, el kernel detecta la corrupcion.
- **NX bit (No-Execute):** marca las paginas de datos como no ejecutables, impidiendo la ejecucion de codigo inyectado.

## Desactivacion de root por SSH

La autenticacion remota de root es un vector de ataque comun. Es recomendable desactivar el login directo de root y forzar el uso de un usuario normal con sudo:

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no  # Solo autenticacion por clave publica
PubkeyAuthentication yes
MaxAuthTries 3
```

> El hardening no es un evento unico sino un proceso continuo. Los nuevos vectores de ataque aparecen regularmente, y las configuraciones seguras de ayer pueden ser vulnerables hoy. La auditoria periodica con herramientas como Lynis, OpenSCAP o CIS Benchmarks ayuda a mantener el nivel de seguridad adecuado.

