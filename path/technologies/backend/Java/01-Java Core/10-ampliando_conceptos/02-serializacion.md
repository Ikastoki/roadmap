# Serializacion

La serializacion en Java es el proceso de convertir el estado de un objeto en una secuencia de bytes que puede persistirse (archivo, base de datos) o transmitirse (red, mensajeria). La deserializacion reconstruye el objeto a partir de esos bytes. Es un mecanismo integrado en la JVM que requiere que la clase implemente la interfaz marcadora `Serializable`.

## Serializable

`Serializable` es una interfaz de marcado (sin metodos) en `java.io`. Una clase que la implementa declara que puede ser serializada.

```java
import java.io.Serializable;

public class Persona implements Serializable {
    private static final long serialVersionUID = 1L;
    private String nombre;
    private int edad;
    // ... constructores, getters, setters
}
```

Si una clase es `Serializable`, todas sus subclases tambien lo son. Si una clase no es `Serializable` pero se intenta serializar una subclase, se lanza `NotSerializableException`.

## serialVersionUID

`serialVersionUID` es un identificador unico de version para cada clase `Serializable`. Se usa durante la deserializacion para verificar que el remitente y el receptor tienen clases compatibles.

```java
private static final long serialVersionUID = 1L;
```

Si no se declara, la JVM lo genera automaticamente basandose en los detalles de la clase (metodos, campos, interfaces). Esto es frgil: cualquier cambio en la clase cambia el UID generado, provocando `InvalidClassException` al deserializar.

> El compilador advierte si una clase `Serializable` no declara `serialVersionUID`. Siempre debe declararse explicitamente para control de versiones.

### Compatibilidad de versiones

- **Cambio compatible**: anadir campos, metodos, constructores. `serialVersionUID` se mantiene y los campos faltantes se inicializan con valores por defecto.
- **Cambio incompatible**: eliminar campos, cambiar tipo de campo. Debe cambiarse `serialVersionUID` (o la deserializacion fallara).

## Transient

La palabra clave `transient` marca campos que no deben serializarse. Al deserializar, estos campos reciben su valor por defecto (`null` para objetos, 0 para numeros, `false` para boolean).

```java
public class Usuario implements Serializable {
    private static final long serialVersionUID = 1L;
    private String nombre;
    private transient String password; // no se serializa
    private transient Connection dbConnection; // no serializable
}
```

Campos tipicamente marcados como `transient`:

- Contrasenas y datos sensibles.
- Conexiones de red, sockets, streams.
- Referencias a objetos no serializables.
- Cache o datos derivados que pueden recalcularse.

## ObjectOutputStream y ObjectInputStream

La serializacion se realiza mediante flujos (`ObjectOutputStream` para escritura, `ObjectInputStream` para lectura).

```java
// Serializar a archivo
Persona p = new Persona("Ana", 30);
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("persona.ser"))) {
    oos.writeObject(p);
} catch (IOException e) {
    e.printStackTrace();
}
```

```java
// Deserializar desde archivo
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("persona.ser"))) {
    Persona p = (Persona) ois.readObject();
    System.out.println(p.getNombre());
} catch (IOException | ClassNotFoundException e) {
    e.printStackTrace();
}
```

### Flujo de serializacion

1. Se abre `ObjectOutputStream` sobre un `OutputStream`.
2. `writeObject()` inicia el proceso: escribe el `serialVersionUID`, la clase del objeto, y recursivamente serializa los campos no `transient`.
3. Si un campo es una referencia a otro objeto serializable, se serializa tambien (grafo de objetos).
4. La JVM mantiene un registro de objetos ya serializados para evitar duplicados y ciclos.

### Flujo de deserializacion

1. Se abre `ObjectInputStream` sobre un `InputStream`.
2. `readObject()` lee la clase y verifica que el `serialVersionUID` coincida.
3. Se crea una nueva instancia sin llamar al constructor (usando mecanismos nativos de la JVM).
4. Los campos se rellenan con los valores del stream; los `transient` quedan con valores por defecto.

## Serializacion personalizada

### writeObject y readObject

Las clases pueden definir metodos privados `writeObject()` y `readObject()` para controlar el proceso.

```java
public class Configuracion implements Serializable {
    private static final long serialVersionUID = 1L;
    private transient String token;
    private long expiracion;

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject(); // serializa campos no transient
        // Serializar token manualmente (ej. encriptado)
        oos.writeObject(encriptar(token));
    }

    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject(); // deserializa campos no transient
        this.token = desencriptar((String) ois.readObject());
    }
}
```

### readResolve

`readResolve()` permite reemplazar el objeto deserializado por otro. Es crucial para singletons:

```java
public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() { return INSTANCE; }

    protected Object readResolve() {
        return INSTANCE; // asegura que solo existe una instancia
    }
}
```

### writeReplace

`writeReplace()` permite especificar un objeto sustituto para la serializacion:

```java
private Object writeReplace() throws ObjectStreamException {
    return new ObjetoProxy(this); // se serializa el proxy en lugar del original
}
```

## Colecciones y serializacion

Las colecciones estandar de Java (`ArrayList`, `HashMap`, etc.) son serializables si sus elementos tambien lo son.

```java
List<Persona> lista = new ArrayList<>();
// Si Persona es Serializable, la lista se puede serializar
```

## Consideraciones de seguridad

- La deserializacion ejecuta codigo: `readObject()` puede ser explotado para ataques de deserializacion.
- Filtrar clases permitidas con `ObjectInputFilter` (Java 9+):

```java
ObjectInputStream ois = new ObjectInputStream(fis);
ois.setObjectInputFilter(filterInfo -> {
    if (filterInfo.serialClass() == null) return Status.UNDECIDED;
    if (filterInfo.serialClass().getName().startsWith("com.seguro."))
        return Status.ALLOWED;
    return Status.REJECTED;
});
```

> Desde Java 17, la serializacion esta siendo evaluada como mecanismo legacy. Alternativas como JSON, XML, Protocol Buffers o Kryo ofrecen mejor control de versiones y seguridad.
