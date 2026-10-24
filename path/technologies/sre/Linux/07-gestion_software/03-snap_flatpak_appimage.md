# Snap, Flatpak y AppImage

La gestion de paquetes tradicional ata el software a una distribucion concreta y sus bibliotecas compartidas. Un paquete `.deb` compilado para Ubuntu 20.04 puede fallar en Ubuntu 22.04 si una biblioteca cambia su ABI, y desde luego no funcionara en Fedora. Los formatos universales nacen de la necesidad de empaquetar una aplicacion una sola vez y ejecutarla en cualquier distribucion Linux, independientemente de su sistema base.

Tres tecnologias compiten en este espacio: Snap, Flatpak y AppImage. Las tres persiguen el mismo objetivo --la portabilidad binaria entre distribuciones-- pero lo abordan con arquitecturas y filosofias notablemente distintas.

## Snap

Snap fue desarrollado por Canonical, la empresa detras de Ubuntu, y se presento en 2016 como un sistema de empaquetado universal y transaccional. Un Snap es una imagen de sistema de archivos comprimida que contiene la aplicacion, sus dependencias y los metadatos necesarios para ejecutarla en un entorno aislado.

La tecnologia clave detras de Snap es squashfs: el contenido del paquete se monta como un sistema de archivos de solo lectura en `/snap/<nombre>/<version>/`. Este montaje se realiza bajo demanda y en espacio de usuario mediante `snapd`, el demonio que gestiona el ciclo de vida de los Snaps.

Caracteristicas distintivas:

- **Actualizaciones atomicas**: `snapd` descarga las actualizaciones en segundo plano y las aplica mediante un mecanismo transaccional. Si la actualizacion falla, el sistema revierte automaticamente a la version anterior.
- **Canales de actualizacion**: cada Snap puede definir canales (`stable`, `candidate`, `beta`, `edge`) que permiten a los usuarios elegir el nivel de riesgo que asumen.
- **Aislamiento mediante AppArmor**: los Snaps se ejecutan en un sandbox restringido por perfiles AppArmor que limitan el acceso a la red, el sistema de archivos y los recursos del sistema. El grado de confinement se controla con niveles como `strict`, `classic` y `devmode`.
- **Snap Store centralizado**: Canonical opera la tienda oficial donde los desarrolladores publican sus Snaps. La busqueda e instalacion se realiza con un solo comando:

```bash
snap find "aplicacion"
snap install aplicacion
snap install --channel=beta aplicacion
snap refresh                    # Actualiza todos los Snaps instalados
snap revert aplicacion          # Revierte a la version anterior
```

La principal critica a Snap es su modelo centralizado: la tienda es propiedad de Canonical y el demonio `snapd` es software libre pero con componentes propietarios en el backend. Ademas, el montaje de squashfs y el aislamiento via AppArmor introducen una latencia perceptible en el arranque de aplicaciones, especialmente en discos lentos o sistemas sin soporte nativo de snapd.

> El Snap Store permite la publicacion automatica desde CI/CD, lo que facilita que los desarrolladores entreguen actualizaciones directamente a los usuarios sin pasar por los repositorios de las distribuciones. Esto acelera el ciclo de entrega pero desplaza la responsabilidad de la curaduria desde la distribucion hacia el desarrollador.

## Flatpak

Flatpak, impulsado por Red Hat y la comunidad GNOME, adopta un enfoque diferente. En lugar de empaquetar la aplicacion con todas sus dependencias, Flatpak utiliza el concepto de runtimes: capas base que contienen las bibliotecas compartidas que muchas aplicaciones necesitan (GTK, Qt, PulseAudio, D-Bus). Una aplicacion Flatpak declara su dependencia de un runtime concreto y solo empaqueta lo que el runtime no proporciona.

La arquitectura de Flatpak se compone de:

- **Runtimes**: conjuntos de bibliotecas mantenidos por la comunidad Flatpak. Ejemplos: `org.freedesktop.Platform`, `org.gnome.Platform`. Se instalan una sola vez y todas las aplicaciones que dependen de ellos los comparten.
- **Aplicaciones**: paquetes que contienen solo el codigo especifico de la aplicacion y las dependencias no cubiertas por el runtime.
- **Burbuja de usuario (Bubblewrap)**: tecnologia de contenedorizacion que aísla la aplicacion del sistema anfitrion mediante espacios de nombres de Linux (namespaces). Flatpak no necesita un demonio; la aplicacion se lanza directamente con `flatpak run`.

