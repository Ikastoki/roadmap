# Composite

Composite es un patron estructural que permite tratar objetos individuales y compuestos de manera uniforme. Organiza los objetos en una estructura de arbol para representar jerarquias parte-todo, donde los nodos pueden ser elementos individuales (hojas) o contenedores de otros elementos (compuestos). Los clientes interactuan con ambos tipos a traves de la misma interfaz.

## Estructura de arbol

El patron define tres elementos:

- **Component**: la interfaz o clase abstracta que declara las operaciones comunes a todos los nodos.
- **Leaf**: la clase que representa objetos sin hijos. Implementa el comportamiento primitivo del Component.
- **Composite**: la clase que contiene hijos (Leaf u otros Composite) y delega en ellos las operaciones.

```java
// Component
interface ComponenteSistema {
    String getNombre();
    long getTamaño();
    void mostrar(int profundidad);
}

// Leaf: archivo individual
class Archivo implements ComponenteSistema {
    private String nombre;
    private long tamaño;

    public Archivo(String nombre, long tamaño) {
        this.nombre = nombre;
        this.tamaño = tamaño;
    }

    @Override
    public String getNombre() {
        return nombre;
    }

    @Override
    public long getTamaño() {
        return tamaño;
    }

    @Override
    public void mostrar(int profundidad) {
        String indent = "  ".repeat(profundidad);
        System.out.println(indent + "📄 " + nombre + " (" + tamaño + " bytes)");
    }
}

// Composite: directorio que contiene archivos y subdirectorios
class Directorio implements ComponenteSistema {
    private String nombre;
    private List<ComponenteSistema> hijos = new ArrayList<>();

    public Directorio(String nombre) {
        this.nombre = nombre;
    }

    public void agregar(ComponenteSistema componente) {
        hijos.add(componente);
    }

    public void eliminar(ComponenteSistema componente) {
        hijos.remove(componente);
    }

    @Override
    public String getNombre() {
        return nombre;
    }

    @Override
    public long getTamaño() {
        // El tamaño del directorio es la suma de todos sus hijos
        long total = 0;
        for (ComponenteSistema hijo : hijos) {
            total += hijo.getTamaño();
        }
        return total;
    }

    @Override
    public void mostrar(int profundidad) {
        String indent = "  ".repeat(profundidad);
        System.out.println(indent + "📁 " + nombre + "/");
        for (ComponenteSistema hijo : hijos) {
            hijo.mostrar(profundidad + 1);
        }
    }
}
```

El cliente trata todo de manera uniforme:

```java
// Construccion de la estructura
Directorio raiz = new Directorio("raiz");

Directorio documentos = new Directorio("documentos");
documentos.agregar(new Archivo("cv.pdf", 204800));
documentos.agregar(new Archivo("notas.txt", 1024));

Directorio imagenes = new Directorio("imagenes");
imagenes.agregar(new Archivo("foto.jpg", 3145728));
imagenes.agregar(new Archivo("logo.png", 65536));

Directorio proyectos = new Directorio("proyectos");
proyectos.agregar(new Archivo("codigo.java", 4096));
proyectos.agregar(new Archivo("README.md", 512));

imagenes.agregar(proyectos);
raiz.agregar(documentos);
raiz.agregar(imagenes);

// Operacion recursiva: el cliente no distingue entre archivo y directorio
raiz.mostrar(0);
System.out.println("Tamaño total: " + raiz.getTamaño() + " bytes");
```

Salida:

```
📁 raiz/
  📁 documentos/
    📄 cv.pdf (204800 bytes)
    📄 notas.txt (1024 bytes)
  📁 imagenes/
    📄 foto.jpg (3145728 bytes)
    📄 logo.png (65536 bytes)
    📁 proyectos/
      📄 codigo.java (4096 bytes)
      📄 README.md (512 bytes)
Tamaño total: 3413696 bytes
```

## Operaciones recursivas

La clave de Composite es que las operaciones en el Composite se implementan recorriendo recursivamente los hijos. El `getTamaño()` del Directorio suma recursivamente los tamaños de todos los componentes hijos. Si un hijo es otro Directorio, su tamaño se calcula a su vez sumando sus hijos. La recursion natural del arbol simplifica estas operaciones.

El metodo `mostrar()` sigue la misma estructura recursiva, anadiendo indentacion para reflejar la profundidad en el arbol.

## Transparencia vs seguridad

Existe un dilema de diseno en Composite: ubicar los metodos de gestion de hijos (`agregar`, `eliminar`, `getHijos`) en la interfaz Component o solo en Composite.

- **Diseno transparente**: los metodos de gestion de hijos se declaran en Component. Las hojas los implementan lanzando una excepcion o sin operacion. Permite tratar todo uniformemente, pero viola el principio de sustitucion de Liskov (las hojas tienen metodos que no tienen sentido).

```java
interface Componente {
    void agregar(Componente c);
    void eliminar(Componente c);
    // ...
}

class Archivo implements Componente {
    @Override
    public void agregar(Componente c) {
        throw new UnsupportedOperationException("No se puede agregar a un archivo");
    }
}
```

- **Diseno seguro**: los metodos de gestion solo estan en Composite. El cliente debe verificar el tipo si necesita agregar hijos. Es mas seguro pero requiere instanceof.

```java
if (componente instanceof Directorio dir) {
    dir.agregar(nuevoComponente);
}
```

La API de Swing opta por el diseno transparente (`Container` tiene `add(Component)`, y `Component` tiene metodos como `add` que no hacen nada en los componentes hoja).

## Ejemplo en Swing: Component / Container

La jerarquia de Swing es una implementacion real de Composite:

```java
JFrame frame = new JFrame("Ventana");  // Container (Composite)
JPanel panel = new JPanel();           // Container (Composite)
JButton boton = new JButton("Clic");   // Component (Leaf)
JLabel etiqueta = new JLabel("Texto"); // Component (Leaf)

panel.add(boton);
panel.add(etiqueta);
frame.add(panel);

// El metodo paint() se propaga recursivamente
// frame.paint() -> panel.paint() -> boton.paint(), etiqueta.paint()
```

Cuando se invoca `repaint()` en el `JFrame`, este delega en sus hijos (`JPanel`), que a su vez delegan en los suyos (`JButton`, `JLabel`). Tanto los contenedores como los componentes individuales extienden `java.awt.Component`, permitiendo tratarlos uniformemente.

> Composite es el patron natural para cualquier estructura jerarquica: sistemas de archivos, interfaces de usuario, arboles de decisiones, estructuras organizativas, documentos con secciones y subsecciones. Siempre que exista una relacion parte-todo que el cliente deba tratar de forma uniforme, Composite es la solucion.
