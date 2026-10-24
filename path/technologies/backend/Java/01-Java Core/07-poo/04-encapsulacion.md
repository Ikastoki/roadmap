# Encapsulacion

La encapsulacion es el mecanismo que oculta el estado interno de un objeto y protege los datos de accesos directos no controlados. En Java, la encapsulacion se implementa mediante modificadores de acceso (especialmente `private` para los campos) y metodos publicos que actuan como interfaz controlada.

## Getters y setters con validacion

Los metodos getter y setter (tambien llamados accessors y mutators) son metodos publicos que permiten leer y modificar campos privados de forma controlada.

```java
public class CuentaBancaria {
    private double saldo;
    private String titular;

    public double getSaldo() {
        return saldo;
    }

    public void depositar(double cantidad) {
        if (cantidad <= 0) {
            throw new IllegalArgumentException("La cantidad debe ser positiva");
        }
        this.saldo += cantidad;
    }

    public void retirar(double cantidad) {
        if (cantidad <= 0) {
            throw new IllegalArgumentException("La cantidad debe ser positiva");
        }
        if (cantidad > saldo) {
            throw new IllegalStateException("Fondos insuficientes");
        }
        this.saldo -= cantidad;
    }

    public String getTitular() {
        return titular;
    }

    public void setTitular(String titular) {
        if (titular == null || titular.trim().isEmpty()) {
            throw new IllegalArgumentException("El titular no puede estar vacio");
        }
        this.titular = titular.trim();
    }
}
```

En este ejemplo, el saldo no tiene un setter tradicional. Solo puede modificarse mediante `depositar()` y `retirar()`, que incorporan validacion. Esto evita estados invalidos como saldos negativos o depositos de cantidades negativas.

> Un getter no debe exponer directamente referencias a objetos mutables internos. Para colecciones debe devolverse una copia o una vista no modificable. Para arrays, una copia defensiva con `clone()` o `Arrays.copyOf()`.

## Patron JavaBean

El patron JavaBean es una convencion para disenar clases reutilizables que puedan manipularse mediante herramientas graficas y frameworks. Establece reglas especificas para getters y setters:

- La clase debe tener un constructor sin argumentos
- Los campos deben ser `private`
- Los getters siguen la forma `getPropiedad()` para propiedades no booleanas, `isPropiedad()` para booleanas
- Los setters siguen la forma `setPropiedad(Tipo valor)`

```java
public class UsuarioBean {
    private String nombre;
    private int edad;
    private boolean activo;

    public UsuarioBean() {
    }

    public String getNombre() {
        return nombre;
    }

    public void setNombre(String nombre) {
        this.nombre = nombre;
    }

    public int getEdad() {
        return edad;
    }

    public void setEdad(int edad) {
        this.edad = edad;
    }

    public boolean isActivo() {
        return activo;
    }

    public void setActivo(boolean activo) {
        this.activo = activo;
    }
}
```

Frameworks como Spring, Hibernate, Jackson y Jakarta Faces dependen del patron JavaBean para la inyeccion de dependencias, serializacion, mapeo objeto-relacional y binding de formularios.

## Beneficios de la encapsulacion

**Proteccion del estado del objeto.** Al controlar como se accede y modifica cada campo, se garantiza que el objeto nunca entre en un estado inconsistente. Las invariantes de la clase se mantienen siempre.

```java
public class Rango {
    private int minimo;
    private int maximo;

    public Rango(int minimo, int maximo) {
        if (minimo > maximo) {
            throw new IllegalArgumentException(
                "El minimo no puede ser mayor que el maximo");
        }
        this.minimo = minimo;
        this.maximo = maximo;
    }

    public void setMinimo(int minimo) {
        if (minimo > this.maximo) {
            throw new IllegalArgumentException(
                "El minimo no puede superar al maximo actual");
        }
        this.minimo = minimo;
    }

    public void setMaximo(int maximo) {
        if (maximo < this.minimo) {
            throw new IllegalArgumentException(
                "El maximo no puede ser menor al minimo actual");
        }
        this.maximo = maximo;
    }
}
```

**Cambio de implementacion sin afectar a consumidores.** Al acceder a los datos mediante metodos, la implementacion interna puede modificarse sin cambiar la API publica.

```java
public class Temperatura {
    private double celsius; // representacion interna

    public double getCelsius() {
        return celsius;
    }

    public void setCelsius(double celsius) {
        this.celsius = celsius;
    }

    // Se puede anadir nueva funcionalidad sin romper codigo existente
    public double getFahrenheit() {
        return celsius * 9.0 / 5.0 + 32;
    }

    public void setFahrenheit(double fahrenheit) {
        this.celsius = (fahrenheit - 32) * 5.0 / 9.0;
    }
}
```

Si se decide cambiar la representacion interna de celsius a fahrenheit, los getters y setters existentes se adaptan y el codigo que usa la clase no requiere cambios.

**Facilidad de mantenimiento y depuracion.** Los puntos de acceso controlados permiten anadir logging, validacion, sincronizacion o lazy initialization sin modificar el codigo que consume la clase.

```java
public class Servicio {
    private ClienteHttp cliente;

    public ClienteHttp getCliente() {
        if (cliente == null) {
            cliente = new ClienteHttp(); // lazy initialization
        }
        return cliente;
    }

    public void setCliente(ClienteHttp cliente) {
        if (cliente == null) {
            throw new IllegalArgumentException("Cliente no puede ser null");
        }
        this.cliente = cliente;
    }
}
```

La encapsulacion es, junto con la herencia y el polimorfismo, uno de los pilares de la programacion orientada a objetos. Sin encapsulacion, los objetos son meros contenedores de datos sin garantias de integridad.
