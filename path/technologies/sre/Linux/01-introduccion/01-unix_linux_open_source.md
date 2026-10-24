# Historia de Unix, Linux y el open source

La computacion moderna descansa sobre una cadena de decisiones tecnicas y filosoficas que arrancan en los laboratorios de los anos sesenta. Cada capa del sistema operativo que usamos hoy -- desde la jerarquia de archivos hasta los permisos, pasando por la separacion entre kernel y espacio de usuario -- tiene una historia que explica por que las cosas son como son. Sin ese trasfondo, la tecnologia parece arbitraria; con el, se revela como el resultado de decadas de experiencia colectiva.

## Multics y AT&T Bell Labs

En 1965, el MIT, General Electric y los laboratorios Bell de AT&T se embarcaron en el proyecto Multics (Multiplexed Information and Computing Service). El objetivo era construir un sistema operativo de tiempo compartido que permitiera a multiples usuarios trabajar simultaneamente en una misma maquina, una idea revolucionaria cuando el modelo predominante era el proceso batch. Multics introdujo conceptos que hoy damos por sentados: el sistema de archivos jerarquico, la segmentacion de memoria, el anillo de proteccion de seguridad y el propio lenguaje de programacion C.

Multics resulto demasiado ambicioso. Los plazos se alargaron, el rendimiento no era el esperado y AT&T, que pagaba la factura, decidio retirarse del proyecto en 1969. Ken Thompson, Dennis Ritchie y otros investigadores de Bell Labs se quedaron sin un sistema operativo moderno sobre el que trabajar. Esta frustracion fue el germen de Unix.

## El nacimiento de Unix y la filosofia "hacer una cosa bien"

Thompson, Ritchie, Brian Kernighan y sus colegas empezaron a escribir un sistema operativo mas modesto en un PDP-7. El nombre Unix es un juego de palabras con Multics: "Unics" (Uniplexed Information and Computing Service), que luego derivo en Unix. La premisa era radicalmente opuesta a la grandiosidad de Multics: construir un sistema pequeno, modular y eficiente que hiciera bien lo que tenia que hacer.

La filosofia Unix se cristalizo en varias maximas:
- Haz una sola cosa y hazla bien.
- Compone programas pequenos mediante tuberias (pipes) para resolver tareas complejas.
- Todo es un archivo.
- La configuracion se expresa en texto plano.
- El silencio del programa es exito.

En 1973, Ritchie y Thompson reescribieron Unix en C, un lenguaje de alto nivel que ellos mismos habian creado. Esto fue un punto de inflexion: por primera vez, un sistema operativo podia portarse a diferentes arquitecturas con solo recompilar el codigo. Unix dejo de estar atado a un hardware concreto y comenzo a proliferar en universidades y centros de investigacion.

## La era del software libre y el movimiento GNU

En 1983, Richard Stallman, investigador del MIT, anuncio el proyecto GNU (GNU's Not Unix). Stallman habia trabajado en entornos donde el codigo fuente se compartia libremente, pero la industria del software empezaba a cerrar sus productos. El sistema operativo Unix se estaba fragmentando en versiones propietarias (System V de AT&T, BSD de Berkeley), y cada una imponia restricciones de licencia.

Stallman concibio GNU como un sistema operativo completo compatible con Unix pero completamente libre. Fundo la Free Software Foundation (FSF) en 1985 y definio el concepto de software libre: la libertad de ejecutar, estudiar, modificar y redistribuir el software. Para garantizar estas libertades, creo la licencia GPL (General Public License), que utiliza el mecanismo del copyright para asegurar que el software permanezca libre: cualquiera puede modificar y distribuir el codigo, pero las modificaciones deben distribuirse bajo la misma licencia.

La GPL introdujo el concepto de copyleft, una estrategia legal que usa los derechos de autor para garantizar que las libertades se preserven en todas las copias y derivados del programa.

GNU desarrollo herramientas esenciales: el compilador GCC, el editor Emacs, el depurador GDB, el interprete de comandos Bash, las utilidades del nucleo (coreutils) y la biblioteca C glibc. Lo unico que faltaba era el kernel.

## Minix y la creacion del kernel Linux

En 1987, Andrew Tanenbaum, profesor de la Universidad Libre de Amsterdam, creo Minix como sistema operativo educativo para su libro "Operating Systems: Design and Implementation". Minix era funcional pero tenia limitaciones intencionadas: su licencia lo restringia al uso academico, y su diseno monolitico estaba pensado para ensenanza, no para produccion.

El 25 de agosto de 1991, un estudiante finlandes de 21 anos llamado Linus Torvalds publico un mensaje en el grupo de noticias comp.os.minix:

> "Estoy haciendo un sistema operativo (gratuito, solo un hobby, no va a ser grande y profesional como GNU) para clones AT 386(486)."

Torvalds habia comenzado su proyecto como un emulador de terminal para acceder a los servidores Unix de la universidad desde su ordenador personal. El kernel que habia escrito funcionaba sobre el hardware x86 y utilizaba las herramientas de GNU para ser util. La combinacion de ambos produjo un sistema operativo completo. Por eso se discute si el nombre correcto es "Linux" o "GNU/Linux": tecnicamente, Linux es el kernel, y el sistema operativo es la integracion del kernel Linux con las herramientas GNU.

El kernel Linux se publico bajo la GPL en 1992, lo que permitio que cualquiera pudiera estudiarlo, modificarlo y mejorarlo. La decision de liberar el codigo fue probablemente la mas importante de la historia del software: miles de desarrolladores comenzaron a contribuir, y Linux paso de ser un proyecto personal a un sistema operativo que hoy ejecuta desde telefonos Android hasta el 90% de los servidores web, pasando por la mayoria de los superordenadores del mundo.

## Codigo abierto frente a software libre

En 1998, un grupo de desarrolladores liderados por Eric Raymond y Bruce Perens propusieron el termino "open source" como una alternativa mas pragmatica a "free software". El problema era que "free" en ingles se interpretaba como "gratuito" y resultaba ambiguo en contextos comerciales. La Open Source Initiative (OSI) nacio con el objetivo de hacer que el modelo de desarrollo abierto fuera aceptable para las empresas.

La diferencia entre ambos movimientos es filosofica. El software libre se centra en la etica y los derechos del usuario: la libertad como valor moral. El codigo abierto se centra en las ventajas practicas del desarrollo colaborativo: calidad, transparencia, seguridad y rapidez de innovacion. En la practica, la mayoria de las licencias de codigo abierto (Apache, MIT, BSD) son tambien licencias de software libre, y viceversa. La GPL es ambas cosas, pero la tension entre las dos visiones persiste.

Las licencias mas relevantes en el ecosistema actual son:
- **GPL**: exige que cualquier obra derivada se distribuya bajo la misma licencia (copyleft fuerte).
- **LGPL**: variante de la GPL que permite enlazar con software propietario sin contagiar la licencia.
- **MIT y BSD**: permisivas, permiten usar el codigo en software propietario sin restricciones.
- **Apache 2.0**: permisiva, pero incluye clausulas de patentes y atribucion.

> Linux es el ejemplo mas exitoso del modelo de desarrollo abierto, pero no el unico. El kernel, con mas de 28 millones de lineas de codigo y miles de contribuyentes por version, demuestra que el desarrollo descentralizado puede producir software de calidad industrial.

