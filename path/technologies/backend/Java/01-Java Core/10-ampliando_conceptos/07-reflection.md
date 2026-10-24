# Reflection

La reflection (reflexion) es la capacidad de un programa para inspeccionarse y manipularse a si mismo en tiempo de ejecucion. Java proporciona la API `java.lang.reflect` que permite examinar clases, interfaces, metodos, campos y constructores, e invocar metodos o acceder a campos dinamicamente.

## Obtencion de informacion de clases

El punto de entrada a la reflection es el objeto `Class<?>`.

```java
// Tres formas de obtener un objeto Class
Class<String> c1 = String.class;
Class<?> c2 = Class.forName("java.lang.String");
Class<?> c3 = "hola".getClass();
```

```java
// Informacion de la clase
String nombre = c1.getName();        // "java.lang.String"
String simple = c1.getSimpleName();  // "String"
Package pkg = c1.getPackage();       // "java.lang"
Class<?> superClass = c1.getSuperclass();
Class<?>[] interfaces = c1.getInterfaces();
int modifiers = c1.getModifiers();
boolean esPublic = Modifier.isPublic(modifiers);
```

## Inspectores de clase

Son metodos en `Class<?>` que devuelven los miembros declarados o publicos:

| Metodo | Devuelve |
|--------|----------|
| `getMethods()` | Metodos publicos (incluyendo heredados) |
| `getDeclaredMethods()` | Todos los metodos declarados (privados, protegidos, publicos) |
| `getFields()` | Campos publicos (incluyendo heredados) |
| `getDeclaredFields()` | Todos los campos declarados |
| `getConstructors()` | Constructores publicos |
| `getDeclaredConstructors()` | Todos los constructores declarados |

```java
Class<?> clazz = Class.forName("com.ejemplo.MiClase");

Method[] metodos = clazz.getDeclaredMethods();
for (Method m : metodos) {
    System.out.println(m.getName() + " - " + m.getReturnType());
}

Field[] campos = clazz.getDeclaredFields();
for (Field f : campos) {
    System.out.println(f.getName() + " : " + f.getType());
}
```

## Acceso a metodos y campos

### Inspeccion de metodos

```java
Class<?> clazz = Class.forName("java.util.ArrayList");

// Obtener un metodo especifico por nombre y tipos de parametros
Method metodoSize = clazz.getMethod("size");
Method metodoAdd = clazz.getMethod("add", Object.class);

// Informacion del metodo
String nombre = metodoAdd.getName();
Class<?> retorno = metodoAdd.getReturnType();
Class<?>[] paramTipos = metodoAdd.getParameterTypes();
int mod = metodoAdd.getModifiers();
boolean esPublic = Modifier.isPublic(mod);
```

### Inspeccion de campos

```java
class Persona {
    private String nombre;
    public int edad;
}

Class<?> clazz = Persona.class;

Field campoEdad = clazz.getField("edad");          // solo publicos
Field campoNombre = clazz.getDeclaredField("nombre"); // cualquier visibilidad

int mod = campoNombre.getModifiers();
boolean esPrivado = Modifier.isPrivate(mod);
```

### Inspeccion de constructores

```java
Constructor<?>[] constructores = clazz.getDeclaredConstructors();
for (Constructor<?> c : constructores) {
    Class<?>[] paramTypes = c.getParameterTypes();
    int count = c.getParameterCount();
}
```

## Invocacion dinamica de metodos

```java
public class Calculadora {
    public int sumar(int a, int b) {
        return a + b;
    }

    private int multiplicar(int a, int b) {
        return a * b;
    }
}
```

```java
Class<?> clazz = Calculadora.class;
Object instancia = clazz.getDeclaredConstructor().newInstance();

// Invocar metodo publico
Method metodoSumar = clazz.getMethod("sumar", int.class, int.class);
int resultado = (int) metodoSumar.invoke(instancia, 3, 4);
System.out.println(resultado); // 7

// Invocar metodo privado
Method metodoMultiplicar = clazz.getDeclaredMethod("multiplicar", int.class, int.class);
metodoMultiplicar.setAccessible(true); // supera el control de acceso
int resultado2 = (int) metodoMultiplicar.invoke(instancia, 3, 4);
System.out.println(resultado2); // 12
```

## Acceso a campos dinamico

