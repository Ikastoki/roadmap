# Tecnologias de persistencia en Java

El ecosistema Java ofrece un conjunto de tecnologias de persistencia que cubren desde el acceso directo a la base de datos hasta abstracciones completas del modelo relacional. La eleccion entre una u otra determina el nivel de control sobre el SQL generado, la productividad del equipo, el rendimiento de las consultas y la portabilidad entre distintos motores de base de datos.

## JDBC

Java Database Connectivity (JDBC) es la capa de acceso a datos mas baja del ecosistema Java, parte del JDK desde su version 1.1. Proporciona una API estandar para conectar con cualquier base de datos relacional, enviar sentencias SQL y procesar los resultados. El programador escribe SQL directamente, gestiona manualmente las conexiones mediante `DriverManager` o `DataSource`, y recorre los `ResultSet` fila a fila. No hay ORM, no hay caché de primer nivel, no hay generacion automatica de consultas. El control total tiene un coste: el codigo JDBC es verbose, propenso a errores de cierre de recursos y requiere conocimiento profundo de SQL y del esquema de la base de datos.

```java
Connection conn = DriverManager.getConnection(url, user, pass);
PreparedStatement ps = conn.prepareStatement("SELECT * FROM usuarios WHERE id = ?");
ps.setInt(1, 42);
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    String nombre = rs.getString("nombre");
}
rs.close(); ps.close(); conn.close();
```

## JPA (Jakarta Persistence API)

JPA es un estandar (especificacion) dentro de Jakarta EE (antes Java EE) que define un modelo de mapeo objeto-relacional (ORM). No es una implementacion, sino un conjunto de interfaces y anotaciones en el paquete `jakarta.persistence`. La especificacion cubre la asignacion de clases Java a tablas relacionales, el ciclo de vida de las entidades (managed, detached, removed), el lenguaje de consultas JPQL, el mecanismo de transacciones (EntityTransaction) y la generacion de esquemas. Al ser una especificacion, cualquier implementacion conforme (Hibernate, EclipseLink, OpenJPA) puede usarse de forma intercambiable.

```java
@Entity
@Table(name = "usuarios")
public class Usuario {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
}
```

## Hibernate

Hibernate es la implementacion JPA mas extendida y madura del ecosistema Java. Nacio en 2001 como ORM independiente y posteriormente adopto el estandar JPA, aunque conserva su API nativa (Session, Criteria, HQL). Hibernate anade funcionalidades que la especificacion JPA no cubre: caché de segundo nivel (con proveedores como EHCache o Redis), mapeos mas flexibles (Formula, Any, Join), batch insercion optimizada, y un optimizador de consultas (QueryPlanCache). La mayoria de proyectos que usan JPA usan Hibernate como motor subyacente, a menudo sin ser conscientes de ello.

## MyBatis

MyBatis no es un ORM en sentido estricto. Proporciona un mapeo manual entre resultados SQL y objetos Java, pero el programador escribe las consultas SQL explicitamente en ficheros XML o anotaciones. MyBatis resuelve el problema de la impedancia objeto-relacional sin ocultar el SQL: el desarrollador conserva control total sobre cada consulta, lo que resulta util para esquemas heredados, consultas enormemente optimizadas o bases de datos no normalizadas. Ofrece mapeo automatico de resultados (ResultMap), gestion de relaciones (association, collection) y paginacion, pero no genera SQL ni gestiona automaticamente el estado de las entidades.

```xml
<select id="findById" resultType="Usuario">
    SELECT * FROM usuarios WHERE id = #{id}
</select>
```

## Spring Data

Spring Data es un proyecto del ecosistema Spring que anade una capa de abstraccion sobre JPA, MongoDB, Redis, Cassandra y otros motores. Para JPA, Spring Data JPA elimina la necesidad de escribir la implementacion del DAO: basta con definir una interfaz que extienda `JpaRepository` y declarar metodos cuya firma siga una convencion de nombres (`findByNombre`, `findByApellidoAndEdadGreaterThan`). El framework genera automaticamente la implementacion y las consultas JPQL. Spring Data no sustituye a JPA o Hibernate: se situa por encima, simplificando el uso.

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    List<Usuario> findByNombre(String nombre);
}
```

## Comparativa

| Aspecto | JDBC | JPA | Hibernate | MyBatis | Spring Data |
|---------|------|-----|-----------|---------|-------------|
| Control SQL | Total | Bajo | Bajo/Medio | Total | Bajo |
| Productividad | Baja | Alta | Alta | Media | Muy alta |
| Curva aprendizaje | Baja | Alta | Alta | Media | Media |
| Cache | No | 1er nivel | 1er y 2o nivel | 2o nivel opcional | Hereda JPA |
| Portabilidad BD | Alta | Alta | Alta | Media | Alta |
| Rendimiento optimo | Manual | Automatico | Configurable | Manual | Automatico |
| Codigo verboso | Mucho | Poco | Poco | Moderado | Minimo |

JDBC sigue siendo la opcion correcta cuando se requiere control absoluto sobre las consultas, cuando se trabaja con procedimientos almacenados complejos o cuando la capa de abstraccion de un ORM introduciria una sobrecarga inaceptable. JPA/Hibernate brillan en aplicaciones con modelo de dominio rico, donde el coste de desarrollo y mantenimiento del mapeo manual supera al del ORM. MyBatis ocupa un punto intermedio: control SQL sin renunciar al mapeo automatico de resultados. Spring Data simplifica aun mas cuando se usa con Spring Boot y no se necesita personalizar las consultas mas alla de lo que ofrecen los metodos derivados del nombre.
