# Entrada principal

Toda aplicacion Java de consola requiere un metodo de entrada que sirva como punto de inicio de la ejecucion. Este metodo tiene una firma especifica y estricta que la JVM reconoce automaticamente al iniciar.

## La firma del metodo main

```java
public static void main(String[] args) {
    // codigo de la aplicacion
}
```

Cada palabra de esta firma tiene un proposito preciso y su ausencia genera un error en tiempo de ejecucion:

**public** es necesario porque la JVM debe invocar este metodo desde fuera de la clase. Si el metodo tuviera un nivel de acceso menor (protected, default o private), la JVM no tendria permiso para acceder a el al iniciar la ejecucion.

**static** permite invocar el metodo sin crear una instancia de la clase. La JVM necesita un punto de entrada antes de que exista cualquier objeto. Si el metodo no fuera estatico, la JVM tendria que instanciar la clase primero, lo que generaria un problema circular: para instanciar la clase necesitaria resolver el constructor, que podria depender de logica que aun no se ha inicializado.

**void** indica que el metodo no retorna ningun valor. La JVM no espera recibir ningun dato de vuelta del metodo main. Si se necesita comunicar el resultado de la ejecucion, se recurre a otros mecanismos como codigos de salida del proceso o archivos de salida.

**String[] args** es un array de cadenas que contiene los argumentos pasados por linea de comandos al ejecutar el programa. Si no se proporcionan argumentos, el array existe pero esta vacio (longitud 0, no null).

```java
public class EjemploArgumentos {
    public static void main(String[] args) {
        System.out.println("Numero de argumentos: " + args.length);
        for (int i = 0; i < args.length; i++) {
            System.out.println("Argumento " + i + ": " + args[i]);
        }
    }
}
```

```bash
# Ejecucion con argumentos
java EjemploArgumentos hola mundo 42

# Salida
Numero de argumentos: 3
Argumento 0: hola
Argumento 1: mundo
Argumento 2: 42
```

> Los argumentos se pasan como cadenas de texto. Si se necesita un tipo distinto (entero, double, boolean), debe realizarse la conversion explicita dentro del metodo main.

## Paso de argumentos

Los argumentos de linea de comandos se separan por espacios. Si un argumento contiene espacios, debe encerrarse entre comillas dobles o comillas simples.

```bash
# Argumento con espacios
java MiApp "primer argumento" segundo tercero

# Argumentos en un IDE (IntelliJ, Eclipse, VS Code)
# Se configuran en la configuracion de ejecucion Run Configuration
# en el campo "Program arguments" o "Arguments"
```

Un patron comun es validar que se hayan proporcionado los argumentos necesarios antes de proceder:

```java
public class ProcesadorArchivos {
    public static void main(String[] args) {
        if (args.length < 1) {
            System.err.println("Uso: java ProcesadorArchivos <archivo>");
            System.exit(1);
        }

        String rutaArchivo = args[0];
        // procesar el archivo...
    }
}
```

## Metodos main alternativos

Desde Java 7, la JVM es flexible con la firma del metodo main. Ademas de la forma clasica, acepta varargs:

```java
// Forma alternativa equivalente (Java 7+)
public static void main(String... args) {
    // el compilador genera el mismo bytecode
}
```

La forma `String... args` (varargs) es sintacticamente equivalente a `String[] args`. El compilador convierte internamente el varargs en un array, por lo que no hay diferencia en comportamiento ni rendimiento.

Desde Java 21, el JEP 463 (parte de la preview de unnamed variables) no modifica el main, pero la tendencia del lenguaje apunta hacia simplificaciones futuras del metodo de entrada.

> En aplicaciones con modulos (JPMS), la clase que contiene el metodo main puede no ser accesible desde el modulo raiz. En esos casos, se debe declarar la clase como `public` y el paquete debe ser exportado por el modulo.

## Flujo de ejecucion

Cuando se ejecuta `java MiClase`, la JVM realiza las siguientes operaciones:

1. Carga la clase `MiClase` usando el ClassLoader de sistema.
2. Resuelve las dependencias de clases necesarias.
3. Ejecuta los bloques de inicializacion estatica de la clase.
4. Invoca el metodo `public static void main(String[])`.
5. El hilo principal ejecuta el codigo del metodo main.
6. Cuando el metodo main termina (o llama a `System.exit()`), la JVM finaliza todos los hilos y termina el proceso.

Si la clase no contiene un metodo main con la firma correcta, la JVM lanza un error:

```
Error: Main method not found in class MiClase, please define the main method as:
   public static void main(String[] args)
```
