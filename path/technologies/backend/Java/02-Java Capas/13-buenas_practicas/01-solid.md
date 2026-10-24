# Principios SOLID aplicados a las capas

Los principios SOLID, acunados por Robert C. Martin, no son reglas abstractas de diseno de clases: son directrices que, cuando se aplican de manera consistente a lo largo de una arquitectura por capas, producen un sistema donde cada elemento tiene una responsabilidad clara, las dependencias apuntan en la direccion correcta y el impacto de los cambios queda acotado. Una aplicacion Java en capas que respeta SOLID es intrinsecamente mas mantenible, testeable y evolutiva que otra que los ignora.

## Single Responsibility (cada capa tiene un proposito)

El principio de responsabilidad unica (SRP) afirma que una clase debe tener una sola razon para cambiar. En la arquitectura por capas, este principio se eleva al nivel de componente: cada capa tiene exactamente una responsabilidad dentro del sistema.

La capa de presentacion cambia cuando cambian los requisitos de interfaz de usuario o de exposicion de la API. La capa de negocio cambia cuando cambian las reglas de negocio. La capa de persistencia cambia cuando cambia la tecnologia de almacenamiento. Si una misma clase mezcla, por ejemplo, logica de negocio con construccion de consultas SQL, recibe dos razones para cambiar, violando el SRP.

```java
// Bien: responsabilidad unica
@RestController
public class UsuarioController {
    private final UsuarioService service;
    
    @GetMapping("/usuarios/{id}")
    public UsuarioDTO obtener(@PathVariable Long id) {
        return service.buscarPorId(id);
    }
}

@Service
public class UsuarioService {
    private final UsuarioRepository repository;
    
    public UsuarioDTO buscarPorId(Long id) {
        Usuario usuario = repository.findById(id)
            .orElseThrow(() -> new RecursoNoEncontrado("Usuario", id));
        return UsuarioMapper.toDTO(usuario);
    }
}
```

Cada clase anterior cambia por un motivo distinto: el controlador por la forma de la API, el servicio por la logica de negocio. Ninguna de las dos sabe de SQL ni de HTTP al mismo tiempo.

## Open/Closed (abierto a extension, cerrado a modificacion)

El principio Open/Closed (OCP) establece que las entidades de software deben estar abiertas a extension pero cerradas a modificacion. Esto significa que debe ser posible anadir nuevo comportamiento sin alterar el codigo existente que ya funciona.

En la arquitectura por capas, OCP se materializa a traves de interfaces. La capa superior depende de una abstraccion (la interfaz), y la capa inferior proporciona implementaciones concretas. Para anadir una nueva implementacion --un repositorio que usa Redis en lugar de JPA, un servicio de notificaciones que usa SMS en lugar de email-- basta con crear una nueva clase que implemente la interfaz existente, sin tocar el codigo cliente.

```java
public interface NotificacionService {
    void enviar(String destino, String mensaje);
}

@Service
public class EmailNotificacionService implements NotificacionService {
    public void enviar(String destino, String mensaje) { /* ... */ }
}

@Service
@ConditionalOnProperty(name = "notificacion.proveedor", havingValue = "sms")
public class SMSNotificacionService implements NotificacionService {
    public void enviar(String destino, String mensaje) { /* ... */ }
}
```

El Servicio que usa `NotificacionService` queda cerrado a modificacion: no necesita cambiar cuando se anade un nuevo canal de notificacion.

## Liskov (implementaciones intercambiables)

El principio de sustitucion de Liskov (LSP) establece que los subtipos deben ser sustituibles por sus tipos base sin alterar la correccion del programa. En la practica, esto significa que cualquier implementacion de una interfaz debe comportarse de acuerdo a lo que la interfaz promete.

Dentro de la arquitectura por capas, LSP es crucial para que el mecanismo de inyeccion de dependencias funcione correctamente. Si `UsuarioService` depende de `UsuarioRepository`, cualquier implementacion concreta (`JpaUsuarioRepository`, `MongoUsuarioRepository`, `MockUsuarioRepository`) debe poder reemplazar a la otra sin que el servicio falle o se comporte de manera sorprendente.

Una violacion tipica ocurre cuando una implementacion lanza excepciones que la interfaz no declara o cuando tiene efectos secundarios que el cliente no espera. Por ejemplo, si `JpaUsuarioRepository.save()` devuelve la entidad con el ID generado pero `MockUsuarioRepository.save()` devuelve `null`, la sustitucion no es segura: el servicio dejara de funcionar correctamente con el mock.

> Para cumplir LSP, las precondiciones de las implementaciones no pueden ser mas restrictivas que las de la interfaz, y las poscondiciones no pueden ser mas debiles. Una implementacion que exige que el argumento no sea nulo cuando la interfaz no lo exige viola LSP.

## Interface Segregation (interfaces pequenas y especificas)

El principio de segregacion de interfaces (ISP) sostiene que ningun cliente debe ser forzado a depender de metodos que no usa. En lugar de una interfaz grande y omnicomprensiva, es preferible tener varias interfaces pequenas y cohesionadas.

En la arquitectura por capas, esto se traduce en interfaces de repositorio y servicio que exponen exactamente las operaciones que sus clientes necesitan. No tiene sentido que un servicio de consulta dependa de una interfaz de repositorio que incluye metodos de escritura si el servicio solo lee datos.

```java
// Mal: interfaz muy grande
public interface UsuarioRepository {
    Optional<Usuario> findById(Long id);
    List<Usuario> findAll();
    Usuario save(Usuario usuario);
    void deleteById(Long id);
    boolean existsByEmail(String email);
    long count();
}

// Bien: segregada por operacion
public interface UsuarioReadRepository {
    Optional<Usuario> findById(Long id);
    List<Usuario> findAll();
}

public interface UsuarioWriteRepository {
    Usuario save(Usuario usuario);
    void deleteById(Long id);
}
```

Un servicio de solo lectura depende unicamente de `UsuarioReadRepository`. Un servicio administrativo que necesita escribir puede depender de ambas. Esto evita acoplar clientes de solo lectura a metodos de escritura que nunca invocan.

## Dependency Inversion (depender de abstracciones)

El principio de inversion de dependencias (DIP) tiene dos caras: los modulos de alto nivel no deben depender de modulos de bajo nivel; ambos deben depender de abstracciones. Las abstracciones no deben depender de los detalles; los detalles deben depender de las abstracciones.

Este principio es el que da nombre a la inversion de control (IoC) que implementan contenedores como Spring. Sin DIP, la capa de negocio importaria directamente clases concretas de la capa de persistencia, creando un acoplamiento rigido. Con DIP, la capa de negocio define la interfaz que necesita (la abstraccion) y la capa de persistencia la implementa.

```java
// La capa de negocio define la abstraccion
public interface UsuarioRepository {
    Optional<Usuario> findById(Long id);
}

// La capa de persistencia implementa la abstraccion
@Repository
public class JpaUsuarioRepository implements UsuarioRepository {
    // ...
}

// La capa de negocio solo conoce la abstraccion
@Service
public class UsuarioService {
    private final UsuarioRepository repository; // abstraccion, no implementacion
    
    public UsuarioService(UsuarioRepository repository) {
        this.repository = repository;
    }
}
```

La direccion de las dependencias se invierte: el modulo de alto nivel (servicio) no importa nada de bajo nivel; define un contrato que el modulo de bajo nivel debe cumplir. La implementacion concreta se inyecta en tiempo de ejecucion, lo que permite cambiar de estrategia de persistencia sin tocar una sola linea de la capa de negocio.
