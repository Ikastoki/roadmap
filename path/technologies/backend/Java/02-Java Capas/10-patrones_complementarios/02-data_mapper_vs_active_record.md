# Data Mapper vs Active Record

La comunicacion entre el modelo de objetos de una aplicacion y el modelo relacional de la base de datos requiere un mecanismo de traduccion. Dos patrones compiten para resolver este desajuste de impedancia: Active Record, donde la propia entidad conoce y gestiona su persistencia, y Data Mapper, donde una capa intermedia separa por completo el dominio del esquema fisico. La eleccion entre uno y otro determina el grado de acoplamiento del codigo de negocio con la infraestructura subyacente.

## Active Record

En Active Record, cada objeto del dominio es al mismo tiempo una fila de la base de datos. La clase expone metodos como `save()`, `delete()` y `findById()` directamente sobre la instancia o la clase. No existe separacion entre el modelo de negocio y el mecanismo de persistencia: la entidad sabe como guardarse a si misma.

```java
public class Usuario {
    private Long id;
    private String email;
    private String nombre;

    // Metodos de persistencia embebidos
    public void save() { /* INSERT INTO usuario ... */ }
    public void delete() { /* DELETE FROM usuario WHERE id = ... */ }
    public static Usuario findById(Long id) { /* SELECT ... */ }
}
```

Este patron es la base de frameworks como Ruby on Rails (ActiveRecord) y Django (ORM). Su principal ventaja es la simplicidad: para casos de uso sencillos donde la logica de negocio es escasa y cada entidad se corresponde directamente con una tabla, Active Record reduce la cantidad de codigo infraestructural y acelera el desarrollo.

Sin embargo, el acoplamiento es problematico en sistemas complejos. La entidad de dominio depende del esquema de base de datos, de la conexion JDBC y de las convenciones del ORM. Cualquier cambio en la tabla --renombrar una columna, modificar un tipo, cambiar la estrategia de herencia-- repercute directamente en la clase de dominio. Ademas, las responsabilidades se mezclan: una entidad que debe aplicar reglas de negocio, validar su estado y calcular valores colaterales tambien debe ocuparse de SQL, conexiones y transacciones.

En el ecosistema Java empresarial, Active Record no es la practica recomendada. JPA se diseno en torno al concepto de entidad gestionada por un EntityManager, no de entidad autonoma. Aunque es posible emular Active Record con JPA --por ejemplo, haciendo que la entidad obtenga un EntityManager estatico via CDI o un singleton--, esta practica rompe el modelo de contexto de persistencia y dificulta las pruebas unitarias.

## Data Mapper

El patrón Data Mapper introduce una capa intermedia --el mapper-- que transfiere datos entre el objeto de dominio y la base de datos sin que ninguno de los dos conozca al otro. Las entidades de dominio son POJOs completamente ignorantes de la persistencia: no extienden ninguna clase base, no implementan interfaces del ORM y no contienen metodos `save()` o `find()`.

```java
public class Usuario {
    private Long id;
    private String email;
    private String nombre;

    // Solo logica de negocio, cero infraestructura
    public void cambiarEmail(String nuevoEmail) {
        if (nuevoEmail == null || !nuevoEmail.contains("@")) {
            throw new IllegalArgumentException("Email invalido");
        }
        this.email = nuevoEmail;
    }
}
```

El mapper se implementa en repositorios que utilizan el EntityManager de JPA. El repositorio conoce la correspondencia entre el objeto de dominio y la tabla, pero la entidad no. El EntityManager actua como el bus de persistencia: las entidades se adjuntan al contexto (``EntityManager.persist()``), se buscan (``find()``), se fusionan (``merge()``) o se eliminan (``remove()``).

```java
@Repository
public class UsuarioRepository {

    @PersistenceContext
    private EntityManager em;

    public Usuario findById(Long id) {
        return em.find(Usuario.class, id);
    }

    public void save(Usuario usuario) {
        if (usuario.getId() == null) {
            em.persist(usuario);
        } else {
            em.merge(usuario);
        }
    }

    public void delete(Usuario usuario) {
        em.remove(em.contains(usuario) ? usuario : em.merge(usuario));
    }

    public List<Usuario> buscarPorEmail(String email) {
        return em.createQuery("SELECT u FROM Usuario u WHERE u.email = :email", Usuario.class)
                .setParameter("email", email)
                .getResultList();
    }
}
```

## Diferencias fundamentales

| Aspecto | Active Record | Data Mapper |
|---|---|---|
| Acoplamiento | La entidad conoce la BD | La entidad ignora la persistencia |
| Responsabilidad | Persistencia + negocio mezclados | Negocio en la entidad, persistencia en el repositorio |
| Test unitario | Requiere mockear conexion o BD | La entidad se prueba sin infraestructura |
| Cambio de esquema | Afecta directamente a la entidad | Solo afecta al mapper/repositorio |
| Complejidad | Menor inicial, mayor al escalar | Mayor inicial, mas mantenible al escalar |
| Lenguaje tipico | Ruby, Python, PHP | Java, C# |

## Por que JPA es Data Mapper

JPA adopta el patron Data Mapper mediante el EntityManager. Las entidades se anotan con ``@Entity`` y ``@Table``, pero eso no las convierte en Active Record: la anotacion es metadata que el proveedor JPA (Hibernate, EclipseLink) lee para configurar el mapper, no codigo ejecutable de persistencia. La entidad sigue siendo un POJO que desconoce como y cuando se sincroniza con la base de datos.

El EntityManager implementa el identity map, el unit of work y el dirty checking por detras. Cuando se modifica una entidad gestionada, no es necesario llamar a un metodo `save()` explicito: el EntityManager detecta los cambios al hacer flush y genera automaticamente las sentencias UPDATE. Esta transparencia es posible precisamente porque el mapper (el EntityManager) mantiene el control total del ciclo de vida de la entidad.

> En Active Record, la llamada a `save()` es explicita y la entidad genera su propio SQL. En Data Mapper con JPA, la entidad se modifica como un objeto cualquiera y el EntityManager sincroniza los cambios de forma transparente al finalizar la transaccion.

## Cuando usar cada uno

Active Record tiene sentido en aplicaciones pequenas, prototipos, frameworks de productividad (Rails, Django, Laravel) o dominios donde la logica de negocio es esencialmente CRUD y el esquema de base de datos es estable. Data Mapper es preferible en sistemas empresariales con modelos de dominio ricos, multiples fuentes de datos, estrategias de herencia complejas o equipos que priorizan la testabilidad y el bajo acoplamiento.

En Java empresarial, la practica aceptada es Data Mapper con JPA. Intentar forzar Active Record sobre JPA resulta en codigo que lucha contra el framework y pierde las ventajas del contexto de persistencia gestionado.
