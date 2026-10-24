# OpenCode CLI

La instalacion de OpenCode esta disenada para adaptarse a multiples entornos y flujos de trabajo, desde el desarrollador individual que busca una configuracion rapida hasta el administrador de sistemas que necesita desplegarlo en equipos de forma estandarizada. La CLI esta escrita en TypeScript y se distribuye como un binario compilado o un paquete de npm, segun el metodo de instalacion elegido.

## Metodos de instalacion

El metodo mas directo y recomendado para la mayoria de los usuarios es el script oficial via curl. Este script detecta automaticamente el sistema operativo y la arquitectura del procesador, descarga el binario precompilado mas reciente desde los servidores de OpenCode y lo ubica en el PATH del sistema.

```bash
curl -fsSL https://opencode.ai/install | bash
```

Este enfoque no requiere Node.js ni ningun otro runtime, ya que el binario incluye todo lo necesario. El script verifica la integridad de la descarga mediante sumas de verificacion SHA-256 y, en sistemas Linux, escribe el binario en `/usr/local/bin/`. Su principal ventaja es la simplicidad y la independencia de gestores de paquetes, aunque no ofrece un mecanismo integrado de desinstalacion automatica mas alla de eliminar el binario manualmente.

Para quienes ya trabajan con el ecosistema Node.js, la instalacion via npm es una alternativa natural que ademas facilita la gestion de versiones:

```bash
npm install -g @opencode/cli
```

Este metodo requiere Node.js 18 o superior, ya que OpenCode hace uso intensivo de las APIs modernas del runtime, incluyendo el sistema de modulo nativo y las capacidades de fetching HTTP sin dependencias externas. La ventaja de npm es que `npm update -g @opencode/cli` mantiene la herramienta actualizada dentro del flujo habitual del desarrollador JavaScript. Sin embargo, el binario resultante no es un ejecutable nativo sino un script que depende del runtime de Node.js, lo que anade una sobrecarga minima en el tiempo de arranque.

En macOS y Linux, los usuarios de Homebrew pueden instalar OpenCode mediante su formula oficial:

```bash
brew install opencode
```

Homebrew gestiona automaticamente las dependencias, las actualizaciones con `brew upgrade opencode` y la desinstalacion limpia con `brew uninstall opencode`. La formula esta mantenida por el equipo de OpenCode en el tap principal de Homebrew, lo que garantiza que las versiones esten sincronizadas con los lanzamientos oficiales. Para los usuarios de Arch Linux, el paquete esta disponible en el AUR bajo el nombre `opencode`:

```bash
paru -S opencode
```

La comunidad de Arch mantiene el PKGBUILD que compila el paquete desde las fuentes publicadas en GitHub, integrando OpenCode en el sistema de actualizaciones del gestor de paquetes nativo. Bun, el runtime alternativo a Node.js, tambien soporta la instalacion global de OpenCode:

```bash
bun install -g opencode
```

En Windows, aunque OpenCode funciona de forma nativa, el equipo recomienda encarecidamente utilizar WSL (Windows Subsystem for Linux) para una experiencia completa, ya que algunas herramientas auxiliares del ecosistema OpenCode estan optimizadas para entornos Unix. Para instalaciones nativas en Windows, estan disponibles los gestores winget y scoop:

```bash
winget install opencode
scoop install opencode
```

> El metodo de instalacion no afecta al comportamiento de OpenCode una vez instalado. La unica diferencia perceptible es el mecanismo de actualizacion y desinstalacion, que depende del gestor de paquetes utilizado. El binario compilado y el paquete de npm ofrecen exactamente las mismas funcionalidades.

## Verificacion y actualizacion

Una vez instalado, la verificacion de la version activa se realiza mediante el flag `--version` o su forma corta `-v`:

```bash
opencode --version
```

OpenCode incorpora un sistema de actualizacion automatica que, por defecto, verifica periodicamente si hay una version mas reciente disponible en los repositorios de GitHub. Cuando detecta una nueva version, descarga en segundo plano el binario actualizado y lo aplica en el siguiente inicio de sesion. Este mecanismo utiliza la API de GitHub Releases para comparar la version local con la ultima estable publicada, sin enviar ningun dato de telemetria.

La CLI expone el comando `upgrade` para forzar una actualizacion manual inmediata:

```bash
opencode upgrade
```

Tambien es posible instalar una version especifica anadiendo el tag de la version como argumento:

```bash
opencode upgrade v0.1.48
```

Para entornos donde las actualizaciones automaticas no son deseables, como servidores de integracion continua o entornos corporativos con politicas de software estrictas, la variable de entorno `OPENCODE_DISABLE_AUTOUPDATE` desactiva completamente el mecanismo:

```bash
export OPENCODE_DISABLE_AUTOUPDATE=true
```

> Las actualizaciones automaticas solo afectan al binario principal. Las dependencias de plugins, configuraciones y datos locales permanecen intactos durante el proceso de actualizacion.

## Desinstalacion

OpenCode proporciona un comando de desinstalacion que maneja la mayoria de los casos de uso de forma predecible:

```bash
opencode uninstall
```

Por defecto, este comando elimina el binario de OpenCode y limpia los archivos de cache, pero preserva los datos de sesiones y la configuracion del usuario para facilitar una reinstalacion posterior. Este comportamiento se controla mediante flags explicitos:

- `--keep-config`: conserva la configuracion en `~/.config/opencode/`
- `--keep-data`: conserva las sesiones y datos locales en `~/.local/share/opencode/`
- `--dry-run`: muestra que archivos se eliminarian sin ejecutar la accion
- `--force`: ejecuta la desinstalacion sin confirmacion interactiva

Si OpenCode se instalo mediante npm, Homebrew o cualquier otro gestor de paquetes, es recomendable utilizar el comando de desinstalacion nativo de ese gestor en lugar de `opencode uninstall`, ya que el gestor mantiene su propio registro de los archivos instalados:

```bash
npm uninstall -g @opencode/cli    # para npm
brew uninstall opencode            # para Homebrew
paru -R opencode                   # para AUR (Arch Linux)
```

La limpieza manual completa, util cuando se quiere eliminar todo rastro de OpenCode del sistema, implica eliminar los directorios de configuracion y datos que el comando `uninstall` preserva por defecto:

```bash
rm -rf ~/.config/opencode/ ~/.local/share/opencode/
```

El directorio `~/.config/opencode/` contiene los archivos de configuracion global (`opencode.json`), los agentes personalizados y las credenciales de proveedores. El directorio `~/.local/share/opencode/` almacena las sesiones, la base de datos interna y la cache de modelos. Ambos directorios se recrean automaticamente si Opencode se reinstala posteriormente.

