# Mappers

Un mapper es el componente responsable de convertir objetos de un tipo a otro. En una arquitectura por capas, los mappers traducen entre entidades de persistencia, modelos de dominio y DTOs de presentacion. Sin ellos, cada conversion quedaria dispersa en servicios y controladores, duplicando logica y dificultando el mantenimiento.

## Mapper manual

El mapper manual consiste en escribir el codigo de conversion a mano, campo por campo. Es el enfoque mas verboso pero tambien el que ofrece control total sobre cada transformacion.

```java
public class UserMapper {

    public UserDTO toDomain(UserEntity entity) {
        if (entity == null) return null;
        return new UserDTO(
            entity.getId(),
            entity.getName(),
            entity.getEmail(),
            entity.getCreatedAt()
        );
    }

    public UserEntity toEntity(UserDTO dto) {
        if (dto == null) return null;
        UserEntity entity = new UserEntity();
        entity.setName(dto.name());
        entity.setEmail(dto.email());
        entity.setCreatedAt(dto.createdAt());
        return entity;
    }

    public UserResponse toResponse(UserDTO dto) {
        if (dto == null) return null;
        return new UserResponse(
            dto.id(),
            dto.name(),
            dto.email(),
            dto.createdAt()
        );
    }

    public UserSummary toSummary(UserDTO dto) {
        if (dto == null) return null;
        return new UserSummary(
            dto.id(),
            dto.name()
        );
    }
}
```

La ventaja principal del mapper manual es la visibilidad: cada conversion esta explicitamente escrita, no hay magia ni comportamiento inesperado. La desventaja es la cantidad de codigo boilerplate que hay que mantener, especialmente cuando hay muchas entidades y DTOs.

```java
// Mapper manual con logica de transformacion compleja
public OrderDTO toDomain(OrderEntity entity) {
    if (entity == null) return null;

    List<OrderItemDTO> items = entity.getItems().stream()
        .map(this::toItemDTO)
        .toList();

    BigDecimal subtotal = items.stream()
        .map(item -> item.price().multiply(BigDecimal.valueOf(item.quantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);

    BigDecimal tax = subtotal.multiply(new BigDecimal("0.21"));

    return new OrderDTO(
        entity.getId(),
        entity.getUserId(),
        items,
        subtotal,
        tax,
        subtotal.add(tax),
        entity.getStatus()
    );
}
```

> El mapper manual es la opcion recomendada cuando las transformaciones implican logica de negocio, calculos o agregaciones. En estos casos, un mapper automatico ocultaria la complejidad real de la conversion.

## MapStruct

MapStruct es un generador de codigo en tiempo de compilacion. Analiza interfaces de mapper y genera las implementaciones durante la compilacion, eliminando el boilerplate sin introducir overhead en tiempo de ejecucion.

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

    @Mapping(target = "passwordHash", ignore = true)
    @Mapping(target = "createdAt", expression = "java(java.time.LocalDateTime.now())")
    UserEntity toEntity(UserDTO dto);

    @Mapping(source = "email", target = "emailAddress")
    UserDTO toDomain(UserEntity entity);

    List<UserDTO> toDomainList(List<UserEntity> entities);
}
```

MapStruct soporta mapeo de campos con diferentes nombres (`source`/`target`), expresiones, ignorar campos, conversiones de tipos automaticas y mappers personalizados para tipos complejos.

```java
@Mapper(componentModel = "spring", uses = {DateMapper.class, MoneyMapper.class})
public interface OrderMapper {

    @Mapping(target = "items", source = "orderItems")
    @Mapping(target = "total", source = "totalAmount", qualifiedByName = "bigDecimalToMoney")
    OrderDTO toDomain(OrderEntity entity);

