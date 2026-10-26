# Build Automatizado

Un proyecto de software vive en un estado de incertidumbre mientras su codigo fuente no puede transformarse en algo ejecutable. El build automatizado elimina esa incertidumbre al convertir el codigo fuente en artefactos de forma determinista, sin intervencion manual, cada vez que se detecta un cambio. Un solo desarrollador que compila en su maquina local y obtiene un resultado distinto al de su collega tiene un problema de reproducibilidad que escalara rapidamente en un equipo. El build automatizado no es simplemente "compilar antes de desplegar": es el contrato que garantiza que el mismo commit siempre produce el mismo artefacto, en cualquier entorno, en cualquier momento.

La compilacion del codigo es solo la primera fase de un pipeline de build. Dependiendo del lenguaje y la plataforma, el proceso puede incluir la resolucion de dependencias externas, la generacion de codigo fuente a partir de plantillas o esquemas (code generation), el empaquetado de recursos estaticos y la generacion del artefacto final. Cada uno de estos pasos debe ser declarativo y versionado: si un archivo `pom.xml`, `package.json` o `go.mod` cambia de version, el build debe reflejar ese cambio automaticamente. El objetivo es que el proceso completo quede descrito en codigo (infrastructure as code) y no en la memoria de nadie.

## Resolucion de dependencias

Todo proyecto moderno depende de bibliotecas de terceros. Un build correcto debe ser capaz de instalar exactamente las versiones especificadas, sin importar el sistema operativo o la maquina donde se ejecute. Herramientas como Maven, Gradle, npm o Go modules resuelven este problema descargando artefactos desde repositorios centralizados y verificando su integridad mediante checksums. Si una dependencia no esta disponible o su hash no coincide, el build debe fallar explicitamente.

> Un build que funciona "a veces" porque descarga distintas versiones de una libreria no es un build confiable. El lockfile (`package-lock.json`, `go.sum`, `Pipfile.lock`) es el mecanismo que garantiza la exactitud en las dependencias transitivas.

La gestion de dependencias no se limita a bibliotecas de codigo. Un build moderno tambien puede depender de herramientas de compilacion (compiladores, transpiladores), imagenes base de Docker o incluso versiones especificas de runtime como Node.js o JDK. Herramientas como `sdkman`, `nvm` o contenedores con versiones fijas permiten encapsular estas dependencias para que el build sea verdaderamente portable.

## Generacion de artefactos

El resultado concreto de un build es un artefacto: un JAR en proyectos Java, una imagen Docker, un binario compilado en Go o Rust, un paquete npm o un ejecutable con sus librerias vinculadas. El artefacto es la unidad de despliegue, no el codigo fuente. Esta distincion es fundamental: en produccion nunca se compila el codigo, se ejecuta el artefacto que el build produjo.

El proceso de generacion varia significativamente entre ecosistemas. En Java, el build produce un JAR o WAR que contiene clases compiladas, metadatos y dependencias. En un proyecto con Docker, el artefacto final es una imagen que encapsula el binario junto con su sistema de archivos, runtime y configuracion. En sistemas embebidos, el artefacto puede ser un firmware binario con un layout de memoria especifico. Independientemente del formato, el build debe producir un artefacto que sea una capsula autocontenida, verificable y listo para ejecutar.

```dockerfile
# Ejemplo simplificado de un build que produce una imagen Docker
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/server

FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
```

Este patron multi-stage garantiza que el artefacto final solo contenga lo necesario para ejecutar la aplicacion, reduciendo la superficie de ataque y el tamano de la imagen.

## Versionado de artefactos

Cada artefacto producido por un build debe tener una identidad unica y rastreable. Esta identidad generalmente combina el nombre del proyecto, el numero de version y un identificador unico del commit (como un SHA corto de Git). El esquema de versionado mas comun sigue el patron `MAJOR.MINOR.PATCH` (Semantic Versioning), aunque en practica muchos equipos usan el numero de build o el hash del commit como sufijo.

La trazabilidad es el objetivo real del versionado. Si un artefacto desplegado en produccion presenta un bug, el equipo debe poder rastrear exactamente que commit de codigo fuente lo genero, que dependencias incluia y que maquina ejecuto el build. Herramientas como JFrog Artifactory, Nexus o GitHub Packages actuan como repositorios de artefactos, almacenando cada build con sus metadatos completos. Algunos proyectos implementan SBOMs (Software Bill of Materials) para documentar cada componente incluido en el artefacto.

> El versionado no es solo un numero bonito. Un artefacto sin version trazable es una caja negra: si falla en produccion, no hay forma de determinar que contiene ni como se genero.

## Build reproducible

La reproducibilidad es la propiedad mas critica de un build automatizado. Un build reproducible significa que, dados los mismos inputs (codigo fuente, dependencias, herramientas), el output (el artefacto) es bit-a-bit identico en cualquier maquina y en cualquier momento. Esto no es trivial: compiladores pueden incluir timestamps, generadores de numeros aleatorios pueden crear diferencias, y el orden de procesamiento de archivos puede variar.

Los beneficios de la reproducibilidad van mas alla de la conveniencia. Un build no reproducible invalida toda la cadena de confianza del pipeline de CI/CD: si el artefacto generado en el entorno de pruebas no es identico al generado en produccion, las pruebas automaticas pierden sentido. Ademas, la reproducibilidad es un requisito normativo en sectores regulados como finanzas o salud, donde se debe demostrar que un artefacto desplegado corresponde exactamente al codigo fuente auditado.

Para alcanzar la reproducibilidad, los equipos adoptan practicas como:

- Fijar versiones exactas de todas las herramientas de build (compilador, runtime, herramientas auxiliares)
- Usar lockfiles para dependencias y verificar su integridad
- Ejecutar el build dentro de contenedores con versiones fijas
- Eliminar fuentes de no-determinismo como timestamps o rutas absolutas en los artefactos
- Validar periodicamente la reproducibilidad ejecutando el mismo build en maquinas distintas

Un build que no es reproducible es un build que no es confiable, y un build que no es confiable desactiva la integracion continua. La automatizacion sin reproducibilidad solo agiliza la produccion de errores inconsistentes.

