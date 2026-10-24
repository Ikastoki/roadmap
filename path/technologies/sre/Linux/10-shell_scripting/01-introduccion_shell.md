# Introduccion al shell

Un interprete de comandos, o shell, es un programa que actua como intermediario entre el usuario y el kernel del sistema operativo. Su funcion principal es leer comandos escritos por el usuario, interpretarlos y traducirlos en llamadas al sistema que el kernel pueda ejecutar. No se trata de un simple traductor: el shell moderno es un entorno de programacion completo con variables, estructuras de control, funciones y capacidades de automatizacion que lo convierten en la herramienta central de cualquier administrador de sistemas o desarrollador que trabaje con Unix/Linux.

La existencia de multiples shells en el ecosistema Linux responde a diferentes filosofias de diseno. Los mas comunes son:

- **Bash** (Bourne Again SHell): el estandar de facto en la mayoria de distribuciones Linux. Combina la compatibilidad hacia atras con el shell Bourne original (sh) con caracteristicas avanzadas tomadas de Korn shell (ksh) y C shell (csh). Es el shell por defecto en sistemas GNU/Linux, macOS (hasta Catalina), y Windows Subsystem for Linux (WSL).
- **Zsh** (Z shell): un shell moderno que extiende Bash con mejoras en autocompletado, correccion ortografica, personalizacion de prompts y un potente sistema de plugins. Es el shell predeterminado en macOS desde Catalina.
- **Fish** (Friendly Interactive SHell): disenado desde cero para ser intuitivo y facil de usar, con autosugerencias, resaltado de sintaxis y configuracion basada en web. No mantiene compatibilidad con POSIX, lo que limita su uso en scripts portables.
- **Dash** (Debian Almquist SHell): un shell minimalista y extremadamente rapido, utilizado como `/bin/sh` en Debian y derivados para ejecutar scripts de inicio del sistema. Carece de las comodidades interactivas de Bash pero ejecuta scripts mas rapidamente.

Bash se ha convertido en el estandar no solo por su ubicuidad sino por su equilibrio entre potencia y compatibilidad. Un script escrito en Bash puro (sin caracteristicas especificas de Bash) funcionara en cualquier shell POSIX; si se usan arrays asociativos, expansiones aritmeticas con `(( ))` o las variables `$RANDOM` y `$SECONDS`, se dependera de extensiones de Bash que otros shells no implementan.

```bash
echo "Hola desde $SHELL"
whoami
pwd
date
```

La terminal es el programa que proporciona la interfaz grafica (ventana) donde se ejecuta el shell. La distincion es sutil pero importante: la terminal muestra el prompt, captura el teclado y gestiona el buffer de pantalla; el shell interpreta los comandos. Los emuladores de terminal modernos (GNOME Terminal, Konsole, Alacritty, kitty) simulan el comportamiento de los terminales fisicos de los anos 70.

> Históricamente, los terminales físicos eran dispositivos de hardware (VT100, xterm) conectados por puerto serie a mainframes. El shell residía en el mainframe; el terminal era solo el dispositivo de E/S. Hoy ambos conceptos coexisten como software en una misma máquina.

Los primeros comandos que se encuentran al abrir una terminal son `ls` (listar archivos), `cd` (cambiar directorio), `pwd` (mostrar directorio actual), `man` (acceder al manual) y `echo` (imprimir texto). Dominar estos comandos basicos y entender que todo en Unix es un archivo constituye la base sobre la que se construye el conocimiento mas avanzado de shell scripting.