Comandos basicos:

```bash
flatpak search aplicacion
flatpak install flathub aplicacion
flatpak update                  # Actualiza runtimes y aplicaciones
flatpak list                    # Muestra aplicaciones y runtimes instalados
flatpak run aplicacion
flatpak override --filesystem=host aplicacion  # Concede permisos adicionales
```

Flatpak utiliza Flathub como repositorio principal, aunque cualquiera puede configurar repositorios remotos adicionales. A diferencia de Snap, Flathub no pertenece a una unica empresa: es un proyecto comunitario gobernado por el Flatpak Team.

Las ventajas respecto a Snap incluyen un aislamiento mas granular (cada aplicacion especifica explicitamente los permisos que necesita mediante archivos de metadatos), un menor consumo de espacio gracias a los runtimes compartidos, y una integracion mas natural con los entornos de escritorio GNOME y KDE. Sin embargo, los runtimes ocupan espacio en disco (cientos de megabytes cada uno) y el sistema de permisos puede resultar confuso para usuarios noveles.

> Flatpak permite anadir repositorios de terceros, como el de la propia empresa que desarrolla el software, lo que ofrece un modelo mas distribuido que el de Snap. El comando `flatpak remotes` muestra los repositorios configurados, y `flatpak remote-add` permite anadir nuevos.

## AppImage

AppImage es el mas simple de los tres formatos: un unico archivo ejecutable que contiene la aplicacion y todas sus dependencias. No requiere instalacion, demonios, montajes de sistema de archivos ni runtimes preinstalados. Se descarga, se le dan permisos de ejecucion y se ejecuta.

```bash
chmod +x aplicacion.AppImage
./aplicacion.AppImage
```

Esta simplicidad es su mayor ventaja y tambien su limitacion fundamental. AppImage no integra la aplicacion en el sistema: no anade entradas al menu de aplicaciones, no registra tipos MIME, no se actualiza automaticamente ni participa en el sistema de paquetes nativo. Existen herramientas como AppImageLauncher que mitigan estas carencias, pero no forman parte del formato en si.

AppImage utiliza un sistema de archivos FUSE para montar el contenido del archivo bajo demanda. La aplicacion se ejecuta dentro de este entorno montado sin necesidad de extraer los archivos al disco. Esto permite que la misma AppImage funcione en cualquier distribucion que tenga FUSE disponible, lo que cubre practicamente todos los sistemas Linux modernos.

Las actualizaciones pueden gestionarse mediante un mecanismo opcional de delta updates: el desarrollador incluye una URL de actualizacion en los metadatos, y herramientas como `appimageupdatetool` descargan solo los cambios binarios entre versiones.

## Comparativa

| Aspecto | Snap | Flatpak | AppImage |
|---|---|---|---|
| Aislamiento | AppArmor (sandbox forzado) | Bubblewrap (namespaces) | Ninguno (acceso completo) |
| Actualizaciones | Atomicas, automaticas | Mediante repositorio | Delta update manual |
| Centralizacion | Canonical (Snap Store) | Flathub (comunitario) | Distribucion directa |
| Dependencias | Empaquetadas con la app | Runtimes compartidos | Todo incluido en el archivo |
| Integracion en escritorio | Buena (snapd gestiona todo) | Buena (Portal D-Bus) | Basica (depende de lanzadores) |
| Tamano en disco | Mayor (cada app incluye todo) | Medio (runtimes compartidos) | Mayor (cada AppImage es autonoma) |

Snap y Flatpak son adecuados para aplicaciones que requieren actualizaciones frecuentes, aislamiento de seguridad y gestion centralizada. AppImage brilla en escenarios de portabilidad pura: un ejecutable que funciona en cualquier sitio sin dejar rastro, ideal para herramientas portables, versiones de prueba o software que no justifica empaquetarse para cada distribucion.

Ninguno de los tres reemplaza al gestor de paquetes nativo del sistema, sino que lo complementan. Las bibliotecas base y las herramientas del sistema (el compilador, el kernel, los controladores) seguiran gestionandose con APT, DNF o Pacman. Los formatos universales cubren el estrato de aplicaciones de usuario, donde la reproducibilidad y la portabilidad pesan mas que la integracion profunda con la distribucion.

