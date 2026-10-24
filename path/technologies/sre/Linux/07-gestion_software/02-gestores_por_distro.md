# Gestores de paquetes por distribucion

Cada distribucion de Linux implementa su propio gestor de paquetes, pero todos comparten el mismo principio fundamental: mantener un inventario de software instalado, permitir consultas sobre paquetes disponibles, resolver dependencias automaticamente y gestionar actualizaciones de forma atomica. Las diferencias radican en la sintaxis de los comandos, la granularidad del control, el formato de paquetes y los detalles de implementacion del resolutor de dependencias.

## APT (Debian, Ubuntu)

Advanced Package Tool (APT) es el frontend que opera sobre la capa de bajo nivel `dpkg`. APT no maneja directamente archivos `.deb`; descarga paquetes desde los repositorios, analiza las dependencias, los descarga en el orden correcto y delega en `dpkg` la instalacion real. El resultado es una interfaz uniforme que abstrae la complejidad de los formatos de paquetes y las fuentes remotas.

Comandos esenciales:

```bash
apt update                     # Actualiza la cache de paquetes desde los repositorios
apt upgrade                    # Actualiza todos los paquetes instalados
apt full-upgrade               # Similar a upgrade, pero resuelve cambios de dependencias
apt install paquete            # Instala un paquete y sus dependencias
apt remove paquete             # Elimina un paquete (deja archivos de configuracion)
apt purge paquete              # Elimina un paquete y sus archivos de configuracion
apt autoremove                 # Elimina dependencias huerfanas
apt search termino             # Busca paquetes por nombre o descripcion
apt show paquete               # Muestra metadatos detallados de un paquete
apt list --installed           # Lista paquetes instalados
apt policy paquete             # Muestra versiones disponibles e instalada
```

El comando `apt` es una version simplificada de `apt-get` y `apt-cache` disenada para uso interactivo. En scripts de automaticion se recomienda usar `apt-get` por tener una interfaz mas estable (sin barras de progreso ni cambios cosmeticos entre versiones).

### /etc/apt/sources.list

El archivo `/etc/apt/sources.list` y los archivos en `/etc/apt/sources.list.d/` definen las fuentes de las que APT descarga paquetes. Cada linea tiene el formato:

```
deb [opciones] url distribucion componentes
```

Las opciones pueden incluir `arch=amd64,i386` para restringir arquitecturas, o `signed-by=/ruta/clave.gpg` para especificar una clave GPG concreta. Un ejemplo tipico:

```
deb http://archive.ubuntu.com/ubuntu focal main restricted
deb-src http://archive.ubuntu.com/ubuntu focal main restricted
```

El campo `deb-src` habilita la descarga de codigo fuente, util para compilar paquetes manualmente o inspeccionar modificaciones. APT verifica la autenticidad de los paquetes mediante firmas GPG; las claves de los repositorios oficiales se incluyen en el paquete `ubuntu-keyring` y se actualizan con el sistema. Para repositorios de terceros, se anaden manualmente con `apt-key` (obsoleto) o copiando el anillo de claves en `/etc/apt/trusted.gpg.d/`.

> En versiones recientes de Debian y Ubuntu, `apt-key` esta deprecado. La practica recomendada es descargar la clave GPG y ubicarla en `/etc/apt/keyrings/`, referenciandola en `sources.list` con la opcion `signed-by`.

## DNF y YUM (RHEL, Fedora)

Yellowdog Updater Modified (YUM) fue durante anos el gestor por defecto en Red Hat Enterprise Linux y Fedora. Con la llegada de RHEL 8 y Fedora 22, fue reemplazado por DNF (Dandified YUM), que conserva la sintaxis basica pero reescribe el nucleo del resolutor de dependencias utilizando la biblioteca libre-solv, mejorando el rendimiento y la precision.

Comandos esenciales en DNF:

```bash
dnf check-update               # Verifica actualizaciones disponibles
dnf upgrade                    # Actualiza todos los paquetes
dnf install paquete            # Instala un paquete y sus dependencias
dnf remove paquete             # Elimina un paquete
dnf autoremove                 # Limpia dependencias huerfanas
dnf search termino             # Busca paquetes
dnf info paquete               # Muestra informacion detallada
dnf list installed             # Lista paquetes instalados
dnf groupinstall "Grupo"       # Instala un grupo de paquetes (ej. "Development Tools")
dnf history                    # Muestra el historial de transacciones
dnf reinstall paquete          # Reinstala un paquete (util para reparar)
```

