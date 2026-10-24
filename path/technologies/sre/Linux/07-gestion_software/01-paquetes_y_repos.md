# Paquetes y repositorios

Un sistema operativo no es mas que un conjunto de archivos organizados en una estructura concreta. Cuando se quiere anadir una nueva funcionalidad -- un navegador, un compilador, un servidor web -- el metodo artesanal consiste en localizar el codigo fuente, compilarlo y copiar los binarios a sus ubicaciones correspondientes. Este proceso es repetitivo, propenso a errores y, sobre todo, dificil de deshacer. Los gestores de paquetes existen para resolver exactamente ese problema: automatizar la instalacion, actualizacion, configuracion y eliminacion de software de forma consistente.

## Que es un paquete

Un paquete es un archivo que contiene todo lo necesario para instalar un programa: los binarios compilados, las bibliotecas compartidas, los archivos de configuracion por defecto, las paginas de manual, los scripts de pre y postinstalacion, y los metadatos que describen el paquete (nombre, version, dependencias, conflicto con otros paquetes, mantenedor, suma de verificacion). Esta estructura permite que el gestor de paquetes sepa exactamente que archivos pertenecen a que programa, donde deben colocarse y que otros paquetes se requieren previamente.

El contenido de un paquete se organiza en torno a dos conceptos clave:

- **Metadatos**: informacion descriptiva y declarativa que el gestor utiliza para tomar decisiones. Incluye el nombre del paquete, la version (siguiendo convenciones como semantic versioning o versiones propias de la distribucion), las dependencias, las sugerencias, los conflictos y la arquitectura objetivo.
- **Payload**: los archivos que realmente se instalaran en el sistema. Suelen comprimirse con algoritmos como gzip, xz o zstd para reducir el tamano de descarga.

Los paquetes pueden ser binarios (precompilados para una arquitectura y distribucion concretas) o de codigo fuente (el gestor descarga las fuentes y las compila en el momento). La mayoria de los usuarios interactua con paquetes binarios, pero los gestores como Gentoo (Portage) o la opcion `src.rpm` de Fedora trabajan desde fuente.

## Repositorios oficiales y de terceros

Un repositorio es un servidor remoto (o un conjunto de ellos) que aloja paquetes junto con sus metadatos en una estructura indexada. El gestor de paquetes descarga el indice (`Packages.gz`, `Packages.xz`, `repomd.xml`), lo almacena localmente en cache y lo utiliza para resolver consultas, dependencias y actualizaciones sin necesidad de conocer las URLs exactas de cada paquete.

Las distribuciones mantienen repositorios oficiales organizados en secciones o componentes segun el modelo de soporte y licencia. Por ejemplo, Debian divide su repositorio en `main` (software completamente libre), `contrib` (software libre que depende de componentes no libres) y `non-free` (software que no cumple con los criterios de libertad de Debian). Ubuntu anade ademas los repositorios `universe` (software mantenido por la comunidad), `restricted` (controladores propietarios con soporte) y `multiverse` (software no libre).

La configuracion de repositorios en sistemas basados en Debian se realiza mediante el archivo `/etc/apt/sources.list` y los fragmentos en `/etc/apt/sources.list.d/`. Cada linea sigue la estructura:

```
deb http://archive.ubuntu.com/ubuntu focal main restricted universe
```

El primer campo indica el tipo de archivo (`deb` para binarios, `deb-src` para fuentes), seguido de la URL del repositorio, la distribucion o version, y los componentes.

Los repositorios de terceros amplian el catalogo oficial pero introducen riesgos de seguridad y estabilidad. Al anadir una clave GPG externa y un nuevo `sources.list`, se confia en que ese tercero mantendra sus paquetes actualizados y no distribuira codigo malicioso. Ejemplos comunes son los repositorios de Docker, Google Chrome, Microsoft VS Code o NodeSource. La practica recomendada es usar repositorios de terceros solo cuando el software no este disponible en los oficiales y verificar la autenticidad de las claves.

