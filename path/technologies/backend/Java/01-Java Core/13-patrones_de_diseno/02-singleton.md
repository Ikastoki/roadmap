# Singleton

Singleton es un patron creacional que garantiza que una clase tenga una unica instancia y proporciona un punto de acceso global a ella. Es util cuando exactamente un objeto debe coordinar acciones en todo el sistema: gestores de configuracion, pools de conexiones, registros de servicios, o caches.

## Una instancia, constructor privado

El mecanismo central del patrón es impedir la creacion externa de instancias mediante un constructor privado. La unica forma de obtener la instancia es a traves de un metodo estatico factory.

```java
public class Configuracion {
    private static Configuracion instancia;
    private Properties propiedades;

    private Configuracion() {
        propiedades = new Properties();
        cargarArchivo();
    }

    public static Configuracion getInstancia() {
        if (instancia == null) {
            instancia = new Configuracion();
        }
        return instancia;
    }

    public String get(String clave) {
        return propiedades.getProperty(clave);
    }
}
```

Este ejemplo basico no es thread-safe: dos hilos pueden ejecutar `getInstancia()` simultaneamente y crear dos instancias distintas.

## Eager vs Lazy initialization

### Eager initialization

La instancia se crea al cargar la clase, garantizando seguridad en hilos de forma natural:

```java
public class Configuracion {
    private static final Configuracion INSTANCIA = new Configuracion();

    private Configuracion() {}

    public static Configuracion getInstancia() {
        return INSTANCIA;
    }
}
```

La JVM garantiza que la inicializacion de campos estaticos ocurra una sola vez, incluso con multiples hilos. El inconveniente es que la instancia se crea aunque nunca se use, lo que puede ser problematico si la inicializacion es costosa.

### Lazy initialization

La instancia se crea solo cuando se solicita por primera vez. La version basica no es thread-safe; la version sincronizada lo es pero introduce contención:

```java
public class Configuracion {
    private static Configuracion instancia;

    private Configuracion() {}

    public static synchronized Configuracion getInstancia() {
        if (instancia == null) {
            instancia = new Configuracion();
        }
        return instancia;
    }
}
```

El uso de `synchronized` en el metodo completo penaliza el rendimiento en cada llamada, incluso cuando la instancia ya existe.

## Thread-safe: double-checked locking

Double-checked locking reduce la sincronizacion al minimo necesario: solo se sincroniza cuando la instancia es null, y se verifica dos veces para evitar la condicion de carrera.

```java
public class Configuracion {
    private static volatile Configuracion instancia;

    private Configuracion() {}

    public static Configuracion getInstancia() {
        Configuracion resultado = instancia;
        if (resultado == null) {
            synchronized (Configuracion.class) {
                resultado = instancia;
                if (resultado == null) {
                    instancia = resultado = new Configuracion();
                }
            }
        }
        return resultado;
    }
}
```

La palabra clave `volatile` es esencial: sin ella, el compilador puede reordenar las operaciones de escritura del constructor, y un hilo podria ver una instancia parcialmente construida. Desde Java 5, `volatile` proporciona la barrera de memoria necesaria para que este patron funcione correctamente.

> Aunque double-checked locking es correcto en Java 5+, la complejidad del codigo rara vez se justifica. Las alternativas (eager initialization, holder pattern, enum singleton) son mas simples y igual de eficientes.

### Initialization-on-demand holder idiom

Un enfoque elegante que aprovecha la inicializacion perezosa de clases internas estaticas:

```java
public class Configuracion {
    private Configuracion() {}

    private static class Holder {
        private static final Configuracion INSTANCIA = new Configuracion();
    }

    public static Configuracion getInstancia() {
        return Holder.INSTANCIA;
    }
}
```

La clase `Holder` no se carga hasta que se accede a ella desde `getInstancia()`, logrando lazy initialization sin sincronizacion. Es la solucion mas recomendada para Singleton en Java.

## Enum Singleton

Desde Java 5, un enum con un unico valor proporciona un Singleton inherentemente seguro contra reflexion, serializacion y ataques de multiples instancias:

```java
public enum ConfiguracionSingleton {
    INSTANCIA;

    private Properties propiedades;

    ConfiguracionSingleton() {
        propiedades = new Properties();
        cargarArchivo();
    }

    public String get(String clave) {
        return propiedades.getProperty(clave);
    }
}

// Uso
String valor = ConfiguracionSingleton.INSTANCIA.get("db.url");
```

El enfoque enum garantiza:
- Inicializacion thread-safe garantizada por la JVM.
- Proteccion contra reflexion: los enums no pueden instanciarse via reflection.
- Serializacion segura: la serializacion de enums mantiene la garantia de instancia unica.
- Sintaxis concisa y clara.

> Joshua Bloch defiende el enum singleton como la mejor forma de implementar Singleton en Java. La unica limitacion es que no soporta lazy initialization en el sentido clasico (el enum se inicializa cuando se carga la clase por primera vez). Para la mayoria de los casos, esto no es un problema.

## Criticas al patrón Singleton

Singleton es frecuentemente criticado por introducir acoplamiento global y dificultar las pruebas unitarias. Una clase que depende directamente de `Configuracion.getInstancia()` no puede probarse con una configuracion diferente sin modificar el estado global.

Las alternativas incluyen inyeccion de dependencias (contenedores como Spring gestionan instancias unicas sin exponer el patron directamente) o pasar la instancia como parametro. Singleton sigue siendo util en contextos donde el acceso global esta justificado y no se requiere testabilidad extensiva.
