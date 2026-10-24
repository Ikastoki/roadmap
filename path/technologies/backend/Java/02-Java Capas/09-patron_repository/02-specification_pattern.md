# Specification pattern

Los sistemas de busqueda rara vez responden a consultas fijas. Un listado de usuarios puede filtrarse por nombre, por estado de actividad, por rango de edad o por cualquier combinacion de estos criterios, y esa combinacion suele definirse en tiempo de ejecucion segun la entrada del usuario. El patron Specification aborda este problema encapsulando cada criterio de busqueda en un objeto independiente, combinable mediante operadores logicos AND, OR y NOT. Cada especificacion es un predicado que pregunta si un candidato dado la satisface, y la composicion de especificaciones permite construir consultas complejas sin reescribir metodos de repositorio para cada combinacion posible.

## Interfaz Specification

La interfaz `Specification<T>` define el contrato que toda especificacion debe cumplir. `T` es el tipo de la entidad sobre la que se evalua el predicado:

```java
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }
}
```

El metodo `isSatisfiedBy` es la unica operacion abstracta. Recibe una entidad y devuelve `true` si la entidad cumple la condicion que la especificacion representa. Los metodos `and`, `or` y `not` son metodos por defecto que devuelven nuevas especificaciones compuestas, aplicando las reglas de la logica booleana.

> La implementacion de `and`, `or` y `not` como metodos por defecto en la interfaz (Java 8+) permite que cualquier clase que implemente `Specification` obtenga automaticamente la capacidad de combinarse sin heredar de una clase base ni implementar metodos adicionales.

Para recuperar la entidad directamente desde una especificacion sin tener que filtrar en memoria, el repositorio acepta una `Specification` como criterio de busqueda:

```java
public interface SpecificationRepository<T, ID> extends GenericRepository<T, ID> {
    List<T> findAll(Specification<T> specification);
    Optional<T> findOne(Specification<T> specification);
    long count(Specification<T> specification);
}
```

Esta variante del repositorio ejecuta la especificacion contra la base de datos, traduciendo el predicado a una clausula WHERE en la consulta subyacente.

## Combinacion de condiciones

La verdadera potencia del patron emerge al combinar especificaciones atomicas para formar predicados complejos. Cada combinacion devuelve una nueva especificacion, lo que permite un estilo declarativo y fluido:

```java
Specification<User> nameSpec = user -> user.getName().equalsIgnoreCase("Juan");
Specification<User> activeSpec = user -> user.isActive();

Specification<User> activeUsersNamedJuan = nameSpec.and(activeSpec);
Specification<User> inactiveUsers = activeSpec.not();
Specification<User> namedJuanOrInactive = nameSpec.or(inactiveUsers);
```

La composicion puede anidarse arbitrariamente. Una busqueda como "usuarios activos llamados Juan, o usuarios inactivos mayores de 30 anos" se construye combinando cuatro especificaciones atomicas:

```java
Specification<User> nameSpec = byName("Juan");
Specification<User> activeSpec = active();
Specification<User> ageSpec = byAgeGreaterThan(30);
Specification<User> inactiveSpec = activeSpec.not();

Specification<User> complexSearch = nameSpec.and(activeSpec).or(inactiveSpec.and(ageSpec));
```

Cada nivel de composicion introduce un nuevo nivel de parentesis implicito. La expresion anterior equivale a `(name = 'Juan' AND active = true) OR (active = false AND age > 30)`.

Para la implementacion en memoria, la evaluacion es directa: se itera sobre la coleccion y se filtra aplicando `isSatisfiedBy`. Para la implementacion contra base de datos, cada especificacion debe traducirse a una expresion del lenguaje de consulta subyacente. En el contexto de JPA, Spring Data JPA proporciona `Specification<T>` (paquete `org.springframework.data.jpa.domain`) que integra directamente con `JpaSpecificationExecutor`:

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> {
}
```

Este enfoque permite:

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public List<User> search(String name, Boolean active, Integer minAge) {
        Specification<User> spec = Specification.where(null);

        if (name != null) {
            spec = spec.and((root, query, cb) ->
                cb.equal(root.get("name"), name));
        }
        if (active != null) {
            spec = spec.and((root, query, cb) ->
                cb.equal(root.get("active"), active));
        }
        if (minAge != null) {
            spec = spec.and((root, query, cb) ->
                cb.greaterThan(root.get("age"), minAge));
        }

        return userRepository.findAll(spec);
    }
}
```

Spring Data JPA traduce automaticamente cada `Specification` a una clausula `WHERE` mediante el uso del API de Criteria de JPA. Cada lambda recibe `root`, `query` y `cb`, que representan respectivamente la entidad raiz, la consulta en construccion y el builder de criterios.

## Especificaciones comunes

Ciertos patrones de busqueda se repiten con frecuencia en todas las aplicaciones. Agrupar estas especificaciones en una clase static factory permite reutilizarlas sin replicar codigo:

```java
public final class UserSpecifications {

    public static Specification<User> byName(String name) {
        return (root, query, cb) -> cb.equal(root.get("name"), name);
    }

    public static Specification<User> byNameContaining(String partial) {
        return (root, query, cb) -> cb.like(root.get("name"), "%" + partial + "%");
    }

    public static Specification<User> active() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<User> byAgeGreaterThan(int age) {
        return (root, query, cb) -> cb.greaterThan(root.get("age"), age);
    }

    public static Specification<User> byAgeBetween(int min, int max) {
        return (root, query, cb) -> cb.between(root.get("age"), min, max);
    }

    public static Specification<User> byEmailDomain(String domain) {
        return (root, query, cb) ->
            cb.like(root.get("email"), "%@" + domain);
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> cb.greaterThan(root.get("createdAt"), date);
    }
}
```

> La factoria de especificaciones actua como un "lenguaje de dominio" para consultas. Los nombres de los metodos ( `byName`, `active`, `byAgeGreaterThan`) expresan intencion de negocio, no detalles tecnicos de persistencia. El servicio que las usa no sabe si la busqueda se ejecuta via SQL, JPQL o Criteria API.

El uso combinado en el servicio resulta expresivo y legible:

```java
public List<User> searchUsers(String name, Boolean active, Integer minAge) {
    Specification<User> spec = Specification.where(null);

    if (name != null) {
        spec = spec.and(UserSpecifications.byNameContaining(name));
    }
    if (active != null) {
        spec = active ? spec.and(UserSpecifications.active())
                      : spec.and(UserSpecifications.active().not());
    }
    if (minAge != null) {
        spec = spec.and(UserSpecifications.byAgeGreaterThan(minAge));
    }

    return userRepository.findAll(spec);
}
```

Para implementaciones en memoria (sin JPA), las especificaciones se definen con funciones lambda sobre el objeto directamente:

```java
public final class InMemoryUserSpecifications {

    public static Specification<User> byName(String name) {
        return user -> user.getName().equalsIgnoreCase(name);
    }

    public static Specification<User> active() {
        return User::isActive;
    }

    public static Specification<User> byAgeGreaterThan(int age) {
        return user -> user.getAge() > age;
    }
}
```

La diferencia fundamental es que la version JPA opera sobre el metamodelo de la entidad (`root.get("name")`) mientras que la version en memoria opera sobre las propiedades del objeto Java (`user.getName()`). El resto del codigo --la composicion, la combinacion y el consumo desde el servicio-- permanece identico.

> El patron Specification no esta limitado a busquedas. Tambien resulta util para validaciones de negocio que dependen de multiples condiciones, reglas de autorizacion que evaluan el contexto del usuario o cualquier escenario donde una condicion logica deba encapsularse, combinarse y reutilizarse.
