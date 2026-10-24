# Jackson (JSON)

Jackson es la biblioteca de serialización/deserialización JSON por defecto en Spring Boot. Se integra automáticamente a través de `spring-boot-starter-web`, configurando un `ObjectMapper` listo para usar con convenciones de Spring. Jackson proporciona anotaciones para controlar cómo los objetos Java se convierten a JSON y viceversa, permitiendo personalizar el comportamiento de serialización a nivel de campo, clase o globalmente.

`@JsonProperty` permite personalizar el nombre de una propiedad en el JSON. Esta anotación es útil cuando el nombre del campo en Java no coincide con el nombre esperado en el JSON, o cuando se necesita usar un nombre diferente por razones de compatibilidad con APIs externas.

```java
public class Usuario {
    
    @JsonProperty("user_id")
    private Long id;
    
    @JsonProperty("full_name")
    private String nombreCompleto;
    
    @JsonProperty("email_address")
    private String email;
    
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private LocalDateTime fechaCreacion;
    
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;
}
```

`@JsonIgnore` excluye una propiedad completamente de la serialización JSON. Esto es útil para campos que nunca deben exponerse al exterior, como contraseñas, tokens de seguridad o datos internos. La propiedad será ignorada tanto en serialización como en deserialización.

`@JsonInclude` controla qué propiedades se incluyen en el JSON basándose en condiciones específicas. `JsonInclude.Include.NON_NULL` excluye propiedades con valores null, `NON_EMPTY` excluye strings vacíos, colecciones vacías y null, y `NON_DEFAULT` excluye propiedades que tienen valores por defecto.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProductoResponse {
    
    private Long id;
    private String nombre;
    private BigDecimal precio;
    
    @JsonInclude(JsonInclude.Include.NON_EMPTY)
    private String descripcion;
    
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private int stock;
    
    private List<String> tags;
}
```

`@JsonFormat` formatea valores de fecha, hora y moneda en el JSON. Esta anotación es especialmente útil para campos de tipo `LocalDate`, `LocalDateTime` y `BigDecimal`, donde el formato de representación es importante para la compatibilidad con el cliente.

```java
public class Pedido {
    
    private Long id;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime fechaCreacion;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd")
    private LocalDate fechaEntrega;
    
    @JsonFormat(shape = JsonFormat.Shape.NUMBER_FLOAT)
    private BigDecimal total;
}
```

Las referencias circulares son un problema común en la serialización JSON cuando dos objetos se referencian mutuamente. Jackson ofrece varias soluciones: `@JsonBackReference` y `@JsonManagedReference` para referencias padre-hijo, `@JsonIdentityInfo` para manejar referencias circulares complejas, y `@JsonIgnore` para romper ciclos simples.

```java
// Solución 1: JsonManagedReference y JsonBackReference
public class Departamento {
    @JsonManagedReference
    private List<Empleado> empleados;
}

public class Empleado {
    @JsonBackReference
    private Departamento departamento;
}

// Solución 2: JsonIdentityInfo para referencias circulares complejas
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, 
                  property = "id")
public class Nodo {
    private Long id;
    private Nodo padre;
    private List<Nodo> hijos;
}
```

El `ObjectMapper` personalizado permite configurar globalmente el comportamiento de Jackson. Spring Boot autoconfigura un `ObjectMapper`, pero se puede personalizar mediante beans o propiedades de configuración. Las configuraciones comunes incluyen fallar en propiedades desconocidas, registrar módulos personalizados y configurar fechas.

```java
@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // Configuración de serialización
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        mapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        
        // Configuración de deserialización
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.configure(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true);
        
        // Registrar módulos
        mapper.registerModule(new JavaTimeModule());
        mapper.registerModule(new Jdk8Module());
        
        // Configurar lenidad para números
        mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);
        
        return mapper;
    }
}
```

Los serializadores y deserializadores personalizados permiten controlar completamente cómo se convierten los objetos a JSON y viceversa. Se definen extendiendo `JsonSerializer<T>` y `JsonDeserializer<T>`, y se registran mediante anotaciones o configuración del `ObjectMapper`. Esto es útil para tipos personalizados que no son soportados nativamente por Jackson.

```java
public class MonedaSerializador extends JsonSerializer<Moneda> {
    
    @Override
    public void serialize(Moneda moneda, JsonGenerator gen, SerializerProvider provider) 
            throws IOException {
        gen.writeString(moneda.getCantidad() + " " + moneda.getSimbolo());
    }
}

public class MonedaDeserializador extends JsonDeserializer<Moneda> {
    
    @Override
    public Moneda deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException {
        String texto = p.getText();
        String[] partes = texto.split(" ");
        BigDecimal cantidad = new BigDecimal(partes[0]);
        String simbolo = partes[1];
        return new Moneda(cantidad, simbolo);
    }
}

public class Moneda {
    
    @JsonSerialize(using = MonedaSerializador.class)
    @JsonDeserialize(using = MonedaDeserializador.class)
    private BigDecimal cantidad;
    private String simbolo;
}
```

`@JsonCreator` define un constructor o método de fábrica para la deserialización de objetos inmutables. Cuando una clase no tiene setters, Jackson necesita una forma de crear instancias con valores. `@JsonCreator` con `@JsonProperty` permite mapear propiedades del JSON a parámetros del constructor.

`@JsonValue` especifica un método que retorna el valor que se debe serializar en lugar del objeto completo. Esto es útil para objetos que tienen una representación JSON simple pero un modelo complejo internamente.

```java
public class RangoFecha {
    
    private final LocalDate inicio;
    private final LocalDate fin;
    
    @JsonCreator
    public RangoFecha(@JsonProperty("inicio") LocalDate inicio,
                     @JsonProperty("fin") LocalDate fin) {
        this.inicio = inicio;
        this.fin = fin;
    }
    
    @JsonValue
    public String toJson() {
        return inicio + " al " + fin;
    }
}
```

> Jackson es altamente configurable pero puede ser complejo. Para casos comunes, las anotaciones estándar son suficientes. Para necesidades avanzadas, considerar usar módulos como `jackson-datatype-jsr310` para fechas Java 8 o `jackson-datatype-hibernate` para entidades JPA.

