# Anotaciones Hibernate

Ademas de las anotaciones estandar de JPA (`@Entity`, `@Table`, `@Column`, `@Id`, etc.), Hibernate proporciona un conjunto de anotaciones propias que extienden el comportamiento del mapeo y permiten optimizar el rendimiento, aplicar filtros dinamicos, modificar el SQL generado o definir estrategias de generacion de identidad mas flexibles.

## @DynamicInsert y @DynamicUpdate

Por defecto, Hibernate genera sentencias INSERT y UPDATE que incluyen todas las columnas de la entidad, incluso aquellas cuyo valor es `null`. Esto puede ser ineficiente en tablas con muchas columnas opcionales o en sistemas de alta concurrencia.

`@DynamicInsert` genera un INSERT que incluye solo las columnas con valor no nulo. `@DynamicUpdate` genera un UPDATE que incluye solo las columnas cuyo valor ha cambiado realmente.

```java
@Entity
@Table(name = "productos")
@DynamicInsert
@DynamicUpdate
public class Producto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String nombre;

    @Column
    private String descripcion;

    @Column
    private BigDecimal precioOferta;
}
```

> `@DynamicUpdate` tiene un coste: Hibernate debe mantener un snapshot de la entidad al cargarla para detectar que columnas han cambiado. En entornos con pocas actualizaciones, el beneficio supera el coste.

## @Where y @Filter / @FilterDef

`@Where` permite anadir una clausula WHERE fija a nivel de clase, de modo que todas las consultas que carguen entidades de ese tipo incluyan automaticamente la condicion especificada.

```java
@Entity
@Table(name = "usuarios")
@Where(clause = "activo = true")
public class Usuario {
    // Solo se recuperaran usuarios con activo = true
}
```

`@FilterDef` y `@Filter` habilitan filtros dinamicos que se activan en tiempo de ejecucion. El filtro se define con `@FilterDef` indicando el nombre y los parametros, y luego se aplica con `@Filter` sobre la entidad o la coleccion.

```java
@Entity
@Table(name = "pedidos")
@FilterDef(name = "filtroFecha", parameters = @ParamDef(name = "fechaMin", type = "date"))
@Filter(name = "filtroFecha", condition = "fecha_creacion >= :fechaMin")
public class Pedido {
    // ...
}

// Activacion en tiempo de ejecucion
session.enableFilter("filtroFecha")
    .setParameter("fechaMin", LocalDate.now().minusDays(30));
```

## @Formula

`@Formula` permite mapear una propiedad de la entidad a una expresion SQL calculada, no a una columna real. El resultado de la expresion se incluye en las consultas SELECT como una subconsulta o funcion.

```java
@Entity
@Table(name = "empleados")
public class Empleado {

    @Id
    private Long id;

    @Column
    private BigDecimal salario;

    @Column
    private BigDecimal comision;

    @Formula("(salario + COALESCE(comision, 0))")
    private BigDecimal salarioTotal;
}
```

> Las propiedades anotadas con `@Formula` son de solo lectura. Cualquier intento de modificar su valor y persistirlo sera ignorado por Hibernate.

## @GenericGenerator

Hibernate amplia las estrategias de generacion de identidad de JPA con la anotacion `@GenericGenerator`, que permite definir generadores propios o utilizar generadores adicionales como `uuid2`, `guid`, `native`, `enhanced-sequence`, `pooled-lo`, entre otros.

```java
@Entity
@Table(name = "pedidos")
public class Pedido {

    @Id
    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    private String id;
}
```

El generador `uuid2` produce identificadores unicos universales en formato String, utiles cuando se necesita evitar la dependencia de secuencias en la base de datos o en sistemas distribuidos.

## @Type

`@Type` permite especificar un tipo personalizado de Hibernate para una propiedad, indicando la clase que implementa `UserType` o `CompositeUserType`. Esto es util para mapear tipos que no tienen una correspondencia directa con los tipos SQL estandar, como enumerados complejos, JSON, arrays o tipos geometricos.

```java
@Entity
@Table(name = "configuraciones")
public class Configuracion {

    @Id
    private Long id;

    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadatos;
}
```

Hibernate 6 incorpora soporte nativo para tipos JSON mediante `@JdbcTypeCode(SqlTypes.JSON)`, pero en versiones anteriores `@Type` con un `UserType` personalizado era la unica opcion.

## @ColumnTransformer

`@ColumnTransformer` permite modificar la lectura o escritura de una columna en tiempo de ejecucion inyectando expresiones SQL arbitrarias en el INSERT/SELECT. Es especialmente util para trabajar con columnas cifradas.

```java
@Entity
@Table(name = "tarjetas")
public class TarjetaCredito {

    @Id
    private Long id;

    @Column(name = "numero_cuenta", length = 64)
    @ColumnTransformer(
        read = "pgp_sym_decrypt(numero_cuenta, 'clave_secreta')",
        write = "pgp_sym_encrypt(?, 'clave_secreta')"
    )
    private String numeroCuenta;
}
```

> `@ColumnTransformer` solo aplica a la columna especifica de una propiedad. Si se necesita transformar toda la entidad, se debe usar `@SqlInsert`, `@SqlUpdate` o `@SqlDelete` a nivel de clase.

## @Generated

`@Generated` indica que el valor de una propiedad es generado por la base de datos, ya sea en el momento de la insercion (`GenerationTime.INSERT`), de la actualizacion (`GenerationTime.ALWAYS`) o en ambos. Hibernate ejecutara un SELECT adicional tras la operacion para obtener el valor generado.

```java
@Entity
@Table(name = "auditoria")
public class Auditoria {

    @Id
    private Long id;

    @Generated(GenerationTime.INSERT)
    @Column(name = "fecha_creacion", columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime fechaCreacion;
}
```

> A diferencia de `@GeneratedValue`, que genera la clave primaria, `@Generated` se usa para columnas cuyo valor es calculado por la base de datos, como timestamps de auditoria, columnas auto-calculadas o valores por defecto de la base de datos.