> En distribuciones como Arch Linux, el Arch User Repository (AUR) permite a la comunidad publicar recetas de compilacion (PKGBUILD) que cualquiera puede revisar y ejecutar. Aunque el AUR no es un repositorio oficial, `yay` o `paru` actuan como wrappers que automatizan la construccion desde esas recetas, lo que ofrece una flexibilidad enorme a cambio de una responsabilidad de revision manual.

## Formato .deb

El formato `.deb` es el sistema de empaquetado de Debian y sus derivados (Ubuntu, Linux Mint, Kali, Pop!_OS). Cada archivo `.deb` es un archivo ar que contiene tres archivos en su interior:

```
control.tar.gz   — metadatos, scripts de mantenimiento
data.tar.gz      — los archivos a instalar (binarios, configuraciones, docs)
debian-binary    — version del formato (actualmente "2.0")
```

Los scripts de mantenimiento (`preinst`, `postinst`, `prerm`, `postrm`) son scripts de shell que se ejecutan en momentos concretos de la instalacion o eliminacion. Esto permite realizar acciones como crear usuarios del sistema, configurar bases de datos o limpiar archivos temporales.

Debian implementa un sistema de dependencias rico mediante campos como `Depends`, `Recommends`, `Suggests`, `Pre-Depends`, `Breaks` y `Conflicts`. La herramienta de bajo nivel `dpkg` maneja la instalacion directa de archivos `.deb`, pero no resuelve dependencias automaticamente. Para eso esta `apt`, que actua como frontend: consulta los repositorios, resuelve el arbol de dependencias, descarga los paquetes necesarios y ejecuta `dpkg` en el orden correcto.

## Formato .rpm

El formato `.rpm` (originalmente Red Hat Package Manager) es el estandar en Red Hat Enterprise Linux, Fedora, CentOS, Rocky Linux, AlmaLinux, openSUSE y derivados. La estructura interna de un `.rpm` incluye:

```
Lead          — cabecera historica para identificar el archivo
Signature     — firmas criptograficas (PGP, SHA) para verificar integridad
Header        — metadatos: nombre, version, dependencias, scripts
Payload       — archivos comprimidos (cpio + gz, xz o zstd)
```

RPM distingue entre paquetes binarios (`.rpm`) y paquetes fuente (`.src.rpm` o `.srpm`), que contienen el codigo fuente, los parches y el spec file necesario para construir el binario. El spec file (`*.spec`) es un archivo de receta que define como compilar, instalar y empaquetar el software.

Al igual que `dpkg`, la herramienta de bajo nivel `rpm` instala paquetes individuales sin resolver dependencias. Los frontends como `dnf` (Fedora, RHEL 8+) o `zypper` (openSUSE) se encargan de la resolucion completa consultando repositorios configurados en `/etc/yum.repos.d/` o `/etc/zypp/repos.d/`.

## Diferencias practicas entre .deb y .rpm

Ambos formatos cumplen la misma funcion pero divergen en detalles tecnicos y de la experiencia de uso:

| Aspecto | .deb | .rpm |
|---|---|---|
| Distribuciones | Debian, Ubuntu, Mint | RHEL, Fedora, openSUSE |
| Herramienta de bajo nivel | dpkg | rpm |
| Frontend comun | apt, aptitude | dnf, yum, zypper |
| Sistema de paquetes | control.tar.gz + data.tar.gz | cpio empaquetado |
| Scripts de mantenimiento | preinst, postinst, prerm, postrm | Pre/post install/uninstall en spec |
| Gestion de configuracion | Debconf para paquetes interactivos | Scriptlet con opciones de linea de comandos |

En la practica, la mayoria de las diferencias quedan ocultas tras la abstraccion del gestor de paquetes. Un usuario de Ubuntu no interactua directamente con archivos `.deb` mas que cuando descarga software que no esta en los repositorios; un usuario de Fedora experimenta una situacion analoga con `.rpm`. La eleccion entre distribuciones basadas en uno u otro formato rara vez viene determinada por el sistema de empaquetado en si, sino por el ecosistema, la filosofia de actualizaciones y la comunidad que rodea a cada distribucion.

