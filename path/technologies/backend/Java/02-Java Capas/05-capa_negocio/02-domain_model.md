# Domain Model

El modelo de dominio es la representacion del negocio en codigo. Son las clases y objetos que reflejan los conceptos, reglas y procesos del dominio del problema. Mientras que la capa de persistencia se preocupa de como se almacenan los datos, el modelo de dominio se preocupa de que significan esos datos y que reglas los gobiernan. Un modelo de dominio bien disenado es la base sobre la que se construye toda la logica de la aplicacion.

## Entidades

Una entidad es un objeto del dominio que posee una identidad unica y continua a lo largo del tiempo. Dos entidades pueden tener los mismos valores en todos sus atributos pero seguir siendo distintas si su identidad es diferente. La identidad suele representarse mediante un identificador unico (ID), aunque puede ser una combinacion de atributos que garanticen la unicidad.

```java
public class User {
    private final UserId id;
    private String name;
    private Email email;
    private boolean active;

    public User(UserId id, String name, Email email) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.active = true;
    }

    public void changeEmail(Email newEmail) {
        if (!this.active) {
            throw new IllegalStateException("No se puede cambiar el email de un usuario inactivo");
        }
        this.email = newEmail;
    }

    public void deactivate() {
        this.active = false;
    }

    public UserId getId() {
        return id;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return id.equals(user.id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

Una entidad tiene un ciclo de vida: se crea, se modifica a traves de eventos de negocio y eventualmente se elimina o desactiva. La identidad la acompania durante todo este ciclo, independientemente de como cambien sus atributos.

## Value Objects

Un Value Object es un objeto del dominio que se define exclusivamente por sus atributos, no por una identidad. Dos Value Objects son iguales si todos sus atributos son iguales. Son inmutables por naturaleza: una vez creados, su estado no puede cambiar. Si se necesita un valor diferente, se crea una nueva instancia.

```java
public record Email(String value) {
    public Email {
        if (value == null || !value.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")) {
            throw new IllegalArgumentException("Email invalido: " + value);
        }
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("El monto no puede ser negativo");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("No se pueden sumar monedas diferentes");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }
}
```

Los Value Objects se usan para modelar conceptos que no tienen identidad propia en el negocio: una cantidad de dinero, una direccion de email, un rango de fechas, una direccion postal. Al usar Value Objects en lugar de tipos primitivos, el codigo se vuelve mas expresivo y seguro. Un `Email` garantiza que la cadena que contiene tiene un formato valido; un `Money` asegura que las operaciones aritmeticas respetan la moneda.

> Usar tipos primitivos para conceptos de negocio es una forma de deuda tecnica conocida como "obsession por tipos primitivos". Cada vez que un concepto de negocio tiene reglas asociadas (validacion, formato, calculos), merece ser un Value Object.

## Diferencia entre Domain y Entity

En el contexto de una arquitectura por capas, es importante distinguir entre el objeto de dominio (Domain) y la entidad de persistencia (Entity). El objeto de dominio es un concepto puro del negocio, sin ninguna preocupacion tecnica. La entidad de persistencia es una representacion de como se almacenan los datos en la base de datos, con anotaciones JPA, restricciones de esquema y relaciones fisicas.

```java
// Objeto de dominio: puro, sin dependencias tecnicas
public class Product {
    private final ProductId id;
    private String name;
    private Money price;
    private Stock stock;

    public void adjustPrice(Money newPrice) {
        if (newPrice == null) {
            throw new IllegalArgumentException("El precio no puede ser nulo");
        }
        this.price = newPrice;
    }

    public boolean isAvailable(int quantity) {
        return stock.hasEnough(quantity);
    }
}

// Entidad de persistencia: tecnica, con anotaciones JPA
@Entity
@Table(name = "products")
public class ProductEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private int stock;
}
```

En aplicaciones simples, el objeto de dominio y la entidad de persistencia suelen ser la misma clase. Sin embargo, en aplicaciones complejas, separarlos permite que el dominio no se contamine con consideraciones tecnicas como anotaciones JPA, indices de base de datos o reglas de serializacion JSON.

## Rich Domain Model vs Anemic Domain Model

El modelo de dominio puede ser rico o anémico dependiendo de donde se ubique la logica de negocio.

Un Rich Domain Model (modelo de dominio rico) coloca el comportamiento y las reglas de negocio dentro de las propias entidades y Value Objects. Las entidades no son simples contenedores de datos, sino objetos que encapsulan tanto estado como comportamiento. Un `User` sabe como cambiar su email, un `Order` sabe calcular su total, un `Product` sabe si tiene stock suficiente.

```java
// Rich Domain Model: la entidad tiene comportamiento
public class Order {
    private final List<OrderLine> lines;
    private OrderStatus status;
    private PaymentInfo payment;

    public void addProduct(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Solo se pueden anadir productos a pedidos en borrador");
        }
        lines.add(new OrderLine(product, quantity));
    }

    public Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public void submit() {
        if (lines.isEmpty()) {
            throw new IllegalStateException("No se puede enviar un pedido sin productos");
        }
        this.status = OrderStatus.SUBMITTED;
    }
}
```

Un Anemic Domain Model (modelo de dominio anémico) es aquel en el que las entidades son meros contenedores de datos con getters y setters, sin ninguna logica de negocio. Toda la logica reside en los Services, que operan sobre los datos de las entidades como si fueran estructuras de datos en lugar de objetos.

```java
// Anemic Domain Model: la entidad solo tiene datos
public class Order {
    private Long id;
    private List<OrderLine> lines;
    private String status;
    private BigDecimal total;
    private Long paymentTransactionId;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public List<OrderLine> getLines() { return lines; }
    public void setLines(List<OrderLine> lines) { this.lines = lines; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public BigDecimal getTotal() { return total; }
    public void setTotal(BigDecimal total) { this.total = total; }
}

// Toda la logica termina en el Service
public class OrderService {
    public void submitOrder(Order order) {
        if (order.getLines() == null || order.getLines().isEmpty()) {
            throw new IllegalStateException("No se puede enviar un pedido sin productos");
        }
        BigDecimal total = BigDecimal.ZERO;
        for (OrderLine line : order.getLines()) {
            total = total.add(line.getPrice().multiply(BigDecimal.valueOf(line.getQuantity())));
        }
        order.setTotal(total);
        order.setStatus("SUBMITTED");
    }
}
```

El Anemic Domain Model esta considerado un anti-patron en la mayoria de las comunidades de diseno orientado a objetos. Martin Fowler lo describio como un anti-patron porque contradice el principio fundamental de la programacion orientada a objetos: combinar datos y comportamiento en una misma unidad. Sin embargo, es extremadamente comun en aplicaciones Java empresariales, en parte porque frameworks como JPA y Spring favorecen este enfoque al requerir entidades con constructores sin argumentos y setters publicos.

> El Anemic Domain Model no es intrinsecamente malo. En aplicaciones simples o CRUD, puede ser perfectamente valido. El problema surge en dominios complejos donde la logica de negocio es rica y termina dispersa en los Services, duplicandose y perdiendo cohesión. La clave esta en reconocer cuando el dominio justifica un modelo rico.
