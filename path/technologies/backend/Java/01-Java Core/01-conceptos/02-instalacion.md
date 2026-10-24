# Instalacion

Para desarrollar en Java se requiere instalar el JDK (Java Development Kit), el paquete completo que incluye las herramientas necesarias para compilar, depurar y ejecutar programas. Comprender las diferencias entre los componentes de la plataforma Java es esencial para configurar correctamente el entorno de desarrollo.

## JDK, JRE y JVM

El ecosistema Java se estructura en capas anidadas. La JVM (Java Virtual Machine) es el motor de ejecucion: interpreta el bytecode y lo traduce a codigo nativo. El JRE (Java Runtime Environment) es la JVM mas las bibliotecas de clases estandar necesarias para ejecutar programas Java, pero no incluye herramientas de desarrollo. El JDK (Java Development Kit) contiene el JRE completo mas el compilador (`javac`), el generador de documentacion (`javadoc`), el depurador (`jdb`) y otras utilidades como `jconsole` y `jhsdb`.

> A partir de Java 9, Oracle elimino el JRE como distribucion independiente. El JDK se distribuye en variantes de tipo "modular" donde solo se incluyen los modulos necesarios, reduciendo significativamente el tamano del runtime.

## Donde descargar

Oracle distribuye el JDK oficial a traves de su pagina de descargas, pero existen alternativas gratuitas y abiertas igual de validas:

- **Oracle JDK:** La distribucion comercial con soporte oficial. Disponible en https://www.oracle.com/java/technologies/downloads/.
- **OpenJDK:** La implementacion de referencia de codigo abierto, mantenido por la comunidad. Es la base sobre la que se construyen todas las demas distribuciones.
- **Eclipse Adoptium (Temurin):** Binarios precompilados de OpenJDK con pruebas de compatibilidad adicionales. Ideal para entornos de produccion.
- **GraalVM:** Una JVM de alto rendimiento que incluye un compilador natico (AOT), soporte para multiples lenguajes y optimizaciones avanzadas.

## Variables de entorno

Tras instalar el JDK, es necesario configurar dos variables de entorno para que el sistema operativo pueda localizar las herramientas de Java.

**JAVA_HOME** debe apuntar al directorio raiz de la instalacion del JDK. Esta variable es utilizada por herramientas como Maven, Gradle, IDEs y servidores de aplicaciones para localizar la instalacion de Java.

```bash
# Linux/macOS - agregar al archivo ~/.bashrc o ~/.zshrc
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

# Windows (PowerShell)
[System.Environment]::SetEnvironmentVariable("JAVA_HOME", "C:\Program Files\Java\jdk-21", "User")
[System.Environment]::SetEnvironmentVariable("Path", "$env:JAVA_HOME\bin;$env:Path", "User")
```

**PATH** debe incluir el directorio `bin` del JDK para que los comandos `java`, `javac` y demas herramientas sean accesibles desde cualquier ubicacion en la terminal.

## Verificar la instalacion

Una vez configuradas las variables, se verifican los comandos esenciales:

```bash
# Verificar la version del runtime
java -version

# Salida esperada (ejemplo con OpenJDK 21)
openjdk version "21.0.2" 2024-01-16
OpenJDK Runtime Environment (build 21.0.2+13-58)
OpenJDK 64-Bit Server VM (build 21.0.2+13-58, mixed mode, sharing)

# Verificar la version del compilador
javac -version

# Salida esperada
javac 21.0.2
```

La diferencia entre ambos comandos es significativa: `java -version` muestra la version del runtime (JVM y bibliotecas), mientras que `javac -version` muestra la version del compilador. En una instalacion completa del JDK, ambas deben coincidir en version.

## JDKs alternativos

Las distribuciones alternativas a Oracle JDK son completamente compatibles y en muchos casos preferidas:

**OpenJDK** es la implementacion de referencia de Java, mantenida como proyecto de codigo abierto bajo la GNU General Public License. Es la base sobre la que Oracle construye su JDK comercial y todas las demas distribuciones. La diferencia principal con Oracle JDK radica en las actualizaciones de seguridad y soporte comercial, no en las funcionalidades del lenguaje.

**Eclipse Adoptium** (anteriormente AdoptOpenJDK) ofrece binarios probados y certificados con el TCK (Technology Compatibility Kit) de Java. Su distribucion Temurin es ampliamente utilizada en entornos de produccion por su estabilidad y ciclo de lanzamiento predecible.

**GraalVM** representa una evolucion significativa de la plataforma JVM. Ademas de ejecutar bytecode Java tradicional, puede compilar codigo Java a binarios nativos sin necesidad de JVM en tiempo de ejecucion a traves de la herramienta `native-image`. Esto produce ejecutables de arranque rapido con bajo consumo de memoria, ideales para microservicios y funciones serverless.

```bash
# Verificar GraalVM
gu --version
native-image --version

# Compilar a nativo
native-image -jar mi-app.jar
```

> GraalVM es compatible con multiples lenguajes (JavaScript, Python, Ruby, R) y permite la interoperabilidad entre ellos en la misma JVM, lo que lo convierte en una plataforma poliglota de referencia.
