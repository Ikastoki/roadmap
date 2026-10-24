# Anotaciones

Las anotaciones son una forma de metadatos sintacticos que pueden anadirse a declaraciones de clases, metodos, campos, parametros y otras construcciones del lenguaje. No afectan directamente la semantica del programa, pero pueden ser procesadas por el compilador, herramientas de desarrollo o en tiempo de ejecucion via reflexion.

## Anotaciones predefinidas del compilador

### @Override

Indica que un metodo sobrescribe uno de la superclase o implementa uno de una interfaz. El compilador verifica que el metodo realmente sobrescribe otro; si no, genera un error.

```java
@Override
public String toString() {
    return "MiClase";
}
```

Previene errores como el nombre incorrecto del metodo o firma incompatible.

### @Deprecated

Marca un elemento como obsoleto. El compilador emite una advertencia si se usa.

```java
@Deprecated
public void metodoViejo() {
    // codigo legacy
}
```

```java
@Deprecated(since = "2.0", forRemoval = true)
public void metodoRemover() {
    // se eliminara en futura version
}
```

`since` indica la version desde la que esta deprecado. `forRemoval = true` indica que se eliminara en el futuro.

### @SuppressWarnings

Suprime advertencias especificas del compilador.

```java
@SuppressWarnings("unchecked")
public void metodo() {
    List raw = new ArrayList();
    raw.add("sin chequeo de tipo");
}
```

```java
@SuppressWarnings({"unchecked", "deprecation", "rawtypes"})
public void metodoConVariasAdvertencias() {
    // ...
}
```

### @SafeVarargs

Suprime advertencias sobre operaciones potencialmente inseguras con varargs genericos.

```java
@SafeVarargs
public static <T> List<T> listar(T... elementos) {
    return Arrays.asList(elementos);
}
```

## @FunctionalInterface

Marca una interfaz como interfaz funcional (una unica metodo abstracto). El compilador verifica que cumple el requisito.

```java
@FunctionalInterface
public interface Comparador<T> {
    int comparar(T a, T b);
}
```

## Anotaciones personalizadas

### Definicion

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Audit {
    String accion();
    String nivel() default "INFO";
}
```

Uso:

```java
public class Servicio {
    @Audit(accion = "CREAR_USUARIO", nivel = "CRITICO")
    public void crearUsuario() {
        // ...
    }
}
```

### Elementos de anotaciones

Los elementos se declaran como metodos abstractos en la interfaz de la anotacion. Pueden tener valores por defecto con `default`.

```java
public @interface Config {
    String valor();                  // requerido
    String entorno() default "dev";  // opcional
    int timeout() default 5000;
    boolean habilitado() default true;
}
```

### Tipos permitidos en elementos

- Primitivos: `int`, `long`, `double`, `boolean`, etc.
- `String`
- `Class` (con o sin genericos)
- Enums
- Otras anotaciones
- Arrays de los tipos anteriores

## Meta-anotaciones

### @Retention

Especifica hasta que punto esta disponible la anotacion.

| Politica | Disponibilidad |
|----------|---------------|
| `RetentionPolicy.SOURCE` | Solo en codigo fuente, descartada por el compilador |
| `RetentionPolicy.CLASS` | En el archivo .class (por defecto), no accesible en runtime |
| `RetentionPolicy.RUNTIME` | Accesible en tiempo de ejecucion via reflexion |

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Procesable {}
```

### @Target

Restringe los tipos de declaracion a los que puede aplicarse la anotacion.

| ElementType | Destino |
|-------------|---------|
| `TYPE` | Clases, interfaces, enums |
| `FIELD` | Campos (incluyendo constantes enum) |
| `METHOD` | Metodos |
| `PARAMETER` | Parametros |
| `CONSTRUCTOR` | Constructores |
| `LOCAL_VARIABLE` | Variables locales |
| `ANNOTATION_TYPE` | Otras anotaciones |
| `PACKAGE` | Paquetes |
| `TYPE_PARAMETER` | Parametros de tipo generico (Java 8+) |
| `TYPE_USE` | Usos de tipo (Java 8+) |

```java
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface Especifica {}
```

### @Inherited

Permite que una anotacion de una superclase sea heredada por sus subclases. Solo funciona para clases, no para interfaces.

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface Version {
    String valor();
}

@Version("1.0")
public class Base {}

public class Derivada extends Base {} // hereda @Version("1.0")
```

### @Documented

Incluye la anotacion en la documentacion Javadoc.

```java
@Documented
public @interface API {}
```

### @Repeatable (Java 8+)

Permite aplicar la misma anotacion multiples veces al mismo elemento.

```java
@Repeatable(Notificaciones.class)
public @interface Notificacion {
    String canal();
}

public @interface Notificaciones {
    Notificacion[] value();
}
```

```java
@Notificacion(canal = "email")
@Notificacion(canal = "sms")
public void enviar() {}
```

## Procesamiento de anotaciones en runtime

Con `@Retention(RUNTIME)`, las anotaciones pueden procesarse via reflexion.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Ejecutar {
    int orden() default 0;
}
```

```java
public class Tareas {
    @Ejecutar(orden = 2)
    public void tareaA() { System.out.println("A"); }

    @Ejecutar(orden = 1)
    public void tareaB() { System.out.println("B"); }

    @Ejecutar(orden = 3)
    public void tareaC() { System.out.println("C"); }
}
```

```java
public class Procesador {
    public static void ejecutar(Object obj) throws Exception {
        for (Method m : obj.getClass().getDeclaredMethods()) {
            if (m.isAnnotationPresent(Ejecutar.class)) {
                Ejecutar anot = m.getAnnotation(Ejecutar.class);
                System.out.println("Ejecutando: " + m.getName() +
                    " (orden " + anot.orden() + ")");
                m.invoke(obj);
            }
        }
    }
}
```

## Procesamiento en tiempo de compilacion (APT)

Las anotaciones con `RetentionPolicy.SOURCE` o `CLASS` pueden procesarse en tiempo de compilacion usando `javax.annotation.processing.Processor`. Es el mecanismo usado por frameworks como Lombok para generar codigo.

```java
@SupportedAnnotationTypes("com.ejemplo.GenerarBuilder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (Element elem : roundEnv.getElementsAnnotatedWith(GenerarBuilder.class)) {
            // generar codigo fuente
        }
        return true;
    }
}
```
