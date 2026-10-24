# Spring Boot CLI

Spring Boot CLI (Command Line Interface) es una herramienta de línea de comandos que permite ejecutar y desarrollar aplicaciones Spring Boot rápidamente sin configuración completa de proyecto. Proporciona una forma de prototipado rápido mediante scripts Groovy, eliminando la necesidad de archivos de configuración de proyecto para pruebas rápidas.

La instalación de Spring Boot CLI se puede realizar mediante多种 métodos. La forma más simple es descargar el archivo de distribución desde el sitio oficial de Spring y descomprimirlo en un directorio. Alternativamente, se puede instalar mediante gestores de paquetes como SDKMAN, Homebrew o GVM (Groovy enVM Manager), lo que facilita la gestión de versiones y actualizaciones.

```bash
# Instalación mediante SDKMAN
sdk install springboot

# Verificar instalación
spring --version

# Instalar usando Homebrew (macOS)
brew tap spring-io/tap
brew install springboot
```

Los scripts Groovy son el corazón de Spring Boot CLI. Permiten definir aplicaciones Spring completas en archivos Groovy con una sintaxis concisa. Spring Boot CLI detecta automáticamente las anotaciones y dependencias necesarias, agregando las importaciones requeridas implícitamente. Esto permite crear aplicaciones web con solo unas pocas líneas de código.

```groovy
// ejemplo.groovy
@RestController
class EjemploController {
    
    @GetMapping("/")
    String inicio() {
        return "Hola Mundo"
    }
    
    @GetMapping("/usuarios/{id}")
    Usuario obtenerUsuario(@PathVariable Long id) {
        new Usuario(id, "Juan", "juan@ejemplo.com")
    }
}

class Usuario {
    Long id
    String nombre
    String email
    
    Usuario(Long id, String nombre, String email) {
        this.id = id
        this.nombre = nombre
        this.email = email
    }
}
```

Para ejecutar una aplicación Groovy, se usa el comando `spring run` seguido del nombre del archivo. Spring Boot CLI compila automáticamente el script Groovy, resuelve las dependencias necesarias y ejecuta la aplicación. El servidor web embebido se inicia automáticamente si se detectan anotaciones de controller.

```bash
# Ejecutar script Groovy
spring run ejemplo.groovy

# Ejecutar con argumentos adicionales
spring run ejemplo.groovy -- --server.port=9090

# Ejecutar con perfil específico
spring run ejemplo.groovy -- --spring.profiles.active=dev
```

Spring Boot CLI incluye comandos útiles para desarrollo. El comando `spring init` genera proyectos Spring Boot desde la línea de comandos, similar a lo que hace Spring Initializr pero sin necesidad de navegador web. Esto es particularmente útil para automatización de scripts y generación rápida de proyectos.

```bash
# Generar proyecto con dependencias específicas
spring init --dependencies=web,data-jpa,devtools mi-proyecto

# Generar proyecto con formato específico
spring init --type=maven-project --java-version=17 mi-proyecto

# Generar proyecto con Kotlin
spring init --language=kotlin mi-proyecto
```

El comando `spring grab` descarga las dependencias necesarias para un script Groovy sin ejecutarlo. Esto es útil para preparar un entorno de ejecución o para depurar problemas de resolución de dependencias. El comando crea un directorio `.grab` con todas las bibliotecas necesarias.

```bash
# Descargar dependencias sin ejecutar
spring grab ejemplo.groovy

# Ejecutar con dependencias pre-cargadas
spring run ejemplo.groovy
```

Spring Boot CLI proporciona un REPL (Read-Eval-Print Loop) interactivo para experimentar con código Groovy y APIs de Spring. El REPL permite ejecutar fragmentos de código, probar configuraciones y explorar funcionalidades sin crear archivos completos. Es una herramienta excelente para aprendizaje y prototipado rápido.

```bash
# Iniciar REPL
spring

# En el REPL
spring> @RestController
spring> class TestController {
spring>     @GetMapping("/test")
spring>     String test() { "Funciona!" }
spring> }
spring> :run
```

> Spring Boot CLI es una herramienta de desarrollo y prototipado, no recomendada para producción. Las aplicaciones creadas con CLI son ideales para pruebas rápidas, demos y experimentación, pero para aplicaciones de producción se deben usar proyectos completos con gestión de dependencias adecuada.