    @AfterMapping
    default void afterMapping(OrderEntity entity, @MappingTarget OrderDTO.OrderDTOBuilder builder) {
        // Logica post-conversion
        builder.status(entity.getStatus().name());
    }
}
```

La principal ventaja de MapStruct es el rendimiento: al generar codigo en compilacion, no usa reflexion ni proxies, siendo esencialmente tan rapido como un mapper manual. Ademas, reporta errores de mapeo en tiempo de compilacion, no en ejecucion.

## ModelMapper

ModelMapper es un mapper automatico basado en reflexion que copia propiedades por coincidencia de nombres. A diferencia de MapStruct, no genera codigo en compilacion sino que resuelve los mapeos en tiempo de ejecucion.

```java
ModelMapper modelMapper = new ModelMapper();

// Configuracion global
modelMapper.getConfiguration()
    .setFieldMatchingEnabled(true)
    .setFieldAccessLevel(Configuration.AccessLevel.PRIVATE)
    .setSkipNullEnabled(true);

// Mapeo basico
UserDTO userDTO = modelMapper.map(userEntity, UserDTO.class);

// Mapeo con configuracion especifica
modelMapper.typeMap(UserEntity.class, UserDTO.class)
    .addMappings(mapper -> {
        mapper.map(UserEntity::getEmail, UserDTO::setEmailAddress);
        mapper.skip(UserDTO::setPasswordHash);
    });
```

ModelMapper requiere menos configuracion inicial que MapStruct, pero tiene desventajas importantes: el mapeo por reflexion es mas lento, los errores aparecen en tiempo de ejecucion y el comportamiento por defecto puede producir resultados inesperados cuando los tipos no coinciden exactamente.

```java
// Configuracion de mapeo condicional
modelMapper.createTypeMap(OrderEntity.class, OrderDTO.class)
    .setPreConverter(context -> {
        OrderEntity source = context.getSource();
        if (source == null) return null;
        // Logica previa al mapeo automatico
        return context.create(source, OrderDTO.class);
    });
```

> ModelMapper puede ser util en prototipos o aplicaciones pequenas, pero en proyectos con decenas de entidades y reglas de mapeo complejas, el coste de depurar errores de mapeo en tiempo de ejecucion supera con creces la comodidad inicial.

## BeanUtils.copyProperties

Spring Framework proporciona `BeanUtils.copyProperties()` como un metodo utilitario para copiar propiedades entre objetos del mismo tipo. Es la opcion mas simple pero tambien la mas limitada.

```java
UserDTO dto = new UserDTO();
BeanUtils.copyProperties(userEntity, dto);
```

El metodo copia propiedades del origen al destino emparejando por nombre. No realiza conversion de tipos (si el origen tiene `Long` y el destino `String`, falla silenciosamente), no ignora campos y no soporta mapeo de nombres diferentes.

```java
public UserDTO toDTO(UserEntity entity) {
    UserDTO dto = new UserDTO();
    BeanUtils.copyProperties(entity, dto);
    // Los campos que no coinciden por nombre hay que mapearlos manualmente
    dto.setFullName(entity.getFirstName() + " " + entity.getLastName());
    dto.setStatus(entity.getActive() ? "active" : "inactive");
    return dto;
}
```

Por su naturaleza generica, `BeanUtils.copyProperties()` oculta errores: si un campo cambia de nombre, la copia simplemente deja de funcionar sin aviso. Es util para prototipos y migraciones rapidas, pero su uso en produccion deberia limitarse a casos donde origen y destino tienen exactamente los mismos tipos y nombres de campo.

```java
// Alternativa mas segura: usar un metodo auxiliar con tipos explicitos
public static <T, R> R copyFields(T source, R target, Class<T> sourceType, Class<R> targetType) {
    BeanUtils.copyProperties(source, target);
    return target;
}
```

> La regla general: usa MapStruct para la mayoria de los casos, mapper manual para transformaciones con logica de negocio, ModelMapper solo en prototipos, y `BeanUtils.copyProperties()` exclusivamente cuando sepas que origen y destino son estructuralmente identicos.
