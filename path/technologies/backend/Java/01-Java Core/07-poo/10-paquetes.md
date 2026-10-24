# Paquetes

Los paquetes (packages) en Java son el mecanismo de organizacion de clases e interfaces. Proporcionan un espacio de nombres que evita colisiones entre clases con el mismo nombre, y establecen el alcance del acceso package-private. La estructura de paquetes refleja tipicamente la organizacion del proyecto y la jerarquia de modulos.

## package

Todo archivo fuente de Java puede comenzar con una declaracion `package` que indica a que paquete pertenecen las clases definidas en el archivo. Si no se especifica, las clases pertenecen al paquete por defecto (no recomendado para proyectos reales).

```java
package com.ejemplo.sistema.usuarios;

public class UsuarioService {
    // ...
}
```

La declaracion `package` debe ser la primera linea del archivo (antes de cualquier `import`). El nombre del paquete se corresponde con la estructura de directorios: `com.ejemplo.sistema.usuarios` espera que el archivo este en `com/ejemplo/sistema/usuarios/`.

Las convenciones de nomenclatura de paquetes siguen el dominio de la organizacion en orden inverso:

```
com.ejemplo.app
org.fundacion.proyecto
net.empresa.producto
edu.universidad.facultad
```

## import

La declaracion `import` permite usar clases de otros paquetes sin escribir el nombre completamente calificado (fully qualified name). Pueden importarse clases individuales o paquetes enteros con el comodin `*`.

```java
package com.ejemplo.app;

// Import de una clase especifica
import java.util.ArrayList;
import java.util.List;

// Import de todas las clases de un paquete
import java.io.*;

// Import estatico (Java 5+)
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;
```

Sin `import`, cada referencia a una clase externa requiere el nombre completo:

```java
java.util.List<String> items = new java.util.ArrayList<>();
```

Con `import`:

```java
import java.util.List;
import java.util.ArrayList;

List<String> items = new ArrayList<>();
```

> El import comodin (`*`) no afecta al rendimiento ni al tamano del codigo compilado. Solo es una cuestion de estilo y legibilidad. Sin embargo, muchos estilos de codigo prefieren imports explicitos para saber exactamente que clases se estan usando.

## Organizacion de paquetes

Un proyecto Java tipico organiza sus paquetes siguiendo una estructura logica que refleja la arquitectura:

```
com.ejemplo.app/
    Main.java
    config/
        DatabaseConfig.java
        AppConfig.java
    modelo/
        Usuario.java
        Pedido.java
    repositorio/
        UsuarioRepository.java
        PedidoRepository.java
    servicio/
        UsuarioService.java
        PedidoService.java
    controlador/
        UsuarioController.java
```

Esta organizacion separa responsabilidades y facilita la navegacion. Cada capa tiene su propio paquete y las dependencias fluyen tipicamente de controlador hacia servicio, y de servicio hacia repositorio.

El nombre del paquete suele incluir el dominio de la empresa y el nombre del proyecto para garantizar unicidad global:

```
com.empresa.miembros

java.* vs javax.*
```

## java.* vs javax.*

La distincion historica entre `java.*` y `javax.*` se ha diluido con el tiempo. Originalmente, `java.*` contenía las APIs del nucleo de Java (Core Java) y `javax.*` contenía extensiones opcionales.

En la practica actual, ambos forman parte de la plataforma Java estandar (Java SE). No hay diferencia funcional entre ellos.

```java
import java.util.List;           // colecciones
import java.io.File;             // entrada/salida
import java.time.LocalDate;      // fecha y hora (Java 8)

import javax.swing.JButton;      // GUI (Swing)
import javax.net.ssl.SSLSocket;  // redes seguras
```

Con la modularizacion en Java 9 (Project Jigsaw), algunos paquetes `javax.*` fueron reubicados en modulos especificos. Por ejemplo, `javax.xml.*` paso a estar en `java.xml`.

## Module system (Java 9+)

Project Jigsaw introdujo el sistema de modulos (Java Platform Module System, JPMS) que anade un nivel de encapsulamiento por encima de los paquetes. Un modulo es un grupo de paquetes que declara explicitamente que exporta y que modulos requiere.

```java
// module-info.java
module com.ejemplo.app {
    requires java.sql;
    requires java.net.http;
    exports com.ejemplo.app.servicio;
    exports com.ejemplo.app.modelo;
}
```

El modulo declara:

- `requires`: que otros modulos necesita
- `exports`: que paquetes del modulo son accesibles desde fuera
- `opens`: permite reflection sobre paquetes especificos (para frameworks)
- `provides` / `uses`: para servicios (ServiceLoader)

Los paquetes no exportados no son accesibles desde fuera del modulo, incluso si sus clases son `public`. Esto proporciona un encapsulamiento mas fuerte que los modificadores de acceso tradicionales.

## Package-private access

El acceso package-private (default) es el nivel de visibilidad por defecto cuando no se especifica un modificador. Las clases, metodos y campos sin modificador son accesibles solo desde el mismo paquete.

```java
package com.ejemplo.nucleo;

// Clase package-private: solo visible dentro del paquete
class ConfiguracionInterna {
    String valor;

    void procesar() {
        // ...
    }
}
```

Este nivel de acceso es la base del diseno cohesivo por paquete: clases que colaboran estrechamente comparten el mismo paquete y pueden acceder a miembros package-private. Fuera del paquete, solo las clases `public` y sus miembros publicos son visibles.

```java
package com.ejemplo.nucleo;

public class Fachada {
    void operacionInterna() {
        ConfiguracionInterna cfg = new ConfiguracionInterna();
        cfg.procesar(); // accesible: mismo paquete
    }
}
```

El acceso package-private es especialmente util en modulos donde se quiere compartir implementacion entre varias clases sin exponerla al mundo exterior. Combinado con el systema de modulos de Java 9+, proporciona dos niveles de encapsulamiento: el modulo controla que paquetes exporta, y los modificadores de acceso controlan que miembros son visibles dentro de cada paquete.
