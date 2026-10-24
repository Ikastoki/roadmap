# Arquitectura Hexagonal (Puertos y Adaptadores)

En las arquitecturas de capas tradicionales, la direccion de las dependencias sigue el flujo de la peticion: el controlador conoce al servicio, y el servicio conoce al repositorio. Esto genera un acoplamiento inevitable hacia los detalles tecnicos --la base de datos, el framework web, el cliente HTTP-- que terminan contaminando el nucleo de la aplicacion. La arquitectura hexagonal, propuesta por Alistair Cockburn en 2005, invierte esta situacion: el dominio ocupa el centro del sistema y no depende de nada externo; todo lo externo depende de el. Tambien conocida como patron Puertos y Adaptadores (Ports and Adapters), esta arquitectura plantea que la aplicacion interactua con el exterior a traves de puertos (interfaces) cuyas implementaciones --los adaptadores-- son intercambiables sin afectar al nucleo.

## Puertos de entrada y de salida

Un puerto es una interfaz que define un contrato, y existen dos categorias. Los puertos de entrada (inbound ports) expresan lo que la aplicacion puede hacer: casos de uso, operaciones que el mundo exterior puede invocar. Los puertos de salida (outbound ports) expresan lo que la aplicacion necesita del exterior: persistencia, envio de notificaciones, llamadas a servicios externos. El dominio implementa los puertos de entrada y declara los puertos de salida; nunca conoce a los adaptadores.

```java
// Puerto de entrada: caso de uso
public interface CreateUserUseCase {
    UserDTO execute(CreateUserRequest request);
}

// Puerto de salida: repositorio
public interface UserRepository {
    User save(User user);
    Optional<User> findById(Long id);
}
```

## Adaptadores

Un adaptador es una implementacion concreta de un puerto. Los adaptadores de entrada convierten las peticiones del mundo exterior (HTTP, CLI, colas de mensajes) en llamadas a los puertos de entrada. Los adaptadores de salida implementan los puertos de salida utilizando herramientas concretas: JPA para persistencia, SMTP para email, REST client para servicios externos. El nucleo de la aplicacion no importa ni conoce ninguna de estas implementaciones.

```java
// Adaptador de entrada: REST controller
@RestController
public class UserController {
    private final CreateUserUseCase createUser;

    public UserController(CreateUserUseCase createUser) {
        this.createUser = createUser;
    }

    @PostMapping("/users")
    public ResponseEntity<UserDTO> createUser(@RequestBody CreateUserRequest request) {
        UserDTO result = createUser.execute(request);
        return ResponseEntity.status(201).body(result);
    }
}

// Adaptador de salida: repositorio JPA
@Repository
public class JpaUserRepository implements UserRepository {
    private final JpaUserEntityRepository jpaRepository;

    public JpaUserRepository(JpaUserEntityRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public User save(User user) {
        UserEntity entity = UserEntity.from(user);
        UserEntity saved = jpaRepository.save(entity);
        return saved.toDomain();
    }

    @Override
    public Optional<User> findById(Long id) {
        return jpaRepository.findById(id).map(UserEntity::toDomain);
    }
}
```

> La separacion entre puerto y adaptador permite, por ejemplo, tener un adaptador de persistencia en memoria para tests y uno con PostgreSQL para produccion, sin modificar una sola linea del dominio.

## Dependencias hacia el centro

La regla fundamental de la arquitectura hexagonal es que las dependencias apuntan hacia el centro. El dominio no conoce los controladores, las bases de datos ni los frameworks. Esta inversion de dependencias se materializa mediante la inyeccion de dependencias en el punto de composicion --tipicamente el `main` o un contenedor IoC-- que ensambla los adaptadores y los inyecta en los puertos.

```
[Controller] --> [CreateUserUseCase] <-- [UserService]
                                                |
                                                v
[UserRepository] <-- [JpaUserRepository] --> [JPA]
```

Las flechas indican la direccion de la dependencia en tiempo de compilacion: `UserService` implementa `CreateUserUseCase` y depende de `UserRepository`; `JpaUserRepository` implementa `UserRepository` y depende de JPA. El dominio (`UserService`, `User`, `UserRepository`) no importa nada de infraestructura.

## Beneficios

El aislamiento del dominio produce tres ventajas practicas inmediatas. La testabilidad: los casos de uso se prueban sin levantar la base de datos ni el servidor HTTP, simplemente proporcionando un adaptador de salida mockeado. La intercambiabilidad: cambiar de base de datos o de framework web implica escribir un nuevo adaptador sin tocar el nucleo. La independencia de frameworks: el dominio permanece puro Java, sin anotaciones JPA, sin dependencias de Spring, sin acoplamiento a ninguna tecnologia concreta.

> La arquitectura hexagonal no prescribe cuantas capas debe tener el interior del hexagono. El dominio puede organizarse internamente en servicios, entidades, value objects y repositorios; lo relevante es que nada de lo que esta dentro depende de lo que esta fuera.
