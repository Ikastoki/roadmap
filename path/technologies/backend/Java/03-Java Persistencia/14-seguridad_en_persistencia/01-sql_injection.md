# SQL Injection

La inyeccion SQL es una vulnerabilidad de seguridad que ocurre cuando una aplicacion construye sentencias SQL concatenando datos proporcionados por el usuario sin la debida validacion o parametrizacion. Un atacante puede manipular la consulta para leer, modificar o eliminar datos no autorizados, ejecutar comandos administrativos o incluso tomar control del servidor de base de datos. Es la amenaza mas antigua y a la vez mas frecuente en aplicaciones web, y su prevencion es responsabilidad directa de la capa de persistencia.

## Que es y como se produce

Cuando una aplicacion construye una consulta SQL mediante concatenacion de cadenas, el interprete SQL no distingue entre el codigo de la consulta y los datos del usuario. Si el usuario introduce un valor que contiene caracteres con significado sintactico (comillas simples, guiones para comentarios, punto y coma), puede alterar la estructura de la sentencia.

```java
// VULNERABLE: concatenacion directa
String sql = "SELECT * FROM usuarios WHERE email = '" + email + "' AND password = '" + password + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

Un atacante que envie `email = admin'--` y `password = cualquiercosa` convierte la consulta en:

```sql
SELECT * FROM usuarios WHERE email = 'admin'--' AND password = 'cualquiercosa'
```

El `--` comenta el resto de la condicion, devolviendo el usuario `admin` sin verificar la contrasena. Variantes mas peligrosas permiten usar `UNION` para extraer datos de otras tablas, `; DROP TABLE` para eliminar tablas, o funciones como `pg_sleep()` para realizar ataques de denial of service.

## Prevencion con PreparedStatement

La defensa fundamental contra SQL Injection es el uso de `PreparedStatement`. Con este mecanismo, la sentencia SQL se compila en el motor de base de datos con placeholders `?` y los valores se envian por separado, de modo que el motor los trata siempre como datos literales, nunca como codigo ejecutable.

```java
// SEGURO: parametrizacion con PreparedStatement
String sql = "SELECT * FROM usuarios WHERE email = ? AND password = ?";
PreparedStatement pstmt = connection.prepareStatement(sql);
pstmt.setString(1, email);
pstmt.setString(2, password);
ResultSet rs = pstmt.executeQuery();
```

La base de datos recibe la estructura de la consulta por un canal y los parametros por otro, haciendo imposible que el contenido de las variables altere la semantica de la sentencia. Incluso si el usuario introduce `admin' OR '1'='1`, el motor lo tratara como el valor literal del campo email, no como parte del SQL.

## JPA y Hibernate: consultas parametrizadas

JPA proporciona dos mecanismos de consulta que evitan la inyeccion SQL si se usan correctamente: JPQL con parametros nombrados y Criteria API. Ambos generan internamente `PreparedStatement`.

JPQL con parametros nombrados:

```java
TypedQuery<Usuario> query = entityManager.createQuery(
    "SELECT u FROM Usuario u WHERE u.email = :email AND u.password = :password",
    Usuario.class
);
query.setParameter("email", email);
query.setParameter("password", password);
List<Usuario> resultado = query.getResultList();
```

Spring Data JPA con `@Query`:

```java
@Query("SELECT u FROM Usuario u WHERE u.email = :email")
Optional<Usuario> findByEmail(@Param("email") String email);
```

> El error mas comun es usar `@Query` con concatenacion nativa. La anotacion soporta `nativeQuery = true` para SQL nativo, y en ese caso es igualmente obligatorio usar `:param` en lugar de concatenar con `+` o `String.format()`.

Criteria API, util para consultas dinamicas:

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Usuario> cq = cb.createQuery(Usuario.class);
Root<Usuario> root = cq.from(Usuario.class);
cq.select(root).where(
    cb.equal(root.get("email"), email),
    cb.equal(root.get("password"), password)
);
List<Usuario> resultado = entityManager.createQuery(cq).getResultList();
```

## Validacion de inputs

La parametrizacion no exime de validar los datos de entrada. La validacion actua como primera barrera y evita que datos malformados lleguen siquiera a la capa de persistencia. En una aplicacion Spring se usan las anotaciones `jakarta.validation`:

```java
public class UsuarioDTO {

    @NotBlank(message = "El email es obligatorio")
    @Email(message = "Formato de email invalido")
    private String email;

    @NotBlank
    @Size(min = 8, max = 100)
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).+$",
             message = "La contrasena debe contener mayusculas, minusculas y digitos")
    private String password;
}
```

Combinada con `@Valid` en el controlador:

```java
@PostMapping("/login")
public ResponseEntity<?> login(@Valid @RequestBody UsuarioDTO dto) { ... }
```

La validacion garantiza que los datos cumplen con el formato esperado antes de ser vinculados a parametros de consulta, reduciendo la superficie de ataque.

## ORM frente a SQL nativo

El uso de un ORM no protege automaticamente contra inyeccion SQL. La seguridad depende de como se construyan las consultas:

- **JPQL/HQL parametrizado**: seguro, siempre que no se concatenen valores.
- **Criteria API**: seguro, genera JPQL parametrizado internamente.
- **SQL nativo con `@Query(nativeQuery = true)`**: seguro solo si se usan parametros nombrados o posicionales.
- **Consultas con `EntityManager.createNativeQuery()`**: mismo requisito.
- **Named queries (`@NamedQuery`)**: seguras, se definen en tiempo de compilacion y no admiten concatenacion.

El riesgo aparece cuando se usa `EntityManager.createQuery()` concatenando cadenas, o cuando se emplea `@DynamicUpdate` con valores que provienen directamente del usuario sin intermediarios.

> Una regla practica infalible: si en la cadena de la consulta aparece el operador `+` o `String.format()` o `StringBuilder` para insertar un valor de usuario, la consulta es vulnerable.

```java
// VULNERABLE aunque use JPA
String jpql = "SELECT u FROM Usuario u WHERE u.email = '" + email + "'";
List<Usuario> resultado = entityManager.createQuery(jpql).getResultList();
```

La prevencion de SQL Injection no es una caracteristica opcional ni una buena practica: es un requisito de seguridad fundamental. Toda consulta, sin excepcion, debe utilizar parametros vinculados, tanto en JPQL como en SQL nativo, y los datos de entrada deben validarse en la capa de presentacion antes de alcanzar la logica de persistencia.
