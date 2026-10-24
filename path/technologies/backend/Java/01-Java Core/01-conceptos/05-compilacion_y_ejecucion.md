# Compilacion y ejecucion

El modelo de compilacion de Java sigue un proceso de dos fases que lo distingue de lenguajes compilados nativos e interpretados puros. Este diseño es la base del principio "Write Once, Run Anywhere" y permite la portabilidad entre plataformas.

## Flujo completo

Un programa Java experimenta las siguientes etapas desde su escritura hasta su ejecucion:

1. **Archivo fuente (.java):** El programador escribe codigo en un archivo de texto con extension `.java`. Cada archivo puede contener una unica clase publica (ademas de clases no publicas auxiliares).

2. **Compilacion (javac):** El compilador Java (`javac`) analiza el codigo fuente, verifica la sintaxis, resuelve referencias entre clases, aplica restricciones de tipos y genera el codigo intermedio conocido como bytecode. El resultado es un archivo `.class` por cada clase definida.

3. **Bytecode (.class):** Los archivos `.class` contienen instrucciones para la JVM, no para un procesador fisico. Este formato es independiente de la plataforma y representa el contrato entre el codigo fuente y la maquina virtual.

4. **Ejecucion (java):** La JVM carga los archivos `.class` en memoria, verifica que el bytecode sea seguro y valido, y comienza a ejecutarlo. Un compilador JIT (Just-In-Time) optimiza las partes criticas del codigo durante la ejecucion.

```java
// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hola, Mundo");
    }
}
```

```bash
# Compilacion
javac HelloWorld.java

# Ejecucion
java HelloWorld
```

> El compilador javac no genera codigo maquina, sino bytecode. Esto significa que un archivo `.class` generado en Windows puede ejecutarse identicamente en Linux o macOS, siempre que exista una JVM compatible.

## Paquetes

Los paquetes son mecanismos de organizacion que agrupan clases relacionadas bajo un namespace jerarquico. La convencion utiliza nombres de dominio invertidos para evitar colisiones: `com.empresa.modulo.submodulo`.

```java
// Declaracion al inicio del archivo
package com.empresa.aplicacion.modelo;

public class Usuario {
    // ...
}
```

La estructura de directorios debe reflejar la declaracion del paquete. El archivo anterior debe ubicarse en:

```
com/empresa/aplicacion/modelo/Usuario.java
```

Cuando se compila, se genera la misma estructura de directorios para los archivos `.class`:

```bash
javac -d salida com/empresa/aplicacion/modelo/Usuario.java
# Genera: salida/com/empresa/aplicacion/modelo/Usuario.class
```

El directorio `-d` especifica donde se genera la estructura de paquetes compilados. Sin esta opcion, los archivos `.class` se crean en el directorio actual.

## Classpath

El classpath es la ruta que la JVM utiliza para localizar las clases necesarias durante la ejecucion. Puede configurarse de varias maneras:

```bash
# Mediante variable de entorno
export CLASSPATH=/ruta/a/librerias/mi-lib.jar:/ruta/a/clases

# Mediante flag -cp en la ejecucion (recomendado)
java -cp /ruta/a/librerias/mi-lib.jar:/ruta/a/clases com.empresa.App

# Incluir todos los JARs de un directorio
java -cp "lib/*" com.empresa.App
```

Cuando la JVM necesita cargar una clase, busca en el classpath en el siguiente orden: directorios de clases, archivos JAR (archivos ZIP que agrupan multiples `.class`) y modulos del modulo path. Si no encuentra la clase en ningun lugar, lanza `ClassNotFoundException`.

> En JDKs modernos, el concepto de classpath esta siendo complementado por el modulo path (module path) del Sistema de Modulos (JPMS), introducido en Java 9. Los modulos proporcionan un nivel adicional de encapsulacion y control de dependencias.

## Compilacion desde terminal

El proceso completo de compilacion y ejecucion desde la linea de comandos implica gestionar paquetes, dependencias y classpath:

```bash
# Compilar todos los archivos .java de un directorio recursivamente
javac -d salida $(find src -name "*.java")

# Compilar con dependencias en un JAR
javac -cp "lib/gson-2.10.jar:lib/commons-lang3-3.12.jar" -d salida src/**/*.java

# Ejecutar con multiples JARs en el classpath
java -cp "salida:lib/gson-2.10.jar:lib/commons-lang3-3.12.jar" com.empresa.App

# Ejecutar un JAR directamente (requiere manifest con Main-Class)
java -jar mi-aplicacion.jar
```

El separador del classpath varia segun el sistema operativo: `:` en Unix/Linux/macOS y `;` en Windows.

```bash
# Windows
java -cp "salida;lib/gson-2.10.jar" com.empresa.App

# Linux/macOS
java -cp "salida:lib/gson-2.10.jar" com.empresa.App
```

## Ejecucion de JARs

Un archivo JAR (Java ARchive) es un archivo ZIP que contiene clases compiladas, metadatos y recursos. Para ejecutar un JAR directamente, este debe contener un manifest (archivo `META-INF/MANIFEST.MF`) que declare la clase principal:

```bash
# Crear un JAR ejecutable
jar cfe mi-app.jar com.empresa.App -C salida .

# Ejecutar el JAR
java -jar mi-app.jar

# Inspeccionar el contenido de un JAR
jar tf mi-app.jar
```

> Los JARs ejecutables son el formato estandar para distribuir aplicaciones Java. Las herramientas de build como Maven y Gradle generan automaticamente el manifest con la clase principal configurada en el script de compilacion.
