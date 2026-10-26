# Eleccion del sistema operativo

La seleccion del sistema operativo para un servidor no es una decision cosmética: determina el ciclo de vida del soporte, la cadena de actualizaciones de seguridad, la disponibilidad de paquetes y la compatibilidad con las herramientas de orquestación que se desplegarán sobre él. A diferencia de un escritorio donde el usuario puede tolerar cambios frecuentes en la interfaz, un servidor exige previsibilidad, estabilidad a largo plazo y un modelo de actualización que no rompa servicios en producción. Cada distribución encode una filosofía distinta sobre cómo equilibrar estas exigencias, y comprender esas diferencias es el primer paso para tomar una decision fundamentada.

## Distribuciones predominantes en entornos servidor

### Debian

Debian es la distribución "madre" de una enorme cantidad de variantes. Su rama `stable` se publica aproximadamente cada dos años y recibe soporte de seguridad durante cinco años o más, lo que la convierte en una base fiable para servicios que no pueden permitirse reinicios frecuentes. El repositorio `stable` contiene paquetes conservadores, probados exhaustivamente antes de su inclusion, lo que minimiza la aparicion de regresiones. Su sistema de paquetes `.deb` y la herramienta `apt` son estandar de facto en gran parte del ecosistema Linux.

La contrapartida es que los paquetes en `stable` suelen estar versionadas por detras de las ultimas ediciones upstream. Para servidores donde la estabilidad es prioritaria y no se necesita la ultima version de una libreria, esta es una ventaja; para entornos de desarrollo o laboratorio, puede resultar restrictivo.

### Ubuntu Server

Ubuntu Server, mantenido por Canonical, toma la base de Debian y añade un ciclo de lanzamiento predecible: cada seis meses aparece una nueva version, y cada dos años se publica una version LTS (Long Term Support) con cinco años de soporte de seguridad ampliable a diez con Ubuntu Pro. Esta previsibilidad facilita la planificacion de actualizaciones en entornos empresariales. Canonical tambien ofrece herramientas propias como `Netplan` para la configuracion de red y `Snap` para la empaquetacion de aplicaciones, aunque estas ultimas generan debate en la comunidad por su enfoque centralizado.

Ubuntu Server domina los servidores en la nube (las imagenes de AWS, Azure y GCP lo ofrecen como opcion por defecto) y es la distribucion mas recomendada para administrators que empiezan. Su comunidad es amplia, la documentacion es extensa y la disponibilidad de PPAs permite acceder a versiones mas recientes de software cuando es necesario.

### CentOS Stream

CentOS Stream ocupa una posicion particular en el ecosistema Red Hat. Tras la discontinuacion de CentOS Linux (que era un clon binario de RHEL), CentOS Stream se convirtio en la plataforma de desarrollo aguas arriba de RHEL: las funcionalidades llegan primero a Stream y despues se filtran a RHEL. Esto implica que Stream es algo mas inestable que RHEL pero sigue beneficiandose del rigor del ecosistema enterprise de Red Hat.

Para organizaciones que ya utilizan RHEL o que necesitan compatibilidad certificada con software enterprise, CentOS Stream ofrece un punto de entrada gratuito. Sin embargo, su ciclo de vida es mas corto (aproximadamente cinco años por version) y su modelo "rolling" dentro de cada ciclo puede introducir cambios inesperados.

### Rocky Linux

Rocky Linux nacio como respuesta directa a la descontinuacion de CentOS Linux. Fundado por Gregory Kurtzer, cofundador de CentOS, su objetivo es ser un reemplazo binario de RHEL, compilado desde las fuentes publicas de Red Hat. Para organizaciones que dependian de la compatibilidad 1:1 con RHEL sin el coste de licencias, Rocky Linux se convirtio en la alternativa natural junto con AlmaLinux.

Rocky sigue un ciclo de lanzamiento sincronizado con RHEL y ofrece soporte de seguridad durante diez años. Su estabilidad y compatibilidad lo hacen apropiado para entornos donde el software esta certificado specificamente contra RHEL.

### Alpine Linux

Alpine Linux adopta un enfoque radicalmente diferente: un sistema minimo construido sobre `musl libc` y `BusyBox`, con una imagen base que ocupa apenas unos megabytes. Este tamano diminuto lo convierte en la distribucion preferida para contenedores Docker, donde cada megabyte de imagen afecta el tiempo de despliegue y el consumo de memoria.

Sin embargo, Alpine presenta diferencias significativas con las distribuciones tradicionales. El uso de `musl` en lugar de `glibc` puede causar incompatibilidades con software compilado dinamicamente contra glibc. El sistema de paquetes `apk` es rapido pero el repositorio es mas reducido. Para servidores tradicionales que ejecutan multiples servicios y dependen de librerias del sistema, estas diferencias pueden generar problemas difficultiles de diagnosticar.

## Criterios de seleccion

### Estabilidad y ciclo de vida

El criterio mas critico es la longitud y previsibilidad del soporte. Un servidor en produccion no puede quedarse sin parches de seguridad por falta de planificacion. Debian stable ofrece el ciclo mas largo y predecible; Ubuntu LTS combina duracion con ecosistema comercial; RHEL/Rocky aportan certificaciones enterprise. Para entornos de contenedores de corta vida, estos criterios se relajan porque las imagenes se reemplazan periodicamente.

### Soporte y ecosistema

El valor de una distribucion no se mide solo por sus paquetes, sino por la disponibilidad de documentacion, foros activos, actualizaciones de seguridad rapidas y compatibilidad con las herramientas de terceros. Ubuntu y Debian tienen la comunidad mas amplia; RHEL y sus derivadas cuentan con soporte comercial oficial. Alpine ha ganado relevancia en el contexto de contenedores, pero su ecosistema sigue siendo mas reducido.

### Seguridad

Todas las distribuciones principales mantienen equipos de seguridad dedicados, pero difieren en politicas de parcheo. Debian prioriza la estabilidad sobre la rapidez; Ubuntu ofrece actualizaciones de seguridad automaticas con `unattended-upgrades`; RHEL proporciona parches certificados. La eleccion depende del modelo operativo: si se prefiere parcheo manual y controlado, Debian o RHEL; si se busca automatizacion, Ubuntu facilita esta tarea.

### Compatibilidad con herramientas de despliegue

Las herramientas de orquestacion como Ansible, Terraform, Kubernetes o Docker tienen preferencias implicitas. Ansible funciona nativamente con cualquier distribucion pero sus modulos estan mas probados en RHEL/Ubuntu. Kubernetes recomienda Ubuntu o Debian para sus nodos. Docker oficialmente soporta Alpine, Debian, Ubuntu, CentOS y RHEL. Verificar la compatibilidad con el stack previsto antes de elegir distribucion evita sorpresas durante el despliegue.

> En la practica, la mayoria de organizaciones eligen entre Debian, Ubuntu y RHEL/Rocky. Las tres cubren el 90% de los casos de uso. La eleccion entre ellas depende mas del contexto organizativo (si ya existe un estandar, si se necesita certificacion, si se prefiere comunidad abierta o soporte comercial) que de diferencias tecnicas sustanciales.