DNF introduce el concepto de transacciones: cada operacion de instalacion, actualizacion o eliminacion se registra como una transaccion que puede deshacerse con `dnf history undo`. Ademas, permite la integracion con modulos de software mediante `dnf module`, una caracteristica que permite instalar versiones especificas de pilas de software (por ejemplo, distintas versiones de PHP o PostgreSQL) en el mismo sistema.

YUM, aunque obsoleto, sigue presente en sistemas RHEL 7 y derivados. Su sintaxis es practicamente identica a DNF, pero su resolutor de dependencias mas lento y propenso a bloqueos en arboles de dependencias complejos. En la practica, `yum` es un alias simbolico de `dnf` en las distribuciones modernas, pero la compatibilidad hacia atras se mantiene.

Los repositorios se configuran en archivos `.repo` dentro de `/etc/yum.repos.d/`:

```ini
[fedora]
name=Fedora $releasever - $basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$basearch
```

## Pacman (Arch Linux)

Arch Linux utiliza Pacman como gestor de paquetes. Pacman maneja paquetes en formato `.pkg.tar.zst` (un archivo tar comprimido con zstd) y opera tanto sobre repositorios oficiales como sobre paquetes locales. Su filosofia es minimalista: no incluye funcionalidades de "auto-deteccion" ni capas de abstraccion innecesarias.

Comandos esenciales:

```bash
pacman -Sy                     # Sincroniza la base de datos de repositorios
pacman -Syu                    # Actualiza el sistema completo (sync + upgrade)
pacman -S paquete              # Instala un paquete
pacman -Rs paquete             # Elimina un paquete y sus dependencias no requeridas
pacman -Ss termino             # Busca paquetes en los repositorios
pacman -Qi paquete             # Muestra informacion de un paquete instalado
pacman -Ql paquete             # Lista los archivos que pertenecen a un paquete
pacman -F archivo              # Busca que paquete provee un archivo
pacman -Sc                     # Limpia la cache de paquetes
```

La sintaxis de Pacman usa banderas cortas agrupadas, lo que la hace mas compacta pero menos autodescriptiva que APT o DNF. La opcion `-S` actua sobre los repositorios (sync), `-Q` sobre los paquetes instalados (query) y `-R` sobre la eliminacion (remove).

Pacman no resuelve dependencias de forma tan agresiva como APT. Si se encuentra con un conflicto, aborta y muestra el error, esperando que el usuario decida como proceder. Esto es coherente con la filosofia KISS (Keep It Simple, Stupid) de Arch, pero requiere que el usuario entienda el sistema de paquetes.

La configuracion de Pacman reside en `/etc/pacman.conf`, donde se definen repositorios, opciones de compresion y comportamiento general. Los repositorios oficiales se configuran en el mismo archivo:

```
[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

[community]
Include = /etc/pacman.d/mirrorlist
```

> Arch Linux sigue un modelo rolling release: los paquetes se actualizan continuamente en lugar de pasar por versiones discretas del sistema. `pacman -Syu` mantiene el sistema al dia con las ultimas versiones estables, lo que elimina las migraciones entre versiones mayores pero exige atencion a los avisos de cambios importantes.

## APK (Alpine Linux)

Alpine Linux utiliza Alpine Package Keeper (APK), un gestor disenado para ser ligero, rapido y seguro. Alpine es conocida por su uso extensivo en contenedores Docker debido a su tamano minimo (la imagen base pesa unos 5 MB), y APK es una parte fundamental de esa eficiencia.

Comandos esenciales:

```bash
apk update                     # Actualiza la cache de repositorios
apk upgrade                    # Actualiza todos los paquetes
apk add paquete                # Instala un paquete
apk del paquete                # Elimina un paquete
apk search termino             # Busca paquetes
apk info paquete               # Muestra informacion detallada
apk list --installed           # Lista paquetes instalados
apk policy paquete             # Muestra versiones disponibles
apk fix                        # Repara dependencias rotas
```

APK se distingue por su sistema de huellas digitales para verificacion de paquetes y por el uso de scripts de instalacion en formato APKBUILD, que son mas simples que los spec de RPM o las reglas de Debian. Cada paquete se verifica mediante firmas RSA de 2048 bits, garantizando la integridad desde el repositorio hasta el disco.

Los repositorios se configuran en `/etc/apk/repositories`, un archivo de una URL por linea:

```
https://dl-cdn.alpinelinux.org/alpine/v3.19/main
https://dl-cdn.alpinelinux.org/alpine/v3.19/community
```

APK no separa los repositorios en archivos separados; todo se define en ese unico archivo. La simplicidad resultante es deliberada: Alpine apunta a entornos donde la superficie de configuracion debe ser minima.

