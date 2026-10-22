<h1 align="center">Formas de instalar</h1>

<h2>📑 Contenido</h2>

- [Formas de Instalar Sass](#formas-de-instalar-sass)
- [Compiladores de Escritorio](#compiladores-de-escritorio)
- [Integraciones en Entornos de Desarrollo (IDE)](#integraciones-en-entornos-de-desarrollo-ide)

## Formas de Instalar Sass

Hay varias maneras de instalar y compilar Sass, dependiendo de tu sistema operativo y tus preferencias de desarrollo. La forma más común y recomendada es a través de la línea de comandos.

1. Instalación con Node.js (la forma más popular)
   Si ya tienes Node.js instalado, puedes usar el gestor de paquetes npm para instalar Sass globalmente. Es el método más común para desarrolladores web.

2. Asegúrate de tener Node.js instalado. Puedes verificarlo abriendo tu terminal y escribiendo node -v. Si no lo tienes, puedes descargarlo desde el sitio oficial de Node.js.

3. Ejecuta el siguiente comando:

   ```bash
   npm install -g sass
   ```

   El flag -g significa "globalmente", lo que te permite usar el comando sass en cualquier proyecto de tu computadora.

## Compiladores de Escritorio

Si no te sientes cómodo usando la línea de comandos, existen aplicaciones de terceros con interfaces gráficas (GUI) que compilan Sass por ti.

Algunas opciones populares son:

- Koala: Un compilador de Sass de código abierto que funciona en Windows, macOS y Linux. Solo tienes que arrastrar y soltar tu carpeta de proyecto.

- Prepros: Una aplicación más completa que no solo compila Sass, sino que también ofrece otras herramientas como minificación de CSS y recarga en vivo del navegador.

Estas herramientas son muy útiles para principiantes, ya que automatizan el proceso de compilación sin necesidad de comandos.

## Integraciones en Entornos de Desarrollo (IDE)

Muchos editores de código modernos, como Visual Studio Code, tienen extensiones que facilitan la compilación de Sass.

- Live Sass Compiler (para VS Code): Esta es una de las extensiones más populares para VS Code. Una vez instalada, añade un botón en la barra de estado llamado "Watch Sass". Cuando lo activas, compila automáticamente tus archivos `.scss` o `.sass` a `.css` cada vez que guardas.
