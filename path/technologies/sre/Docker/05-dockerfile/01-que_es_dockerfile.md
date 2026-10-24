# Que es un Dockerfile

Un Dockerfile es un archivo de texto sin extension que contiene una secuencia de instrucciones que Docker interpreta para construir automaticamente una imagen. Cada instruccion del Dockerfile genera una nueva capa en el sistema de archivos de la imagen, y esas capas se apilan siguiendo el orden de las instrucciones. Cuando se ejecuta `docker build`, el motor de Docker lee el Dockerfile linea por linea, ejecuta cada instruccion en un contenedor temporal y, al finalizar, persiste el resultado como una nueva capa. El resultado final es una imagen lista para ejecutar contenedores, con todo lo necesario: sistema base, dependencias, codigo fuente, variables de entorno y punto de entrada.

## Capas: el mecanismo fundamental

Cada instruccion en el Dockerfile produce una capa de solo lectura. Docker utiliza un sistema de archivos por capas (normalmente OverlayFS) que apila estas capas una sobre otra para formar el sistema de archivos unificado de la imagen. Cuando una instruccion se ejecuta, Docker calcula un hash criptografico del contenido resultante de esa capa. Si en una reconstruccion posterior el hash coincide con una capa previamente almacenada en la cache local, Docker reutiliza esa capa en lugar de ejecutar la instruccion de nuevo.

```dockerfile
FROM python:3.11-slim      # capa 1: imagen base
RUN pip install flask      # capa 2: dependencias
COPY . /app                # capa 3: codigo fuente
CMD ["python", "app.py"]   # capa 4: metadatos
```

Cada capa contiene unicamente los cambios con respecto a la capa anterior. Esto hace que las imagenes sean modulares y eficientes: si dos imagenes comparten las mismas capas iniciales (por ejemplo, la misma imagen base), Docker reutiliza el almacenamiento en disco y acelera las descargas.

> Las capas son de solo lectura. Cuando un contenedor se inicia, Docker anade una fina capa de escritura (container layer) sobre las capas de la imagen. Todos los cambios que ocurren dentro del contenedor se escriben ahi y se pierden al eliminar el contenedor a menos que se use un volumen.

## El contexto de build

El ultimo argumento de `docker build` es el contexto de build, normalmente representado por un punto (`.`). El contexto es el directorio desde el cual Docker empaqueta y envia todos los archivos al demonio para que esten disponibles durante la construccion. El demonio recibe ese directorio comprimido, lo extrae en un directorio temporal y ahi ejecuta las instrucciones del Dockerfile.

```bash
docker build -t mi-app:1.0 .
```

Es fundamental entender que solo los archivos dentro del contexto son accesibles desde el Dockerfile. No se puede hacer referencia a archivos fuera del contexto, ni siquiera con rutas absolutas. Por eso es practica comun situar el Dockerfile en la raiz del proyecto o en un subdirectorio especificando el contexto adecuadamente.

```bash
# Contexto en el directorio actual, Dockerfile en un subdirectorio
docker build -t mi-app:1.0 -f docker/Dockerfile .
```

> El tamanio del contexto influye directamente en la velocidad de build. Cuantos mas archivos se incluyan, mas tardara el demonio en recibirlos y procesarlos. Por eso se usa `.dockerignore` para excluir archivos innecesarios (node_modules, cache, .git, etc.).

## Proceso de construccion paso a paso

Cuando se ejecuta `docker build`, el flujo es el siguiente:

1. El cliente Docker empaqueta el contexto en un archivo `.tar` y lo envia al demonio.
2. El demonio recibe el contexto, lo extrae en un directorio temporal y lee el Dockerfile.
3. Para cada instruccion, el demonio verifica si existe una capa en cache que coincida.
4. Si no hay cache, crea un contenedor temporal a partir de la imagen intermedia actual, ejecuta la instruccion, commitea el resultado como una nueva imagen y la etiqueta con un hash.
5. Al finalizar, la imagen queda registrada localmente con el tag especificado.

```bash
# Ver las capas de cualquier imagen local
docker history mi-app:1.0
```

La salida de `docker history` muestra cada capa con su tamano y el comando que la genero. Es una herramienta de diagnostico valiosa para entender la composicion de una imagen y detectar capas innecesariamente grandes.

## Dockerfile y la estandarizacion OCI

El formato Dockerfile es la forma mas comun de describir una imagen, pero no es la unica. La Open Container Initiative (OCI) define un estandar para imagenes de contenedores que Docker sigue. Cualquier herramienta que cumpla con el estandar OCI (podman, buildah, kaniko) puede procesar un Dockerfile y producir una imagen compatible, lo que garantiza portabilidad entre ecosistemas.

