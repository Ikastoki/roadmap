# Compilacion desde fuente y resolucion de dependencias

Los gestores de paquetes resuelven la mayoria de las necesidades de software, pero no cubren todos los escenarios. Un programa puede no estar empaquetado para la distribucion, puede necesitar parches especificos, o puede requerir opciones de compilacion que el paquete oficial no ofrece. En esos casos, la compilacion desde el codigo fuente sigue siendo la herramientas fundamental del administrador de sistemas.

## Resolucion de dependencias

Antes de entender la compilacion manual es necesario comprender el problema que los gestores resuelven automaticamente. Un paquete rara vez es autonomo; depende de bibliotecas y otros paquetes para funcionar. Por ejemplo, un programa que procesa imagenes puede depender de `libpng`, que a su vez depende de `zlib`. El conjunto de estas relaciones forma un grafo aciclico dirigido (DAG).

El gestor de paquetes debe recorrer este grafo, verificar que cada dependencia esta disponible en la version correcta, descargar los paquetes que falten en el orden topologico adecuado y garantizar que no haya conflictos entre versiones. Este proceso se denomina resolucion de dependencias.

### Dependencias circulares

Ocurren cuando dos o mas paquetes se requieren mutuamente: A depende de B, B depende de C y C depende de A. En teoria, esto no deberia suceder en un sistema bien disenado, pero en la practica aparece por varias razones:

- **Paquetes mal divididos**: dos bibliotecas que deberian estar en un solo paquete se separan artificialmente, generando dependencias cruzadas.
- **Scripts de mantenimiento**: un paquete necesita ejecutar un script que a su vez requiere el otro paquete.
- **Herramientas de compilacion**: al compilar A se necesita B, y al compilar B se necesita A, formando un ciclo de compilacion.

Los gestores modernos detectan estas situaciones y las manejan de forma distinta. APT, por ejemplo, intenta instalar ambos paquetes simultaneamente en una sola transaccion, asumiendo que la dependencia se satisface tras la instalacion aunque el orden exacto sea ambiguo. Si el ciclo implica que un paquete debe estar configurado antes de que el otro pueda instalarse, la transaccion falla.

```bash
apt --fix-broken install
```

Este comando intenta reparar dependencias rotas analizando el estado actual del sistema, eliminando paquetes conflictivos o instalando las dependencias faltantes. Es la primera herramienta que se prueba cuando APT reporta dependencias insatisfechas.

### Versionado y conflictos

Las dependencias no solo especifican que paquete se necesita, sino que version. Los operadores tipicos son:

- `>= 1.2` — version minima
- `<< 2.0` — version anterior a
- `= 1.5` — version exacta
- `1.2-2` — version con revision de empaquetado

Cuando dos paquetes requieren versiones incompatibles de la misma biblioteca, se produce un conflicto. El resolutor debe decidir que combinacion de versiones satisface todas las restricciones. En casos complejos, la unica solucion es actualizar o eliminar uno de los paquetes conflictivos.

## Compilacion desde fuente

Cuando el gestor de paquetes no puede ayudar, la compilacion manual es la via de escape. El proceso canonico sigue cuatro pasos: configuracion, compilacion, instalacion y, opcionalmente, empaquetado.

### Autotools

El sistema Autotools (Autoconf, Automake, Libtool) fue durante decadas el estandar de facto para la compilacion portable en Unix. Su objetivo es detectar las caracteristicas del sistema (compilador disponible, bibliotecas presentes, funciones del nucleo) y generar los archivos de configuracion adecuados.

```bash
tar -xzf programa-1.0.tar.gz
cd programa-1.0
./configure --prefix=/usr/local
make
make install
```

Cada paso tiene una funcion especifica:

1. **`./configure`**: ejecuta una serie de pruebas en el sistema (scripts de shell generados por Autoconf) y genera un archivo `Makefile` personalizado. El parametro `--prefix` define el directorio raiz de instalacion (`/usr/local` es el valor por defecto y el estandar para software compilado manualmente). Otros parametros comunes incluyen `--enable-feature`, `--with-library`, `--disable-debug` y `--help` para listar todas las opciones.

2. **`make`**: lee el `Makefile` generado y ejecuta el compilador (gcc, clang) para transformar el codigo fuente en objetos binarios. `make` utiliza timestamps para solo recompilar lo que ha cambiado, y acepta la opcion `-j$(nproc)` para paralelizar la compilacion en multiples nucleos.

3. **`make install`**: copia los binarios, bibliotecas, archivos de cabecera, paginas de manual y configuraciones a los directorios del sistema. Requiere permisos de superusuario porque escribe en `/usr/local/` o `/usr/`.

### Meson y CMake

Aunque Autotools sigue siendo comun, proyectos modernos adoptan sistemas de compilacion mas eficientes:

**CMake** genera archivos de compilacion nativos (Makefiles, Ninja) desde un archivo declarativo `CMakeLists.txt`:

```bash
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install
```

**Meson** es un sistema mas reciente que prioriza la velocidad y la claridad de los mensajes de error:

```bash
meson setup builddir --prefix=/usr/local
ninja -C builddir
ninja -C builddir install
```

Meson utiliza Ninja como backend de compilacion, que es notablemente mas rapido que Make en proyectos grandes.

### checkinstall y empaquetado manual

Una de las limitaciones de `make install` es que el gestor de paquetes no tiene registro de los archivos instalados. Si luego se quiere desinstalar el programa, no hay forma limpia de hacerlo sin revisar manualmente el `Makefile`. La herramienta `checkinstall` resuelve este problema generando un paquete `.deb` o `.rpm` a partir de los archivos instalados:

```bash
./configure --prefix=/usr/local
make
checkinstall
```

`checkinstall` monitoriza los archivos que `make install` escribe en el sistema, los empaqueta en el formato nativo de la distribucion y registra el paquete en la base de datos del gestor. Esto permite desinstalar el programa con `apt remove` o `dnf remove` como si se hubiera instalado desde un repositorio.

```bash
# checkinstall acepta opciones para personalizar el paquete:
checkinstall --pkgname=programa --pkgversion=1.0 --pkglicense=GPL
```

> `checkinstall` no debe usarse con paquetes que el gestor ya provee, a menos que sea necesario sobrescribirlos deliberadamente. Instalar software compilado manualmente encima de la version del repositorio puede causar conflictos de archivos y dependencias que el gestor no puede resolver.

## Estrategias para compilaciones complejas

Al compilar software que depende de muchas bibliotecas, suele ser mas practico instalar primero las dependencias desde los repositorios antes de compilar manualmente. El gestor de paquetes proporciona paquetes `-dev` o `-devel` que incluyen las cabeceras necesarias para compilar:

```bash
# En Debian/Ubuntu
apt build-dep programa

# En Fedora/RHEL
dnf builddep programa

# En Arch Linux
pacman -S base-devel
```

El comando `apt build-dep` lee la informacion de dependencias de compilacion del paquete especificado (si existe en los repositorios) y las instala automaticamente. Esto ahorra el proceso de localizar manualmente cada biblioteca de desarrollo.

Cuando el software que se quiere compilar no esta en los repositorios, se puede usar `apt-get source` para descargar las fuentes de un paquete existente, modificarlas y recompilarlas con `dpkg-buildpackage`. Este metodo conserva el sistema de empaquetado oficial y produce un `.deb` completamente integrable.

