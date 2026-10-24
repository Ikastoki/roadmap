# Observer

Observer es un patron de comportamiento que define una dependencia uno-a-muchos entre objetos, de modo que cuando un objeto cambia su estado, todos sus dependientes son notificados y actualizados automaticamente. Es la base de los sistemas de eventos, la arquitectura publish-subscribe, y la programacion reactiva.

## Sujeto y observadores

El patron se compone de dos roles principales. El _sujeto_ (observable) mantiene una lista de _observadores_ y los notifica ante cualquier cambio de estado. Los _observadores_ definen la accion a ejecutar cuando reciben la notificacion. La relacion es debil: el sujeto conoce a los observadores solo a traves de una interfaz, sin saber sus tipos concretos.

```java
// Interfaz observador
interface Observador {
    void actualizar(String evento);
}

// Interfaz sujeto
interface Observable {
    void agregar(Observador obs);
    void eliminar(Observador obs);
    void notificar(String evento);
}

// Sujeto concreto
class SensorTemperatura implements Observable {
    private List<Observador> observadores = new ArrayList<>();
    private double temperatura;

    public void setTemperatura(double temp) {
        this.temperatura = temp;
        notificar("Temperatura: " + temp);
    }

    @Override
    public void agregar(Observador obs) {
        observadores.add(obs);
    }

    @Override
    public void eliminar(Observador obs) {
        observadores.remove(obs);
    }

    @Override
    public void notificar(String evento) {
        for (Observador obs : observadores) {
            obs.actualizar(evento);
        }
    }
}

// Observadores concretos
class DisplayLCD implements Observador {
    @Override
    public void actualizar(String evento) {
        System.out.println("LCD muestra: " + evento);
    }
}

class Alarma implements Observador {
    @Override
    public void actualizar(String evento) {
        if (evento.contains("alta")) {
            System.out.println("ALARMA: " + evento);
        }
    }
}
```

El sistema se configura en tiempo de ejecucion registrando observadores:

```java
SensorTemperatura sensor = new SensorTemperatura();
sensor.agregar(new DisplayLCD());
sensor.agregar(new Alarma());

sensor.setTemperatura(35.5); // ambos observadores reciben la notificacion
```

## Acoplamiento debil

La principal ventaja de Observer es el acoplamiento debil entre sujeto y observadores. El sujeto solo conoce la interfaz `Observador`; los observadores concretos pueden anadirse, eliminarse o modificarse sin afectar al sujeto ni a otros observadores. Esto permite variar el comportamiento de respuesta sin modificar el emisor.

Un mismo observador puede registrarse en multiples sujetos:

```java
class Logger implements Observador {
    @Override
    public void actualizar(String evento) {
        System.out.println("LOG: " + evento);
    }
}

Logger logger = new Logger();
sensor.agregar(logger);
otroSensor.agregar(logger);
```

Y un sujeto puede tener cualquier numero de observadores, incluyendo cero.

## Notificacion push vs pull

Existen dos formas de notificar. En el modelo _push_, el sujeto envia los datos del cambio como parte de la notificacion (como en el ejemplo anterior). En el modelo _pull_, el sujeto solo notifica que hubo un cambio y cada observador consulta el estado que necesita:

```java
// Modelo pull: el observador consulta el estado del sujeto
interface ObservadorPull {
    void actualizar(Observable sujeto);
}

class DisplayPull implements ObservadorPull {
    @Override
    public void actualizar(Observable sujeto) {
        SensorTemperatura sensor = (SensorTemperatura) sujeto;
        double temp = sensor.getTemperatura();
        System.out.println("Temperatura actual: " + temp);
    }
}
```

El modelo pull es mas flexible (cada observador decide que datos necesita) pero requiere que los observadores conozcan el tipo concreto del sujeto para acceder a su estado.

## Ejemplos en Java

### EventListener

La API de Swing utiliza Observer extensivamente. Los componentes son sujetos y los listeners son observadores:

```java
JButton boton = new JButton("Haz clic");
boton.addActionListener(event -> System.out.println("Clic!"));
boton.addActionListener(event -> contador++);
```

Cada `ActionListener` es un observador. El boton los notifica cuando ocurre el evento. La interfaz `ActionListener` tiene un unico metodo (`actionPerformed`), y las lambdas permiten implementarla concisamente.

### PropertyChangeListener

JavaBeans proporciona `PropertyChangeSupport` para implementar el patron Observer sobre cambios de propiedades:

```java
public class Producto {
    private PropertyChangeSupport support = new PropertyChangeSupport(this);
    private double precio;

    public void addPropertyChangeListener(PropertyChangeListener listener) {
        support.addPropertyChangeListener(listener);
    }

    public void removePropertyChangeListener(PropertyChangeListener listener) {
        support.removePropertyChangeListener(listener);
    }

    public void setPrecio(double nuevoPrecio) {
        double anterior = this.precio;
        this.precio = nuevoPrecio;
        support.firePropertyChange("precio", anterior, nuevoPrecio);
    }
}
```

### Flow API (Java 9+)

`java.util.concurrent.Flow` define un estandar reactivo basado en Observer con contrapresion (backpressure):

```java
SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
publisher.subscribe(new Flow.Subscriber<>() {
    private Flow.Subscription subscription;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(String item) {
        System.out.println("Recibido: " + item);
        subscription.request(1);
    }

    @Override
    public void onError(Throwable t) { t.printStackTrace(); }

    @Override
    public void onComplete() { System.out.println("Completado"); }
});

publisher.submit("Dato 1");
publisher.submit("Dato 2");
publisher.close();
```

> El patron Observer es omnipresente en Java. Cada vez que se usa un listener, un callback, o un handler, se esta aplicando una variante de Observer. Conocer el patron ayuda a disenar sistemas de eventos coherentes en lugar de soluciones ad-hoc con callbacks dispersos.
