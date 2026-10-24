# Arquitectura Onion (Cebolla)

Jeffrey Palermo presento en 2008 la arquitectura Onion como una respuesta a los problemas de acoplamiento en las arquitecturas de capas tradicionales. Donde la arquitectura de capas clasica coloca la base de datos en la base de la piramide --haciendo que todo dependa de ella--, la Onion invierte la jerarquia y situa el modelo de dominio en el centro, rodeado por anillos concentricos de servicios e infraestructura. Cada anillo solo puede depender de los anillos mas internos, nunca de los externos, lo que garantiza que el nucleo del negocio permanezca aislado de los detalles tecnicos.

## Estructura de capas concentricas

La Onion se organiza en cuatro anillos principales, aunque el numero puede variar segun las necesidades del proyecto. Desde el centro hacia fuera:

**Domain Model:** entidades y value objects que representan el negocio. Contienen estado y comportamiento, sin dependencias externas. Son objetos Java planos (POJOs) sin anotaciones de frameworks.

**Domain Services:** logica de negocio pura que no encaja naturalmente en una entidad. Operan sobre el modelo de dominio y definen interfaces para los repositorios que necesitan, pero no conocen sus implementaciones concretas.

**Application Services:** orquestan los casos de uso de la aplicacion. Coordinan domain services, repositorios y otros servicios de infraestructura, siempre a traves de interfaces. Son responsables de las transacciones, la seguridad y la coordinacion de flujos.

**Infrastructure:** contiene las implementaciones concretas de las interfaces definidas en los anillos internos: repositorios JPA, clientes HTTP, servicios de email, acceso a sistemas de archivos.

**Presentation:** la capa mas externa, que maneja la interaccion con el usuario: controladores REST, vistas, serializacion JSON, validacion de entrada.

```
          [Presentation]
          [Infrastructure]
          [Application Services]
          [Domain Services]
          [Domain Model]   <-- centro
```

## Relaciones entre anillos

Cada anillo solo conoce a los anillos que estan dentro de el. El Domain Model no conoce absolutamente nada fuera de si mismo. Los Domain Services conocen el Domain Model pero no los Application Services. Los Application Services conocen los Domain Services y las interfaces de repositorio (definidas en el dominio), pero no las implementaciones de infraestructura. Infrastructure y Presentation son los anillos mas externos y pueden depender de cualquier anillo interno.

```java
// Domain Model (anillo 1)
public class User {
    private UserId id;
    private Email email;
    private Name name;

    public void changeEmail(Email newEmail) {
        this.email = newEmail;
    }
}

// Domain Service (anillo 2)
public class UserRegistrationService {
    private final UserRepository userRepository;

    public UserRegistrationService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User register(Email email, Name name) {
        if (userRepository.existsByEmail(email)) {
            throw new EmailAlreadyExistsException(email);
        }
        return new User(UserId.generate(), email, name);
    }
}

// Application Service (anillo 3)
public class CreateUserApplicationService {
    private final UserRegistrationService registrationService;
    private final UserRepository userRepository;
    private final UnitOfWork unitOfWork;

    public CreateUserApplicationService(
            UserRegistrationService registrationService,
            UserRepository userRepository,
            UnitOfWork unitOfWork) {
        this.registrationService = registrationService;
        this.userRepository = userRepository;
        this.unitOfWork = unitOfWork;
    }

    public UserDTO execute(CreateUserRequest request) {
        Email email = new Email(request.email());
        Name name = new Name(request.name());
        User user = registrationService.register(email, name);
        userRepository.save(user);
        unitOfWork.commit();
        return UserDTO.from(user);
    }
}

// Infrastructure (anillo 4)
@Repository
public class JpaUserRepository implements UserRepository {
    @PersistenceContext
    private EntityManager em;

    @Override
    public void save(User user) {
        UserEntity entity = UserEntity.from(user);
        em.persist(entity);
    }
}
```

## Testabilidad y sustitucion

Cada anillo puede testearse de forma independiente. El Domain Model se prueba sin ningun mock: solo instancias de entidades y value objects. Los Domain Services se prueban con mocks de las interfaces de repositorio, sin necesidad de base de datos. Los Application Services se prueban combinando mocks de Domain Services y repositorios. Los adaptadores de infraestructura se prueban con tests de integracion contra la base de datos real o embebida.

> La Onion comparte la filosofia de la hexagonal --dominio en el centro, dependencias hacia dentro-- pero enfatiza la estructura de capas concentricas como guia de organizacion del codigo, mientras que la hexagonal se centra en el concepto de puertos y adaptadores como mecanismo de desacoplamiento.

## Relacion con la arquitectura Hexagonal

En la practica, la Onion y la hexagonal son variantes del mismo principio. La Onion explicita la organizacion interna del hexagono dividiendolo en anillos (domain model, domain services, application services), mientras que la hexagonal trata el interior como una caja que puede estructurarse libremente. La Onion suele considerarse un paso intermedio entre la arquitectura de capas clasica y la hexagonal, ya que proporciona mas estructura sin llegar al nivel de detalle de la Clean Architecture.
