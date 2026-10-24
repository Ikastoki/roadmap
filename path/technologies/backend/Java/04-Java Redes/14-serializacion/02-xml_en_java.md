# XML en Java

XML (eXtensible Markup Language) fue durante anos el formato dominante para el intercambio de datos en entornos empresariales. Aunque JSON lo ha desplazado en APIs web modernas, XML sigue siendo relevante en sectores como banca, sanidad, telecomunicaciones y protocolos como SOAP o RSS. Java ofrece multiples APIs para trabajar con XML, desde el modelo de arbol (DOM) hasta el streaming (StAX), pasando por el mapeo objeto-XML con JAXB.

## JAXB (Jakarta XML Binding)

JAXB permite mapear clases Java a documentos XML y viceversa mediante anotaciones, de forma analoga a como Jackson mapea a JSON. Desde Java 11, JAXB fue removido del JDK y debe incluirse como dependencia externa bajo el nombre Jakarta XML Binding.

```xml
<dependency>
    <groupId>jakarta.xml.bind</groupId>
    <artifactId>jakarta.xml.bind-api</artifactId>
    <version>4.0.2</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.5</version>
    <scope>runtime</scope>
</dependency>
```

### Anotaciones basicas

```java
import jakarta.xml.bind.annotation.*;
import java.util.List;

@XmlRootElement(name = "usuario")
@XmlAccessorType(XmlAccessType.FIELD)
public class Usuario {

    @XmlAttribute
    private Long id;

    @XmlElement(name = "nombre_completo")
    private String nombre;

    @XmlElement
    private String email;

    @XmlElementWrapper(name = "direcciones")
    @XmlElement(name = "direccion")
    private List<String> direcciones;

    // Constructor sin args requerido por JAXB
    public Usuario() {}

    // getters y setters...
}
```

- `@XmlRootElement`: marca la clase como elemento raiz del documento XML.
- `@XmlAccessorType(FIELD)`: usa los campos directamente en lugar de getters/setters.
- `@XmlAttribute`: serializa el campo como atributo del elemento XML.
- `@XmlElement`: serializa el campo como elemento hijo.
- `@XmlElementWrapper`: crea un elemento contenedor para listas.

### Marshaller (Java a XML)

```java
import jakarta.xml.bind.*;

JAXBContext context = JAXBContext.newInstance(Usuario.class);
Marshaller marshaller = context.createMarshaller();
marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);

Usuario usuario = new Usuario();
usuario.setId(1L);
usuario.setNombre("Ana Garcia");
usuario.setEmail("ana@example.com");

marshaller.marshal(usuario, System.out);
```

Salida:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<usuario id="1">
    <nombre_completo>Ana Garcia</nombre_completo>
    <email>ana@example.com</email>
</usuario>
```

### Unmarshaller (XML a Java)

```java
String xml = "<usuario id=\"2\">"
        + "<nombre_completo>Luis Perez</nombre_completo>"
        + "<email>luis@example.com</email>"
        + "</usuario>";

Unmarshaller unmarshaller = context.createUnmarshaller();
Usuario u = (Usuario) unmarshaller.unmarshal(
        new StringReader(xml));
System.out.println(u.getNombre()); // Luis Perez
```

> JAXB requiere que la clase tenga un constructor sin argumentos. Si la clase es inmutable (sin constructor default), es necesario usar la anotacion `@XmlType` con `factoryMethod` o un adaptador `XmlAdapter`.

## StAX (Streaming API for XML)

StAX (Streaming API for XML) es una API de streaming bidireccional que permite leer y escribir XML sin cargar todo el documento en memoria. Es ideal para procesar documentos XML de gran tamano que no cabrian en un arbol DOM.

### XMLStreamReader (lectura)

```java
import javax.xml.stream.*;

XMLInputFactory factory = XMLInputFactory.newInstance();
XMLStreamReader reader = factory.createXMLStreamReader(
        new FileInputStream("datos.xml"));

while (reader.hasNext()) {
    int evento = reader.next();
    switch (evento) {
        case XMLStreamConstants.START_ELEMENT ->
            System.out.println("Inicio: " + reader.getLocalName());
        case XMLStreamConstants.CHARACTERS ->
            System.out.println("Texto: " + reader.getText().trim());
        case XMLStreamConstants.END_ELEMENT ->
            System.out.println("Fin: " + reader.getLocalName());
    }
}
reader.close();
```

### XMLStreamWriter (escritura)

```java
XMLOutputFactory factory = XMLOutputFactory.newInstance();
XMLStreamWriter writer = factory.createXMLStreamWriter(
        new FileOutputStream("salida.xml"));

writer.writeStartDocument("UTF-8", "1.0");
writer.writeStartElement("usuarios");
writer.writeStartElement("usuario");
writer.writeAttribute("id", "1");
writer.writeStartElement("nombre");
writer.writeCharacters("Ana Garcia");
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndElement();
writer.writeEndDocument();

writer.flush();
writer.close();
```

> StAX es preferible cuando se procesan documentos XML de gran tamano (cientos de megabytes) porque el consumo de memoria es constante e independiente del tamano del documento. JAXB y DOM, en cambio, cargan el documento completo en memoria.

## DOM (Document Object Model)

DOM carga todo el XML en un arbol en memoria. Es util cuando se necesita acceso aleatorio al documento o modificaciones estructurales.

```java
import javax.xml.parsers.*;
import org.w3c.dom.*;

DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(new File("datos.xml"));

NodeList nodos = doc.getElementsByTagName("usuario");
for (int i = 0; i < nodos.getLength(); i++) {
    Element elem = (Element) nodos.item(i);
    String id = elem.getAttribute("id");
    String nombre = elem
            .getElementsByTagName("nombre_completo")
            .item(0).getTextContent();
    System.out.println(id + ": " + nombre);
}
```

| API | Modelo | Memoria | Uso tipico |
|---|---|---|---|
| JAXB | Mapeo objeto-XML | Alta (carga completa) | Configuracion, intercambio |
| StAX | Streaming | Baja (constante) | Documentos grandes, alto rendimiento |
| DOM | Arbol en memoria | Alta (completa) | Modificacion estructural |

## SAX (Simple API for XML)

SAX es una API de streaming orientada a eventos, similar a StAX pero con direccion push (el parser notifica eventos al handler). Aunque funcional, StAX es mas facil de usar al ser pull-based (el codigo cliente controla cuando leer el siguiente evento).