```java
class Configuracion {
    public String entorno = "produccion";
    private String password = "secreto";
}
```

```java
Configuracion config = new Configuracion();
Class<?> clazz = config.getClass();

// Leer campo publico
Field campoEntorno = clazz.getField("entorno");
String entorno = (String) campoEntorno.get(config);
System.out.println(entorno); // "produccion"

// Leer campo privado
Field campoPassword = clazz.getDeclaredField("password");
campoPassword.setAccessible(true);
String password = (String) campoPassword.get(config);
System.out.println(password); // "secreto"

// Modificar campo
campoEntorno.set(config, "desarrollo");
```

## Creacion de instancias

```java
// Usando Constructor
Constructor<Calculadora> constructor = Calculadora.class.getDeclaredConstructor();
Calculadora calc = constructor.newInstance();

// Con parametros (si existe un constructor con parametros)
Constructor<?> constructorConParam = clazz.getConstructor(String.class, int.class);
Object instancia = constructorConParam.newInstance("test", 42);
```

## Arrays via reflection

```java
Class<?> arrayClass = Class.forName("[Ljava.lang.String;"); // array de String
Object array = Array.newInstance(String.class, 5);
Array.set(array, 0, "elemento1");
Array.set(array, 1, "elemento2");
String elem = (String) Array.get(array, 0);
int length = Array.getLength(array);
```

## Proxies dinamicos

`java.lang.reflect.Proxy` permite crear implementaciones dinamicas de interfaces en tiempo de ejecucion.

```java
public interface Servicio {
    void ejecutar();
}

Servicio proxy = (Servicio) Proxy.newProxyInstance(
    Servicio.class.getClassLoader(),
    new Class<?>[] { Servicio.class },
    (Object proxyObj, Method method, Object[] args) -> {
        System.out.println("Antes de " + method.getName());
        Object resultado = method.invoke(objetoReal, args);
        System.out.println("Despues de " + method.getName());
        return resultado;
    }
);
```

## setAccessible y riesgos

`setAccessible(true)` permite acceder a miembros privados, protegidos o package-private, superando las restricciones del lenguaje.

```java
Field campoPrivado = clazz.getDeclaredField("secreto");
campoPrivado.setAccessible(true); // desactiva la comprobacion del lenguaje
String valor = (String) campoPrivado.get(instancia);
```

Desde Java 9 (modulos), si la clase esta en un modulo que no abre el paquete, `setAccessible` puede lanzar `InaccessibleObjectException`. Se requiere la opcion de JVM `--add-opens` o declaraciones en `module-info.java`:

```java
// module-info.java del modulo cliente
module mi.modulo {
    requires java.base;
}
```

```bash
# JVM option para abrir un paquete
--add-opens java.base/java.lang=ALL-UNNAMED
```

## Consideraciones de rendimiento

La reflection tiene costes significativos:

- Las llamadas via `Method.invoke()` son mucho mas lentas que llamadas directas (hasta 100x).
- `setAccessible()` requiere permisos del SecurityManager.
- El compilador JIT no puede optimizar llamadas reflectivas tan eficazmente.
- `getMethod()`, `getField()` recorren todos los miembros de la clase.

```java
// Estrategia de cache para mejorar rendimiento
private static final Map<Class<?>, Method[]> CACHE = new ConcurrentHashMap<>();

public static Method[] getCachedMethods(Class<?> clazz) {
    return CACHE.computeIfAbsent(clazz, Class::getDeclaredMethods);
}
```

## Riesgos de seguridad

- Violacion del encapsulamiento: acceso a campos privados y metodos internos.
- Invocacion de metodos que deberian estar ocultos.
- Modificacion de campos final (aunque `setAccessible` puede fallar dependiendo del JVM).
- Posible inyeccion de codigo si los nombres de clase o metodo provienen de entrada externa.

```java
// Riesgo: usar reflection con entrada de usuario
String nombreClase = obtenerEntradaUsuario(); // potencialmente malicioso
Class<?> clasePeligrosa = Class.forName(nombreClase);
Method metodoPeligroso = clasePeligrosa.getMethod("ejecutarComando", String.class);
metodoPeligroso.invoke(null, comando);
```

> La reflection es una herramienta poderosa pero debe usarse con prudencia. Es adecuada para frameworks, bibliotecas y herramientas de desarrollo, pero rara vez es necesaria en codigo de aplicacion.
