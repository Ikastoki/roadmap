# Que es JPA

Si ORM es el concepto general, JPA (Jakarta Persistence API, antes Java Persistence API) es la estandarizacion de ese concepto para el ecosistema Java. Antes de JPA, cada framework ORM definia su propia API, sus propias anotaciones y su propia configuracion. Migrar de Hibernate a TopLink significaba reescribir toda la capa de persistencia. JPA nacio en 2006 como JSR 220 dentro de Java EE 5 con el objetivo de unificar el mercado bajo una especificacion comun, permitiendo que distintas implementaciones compitieran ofreciendo la misma interfaz publica.

## Especificacion, no implementacion

JPA es una especificacion, no un producto instalable. Define un conjunto de interfaces (`EntityManager`, `EntityTransaction`, `Query`, `TypedQuery`, `CriteriaBuilder`, etc.), anotaciones (`@Entity`, `@Id`, `@OneToMany`, etc.) y un lenguaje de consultas (JPQL) que cualquier proveedor ORM puede implementar. Es decir, JPA establece el contrato, y cada proveedor ofrece la implementacion subyacente.

Esto implica que el codigo escrito contra la API de JPA funciona con cualquier proveedor compatible. Si se decide migrar de Hibernate a EclipseLink, los cambios se limitan al archivo de configuracion `persistence.xml` y, en el peor de los casos, a ajustes menores en propiedades especificas del proveedor. El codigo fuente que usa `EntityManager`, `@Entity` y JPQL permanece intacto.

```java
// Este codigo funciona con Hibernate, EclipseLink u OpenJPA
@Stateless
public class UsuarioService {
    @PersistenceContext
    private EntityManager em;

    public Usuario buscarPorId(Long id) {
        return em.find(Usuario.class, id);
    }
}
```

> La palabra clave es "Jakarta". Desde 2019, con la transferencia de Java EE a la Eclipse Foundation, la especificacion paso a llamarse Jakarta Persistence (javax.persistence paso a jakarta.persistence). Las versiones modernas (JPA 3.x) usan el nuevo espacio de nombres, aunque el API es esencialmente el mismo.

## Anotaciones principales

JPA define un conjunto de anotaciones en el paquete `jakarta.persistence` que configuran el mapeo objeto-relacional de forma declarativa. Las fundamentales son:

| Anotacion | Proposito |
|---|---|
| `@Entity` | Marca una clase como entidad persistible |
| `@Table` | Especifica la tabla correspondiente en BD |
| `@Id` | Marca la clave primaria |
| `@Column` | Configura la columna asociada a un atributo |
| `@GeneratedValue` | Estrategia de generacion de IDs |
| `@Basic` | Indica que es un atributo persistente simple |
| `@Transient` | Excluye un atributo de la persistencia |
| `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany` | Relaciones |
| `@JoinColumn`, `@JoinTable` | Configuracion de claves ajenas |

```java
@Entity
@Table(name = "empleados")
public class Empleado {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nombre_completo", nullable = false, length = 200)
    private String nombre;

    @Column(unique = true)
    private String email;

    @Transient
    private boolean datosSensiblesCargados;
}
```

## Proveedores JPA

Cualquier implementacion que cumpla con la especificacion JPA puede actuar como proveedor. Los tres mas relevantes son:

**Hibernate** es el ORM mas maduro y utilizado del ecosistema Java. Su implementacion de JPA es completa y ofrece numerosas extensiones propietarias (como `@Filter`, `@DynamicUpdate`, `hibernate.envers` para auditoria). Es la opcion por defecto en Spring Boot y en la mayoria de proyectos empresariales. Su origen como framework independiente (anterior a JPA) le otorga un conjunto de funcionalidades que van mas alla de la especificacion.

**EclipseLink** es el proveedor de referencia de Jakarta EE. Nacio como TopLink y fue donado por Oracle a la Eclipse Foundation. Ofrece caracteristicas avanzadas como soporte nativo para JAXB (MOXy) y para bases de datos NoSQL. Es la opcion preferida en proyectos que corren sobre servidores de aplicaciones Jakarta EE (Payara, WildFly) por su integracion nativa.

**OpenJPA** fue el proveedor de referencia de Apache. Su desarrollo se ha ralentizado en los ultimos anos, aunque sigue siendo funcional. Su principal contribucion fue ser la implementacion de referencia de las primeras versiones de JPA.

```xml
<!-- persistence.xml: seleccion del proveedor -->
<persistence-unit name="miUnidad">
    <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
    <!-- ... -->
</persistence-unit>
```

## Java SE vs Jakarta EE

JPA puede utilizarse tanto en entornos Jakarta EE (ahora dentro de servidores de aplicaciones) como en aplicaciones Java SE de escritorio o consola. La diferencia principal radica en como se gestiona el ciclo de vida del `EntityManagerFactory` y del `EntityManager`.

En **Jakarta EE**, el contenedor gestiona el `EntityManagerFactory` y la transaccion. La inyeccion via `@PersistenceContext` delega en el contenedor la creacion del EntityManager, la sincronizacion con la transaccion JTA y el cierre de recursos. El desarrollador solo se preocupa de la logica de negocio.

```java
// Jakarta EE: gestion por contenedor
@Stateless
public class PedidoService {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void crearPedido(Pedido pedido) {
        em.persist(pedido);
    }
}
```

En **Java SE**, el desarrollador debe gestionar manualmente el `EntityManagerFactory` (crearlo, mantenerlo y cerrarlo al finalizar), crear `EntityManager` para cada unidad de trabajo, gestionar la transaccion mediante `em.getTransaction().begin()/commit()/rollback()` y cerrar el EntityManager al terminar.

```java
// Java SE: gestion manual
EntityManagerFactory emf = Persistence.createEntityManagerFactory("miUnidad");
EntityManager em = emf.createEntityManager();
try {
    em.getTransaction().begin();
    Pedido pedido = new Pedido();
    em.persist(pedido);
    em.getTransaction().commit();
} catch (Exception e) {
    em.getTransaction().rollback();
    throw e;
} finally {
    em.close();
}
emf.close();
```

Spring Boot ocupa un punto intermedio: no es Jakarta EE, pero ofrece gestion automatica del `EntityManagerFactory` (como bean de Spring) y transacciones declarativas mediante `@Transactional`. Es el escenario mas comun en proyectos modernos.
