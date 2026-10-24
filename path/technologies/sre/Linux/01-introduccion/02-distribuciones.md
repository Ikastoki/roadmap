# Distribuciones de Linux

Tener un kernel y herramientas de sistema no basta para tener un sistema operativo util. Hace falta un gestor de paquetes, un instalador, un administrador de servicios, una interfaz grafica, controladores para el hardware y actualizaciones coordinadas. Una distribucion de Linux es todo eso: un empaquetamiento coherente del kernel Linux, las herramientas GNU, un gestor de paquetes, repositorios de software, configuraciones predeterminadas y un equipo que mantiene el conjunto.

La diferencia entre el kernel y la distribucion es la misma que entre un motor y un coche. El kernel es el motor: gestiona los recursos, planifica procesos y media entre el hardware y el software. Pero un coche necesita carroceria, ruedas, asientos, sistema electrico y alguien que garantice que todo funciona junto. La distribucion es ese vehiculo completo, disenado para un publico y un proposito concreto.

## Familias de distribuciones

Todas las distribuciones comparten el mismo kernel Linux y herramientas similares, pero divergen en aspectos clave: el formato de los paquetes, el gestor de paquetes, el sistema de inicio, la configuracion predeterminada y el ciclo de lanzamiento. Estas diferencias definen familias que heredan unas de otras.

### Debian y Ubuntu

Debian es una de las distribuciones mas antiguas y respetadas. Fundada en 1993 por Ian Murdock, se rige por un contrato social que prioriza el software libre y la transparencia. Su gestor de paquetes APT y el formato .deb son el estandar en su ecosistema. Debian tiene tres ramas principales: Stable (probada, adecuada para servidores), Testing (paquetes mas recientes, equilibrio entre estabilidad y novedades) y Sid (inestable, desarrollo continuo). Es la distribucion de referencia para quien valora la estabilidad absoluta por encima de la novedad.

Ubuntu, lanzada en 2004 por Canonical (Mark Shuttleworth), deriva directamente de Debian Testing. Su objetivo historico era llevar Linux al usuario de escritorio con una instalacion sencilla y soporte hardware amplio. Canonical introdujo Unity como interfaz grafica, aunque luego migro a GNOME. Ubuntu mantiene un ciclo de lanzamiento semestral con versiones LTS (Long Term Support) cada dos anos, que reciben soporte durante cinco anos. Es la distribucion mas utilizada en entornos de desarrollo y una de las mas populares en servidores y escritorio.

Derivados notables de Ubuntu incluyen Linux Mint (escritorio tradicional, ideal para migrados desde Windows), Pop!_OS (de System76, optimizada para desarrollo y gaming), y Elementary OS (diseno cuidado, similar a macOS).

### Fedora y Red Hat Enterprise Linux

Fedora es la distribucion community del proyecto Fedora, patrocinado por Red Hat (ahora IBM). Actua como campo de pruebas para tecnologias que luego se incorporan a Red Hat Enterprise Linux (RHEL). Fedora incorpora paquetes modernos, un ciclo de lanzamiento rapido (cada seis meses) y soporte para cada version durante aproximadamente un ano. Es ideal para desarrolladores que quieren lo ultimo del ecosistema Linux sin sacrificar la integridad del sistema.

Red Hat Enterprise Linux es el producto comercial de Red Hat. Se enfoca en estabilidad, soporte a largo plazo (10 anos), certificaciones de hardware y software empresarial (bases de datos, middleware, ERP). Las suscripciones incluyen acceso al portal de soporte, parches de seguridad y herramientas de gestion como Red Hat Satellite. RHEL es el estandar en muchas empresas y organismos gubernamentales.

CentOS (Community Enterprise Operating System) era un clon gratuito de RHEL, pero Red Hat cambio el modelo. Hoy existe CentOS Stream, que se situa entre Fedora y RHEL en el ciclo de desarrollo. El heredero funcional de CentOS clasico es Rocky Linux, mantenido por Gregory Kurtzer (cofundador original de CentOS), y AlmaLinux, respaldado por CloudLinux.

Rocky Linux y AlmaLinux son binariamente compatibles con RHEL, lo que significa que el software certificado para RHEL funciona sin modificaciones.

### Arch Linux

Arch sigue el principio KISS (Keep It Simple, Stupid), entendido como "mantener la simplicidad desde el punto de vista del diseno, no del usuario". Arch no es una distribucion para principiantes: se instala desde cero mediante linea de comandos, no incluye configuraciones innecesarias y otorga control total sobre que paquetes y servicios se instalan.

Arch utiliza pacman como gestor de paquetes y emplea un modelo de lanzamiento continuo (rolling release): no hay versiones discretas, sino que el sistema se actualiza permanentemente con las ultimas versiones del software. Esto garantiza acceso inmediato a las novedades, pero requiere mantenimiento activo y puede introducir cambios que rompan la compatibilidad.

El Arch User Repository (AUR) es una comunidad de paquetes mantenidos por usuarios que extiende enormemente el catalogo de software disponible. Manjaro y EndeavourOS son derivados de Arch que simplifican la instalacion y ofrecen experiencias mas accesibles.

### Alpine Linux

Alpine Linux esta disenado para ser pequeno, simple y seguro. Utiliza musl como biblioteca C (en lugar de glibc) y busybox como conjunto de utilidades base. El resultado es un sistema base de apenas 5 MB, lo que lo convierte en la opcion preferida para contenedores Docker: la imagen oficial de Alpine pesa unos 8 MB, frente a los 150 MB de Ubuntu.

Alpine usa apk como gestor de paquetes e incorpora por defecto protecciones de seguridad como SSP (Stack Smashing Protection) y ASLR posicional. Su publico objetivo son usuarios avanzados que necesitan sistemas minimos y eficientes para contenedores, dispositivos embebidos o servidores con recursos limitados.

## Criterios de eleccion de una distribucion

La eleccion de distribucion depende del caso de uso:
- **Servidores de produccion**: Debian Stable, Rocky Linux, AlmaLinux, Ubuntu LTS. Se prioriza la estabilidad y el soporte a largo plazo.
- **Escritorio personal**: Ubuntu, Linux Mint, Fedora Workstation. Se valora la facilidad de uso y la compatibilidad hardware.
- **Desarrollo de software**: Fedora, Arch, Ubuntu. Se necesita acceso rapido a herramientas y librerias modernas.
- **Contenedores y embebido**: Alpine. Importa el tamano reducido y la superficie de ataque minima.
- **Estudio y aprendizaje**: Arch, Gentoo, Linux From Scratch. Obligan a comprender el sistema en profundidad.

> Ninguna distribucion es objetivamente mejor que otra. Cada una hace concesiones distintas entre estabilidad, novedad, simplicidad y control. La eleccion correcta es la que se alinea con las necesidades del usuario y el entorno de despliegue.

