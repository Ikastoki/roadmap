# Principios de Diseno en Infraestructura

El diseno de infraestructura no es arbitrario: se sustenta en principios que, aplicados de forma consistente, producen sistemas mas predecibles, seguros y mantenibles. Estos principios no son teoria abstracta sino reglas practicas que determinan como se construyen, como se modifican y como se operan los servidores y servicios que componen la plataforma. Su aplicacion correcta reduce la superficie de error y facilita la automatizacion.

## Inmutabilidad

El principio de inmutabilidad establece que, una vez desplegado, un servidor o servicio no debe ser modificado manualmente en produccion. En lugar de parchear, reconfigurar o ajustar un sistema en ejecucion, se construye una nueva版本 con los cambios deseados y se reemplaza la version anterior.

### Por que la mutabilidad es problematica

Cuando un administrador accede a un servidor en produccion y modifica una configuracion, instala un paquete o ajusta un parametro, genera lo que se conoce como deriva de configuracion. El estado real del servidor se aleja gradualmente del estado documentado y esperado. Con el tiempo, el servidor acumula modificaciones ad-hoc que nadie registro, creando un estado unico, irreproducible y dificil de diagnosticar.

Este fenomeno se conoce como "configuration drift" y es una de las fuentes principales de fallos en infraestructura. Un servidor que ha sido configurado manualmente durante meses puede contener docenas de cambios que solo la persona que los realizo conoce. Si ese servidor falla, reconstruirlo exactamente igual es practicamente imposible.

### Aplicacion practica

La inmutabilidad se implementa a traves de maquinas virtuales o contenedores generados a partir de plantillas predefinidas. Cada despliegue crea una nueva instancia desde una imagen base que incluye todos los cambios necesarios. Si se necesita actualizar un servicio, se genera una nueva imagen y se sustituyen las instancias anteriores, en lugar de actualizar las existentes.

Este patron elimina la deriva de configuracion porque cada instancia parte del mismo punto conocido. Si algo falla en una instancia, se descarta y se genera otra nueva desde la plantilla, garantizando que el resultado sea identico al esperado. La capacidad de recrear entornos de forma determinista es uno de los beneficios mas valiosos de este enfoque.

### Relacion con infraestructura como codigo

La inmutabilidad es complementaria con la infraestructura como codigo. El codigo define el estado deseado, y el sistema de despliegue se encarga de materializar ese estado en instancias concretas. Ningun cambio se aplica manualmente sobre una instancia en ejecucion; todo cambio pasa por el pipeline de codigo, lo que garantiza trazabilidad, revision y reversibilidad.

## Idempotencia

Una operacion es idempotente si producir el mismo resultado independientemente de cuantas veces se ejecute. Este principio es fundamental en automatizacion de infraestructura porque permite reintentar operaciones de forma segura sin riesgo de corromper el estado del sistema.

### Ejemplo concreto

Supongamos un script que configura un servidor web. Un enfoque no idempotente verificaria si el servidor web esta instalado y, si no lo esta, lo instalaria. Si el script se ejecuta dos veces, la segunda ejecpcion verificaria que ya esta instalado y no haria nada, funcionando correctamente. Pero si el script simplemente ejecuta `apt-get install nginx` sin verificar el estado previo, la segunda ejecpcion podria comportarse de forma diferente, o incluso fallar si el paquete ya esta instalado de una version anterior.

Las herramientas modernas de configuracion como Ansible, Puppet o Chef estan disenadas para ser idempotentes por defecto. Cuando se define un estado deseado, la herramienta evalua el estado actual y aplica solo los cambios necesarios para alcanzar el estado objetivo, sin importar cuantas veces se invoque.

### Importancia en fallos

En operaciones reales, las ejecuciones fallan. Un servidor puede perder conectividad a mitad de un despliegue, un servicio puede tardar mas de lo esperado en responder o una dependencia externa puede no estar disponible. Si las operaciones no son idempotentes, reintentar despues de un fallo puede producir un estado intermedio corrupto. Con operaciones idempotentes, el reintento es seguro: el sistema simplemente reevalua el estado y aplica lo que falte.

## Minimo privilegio

El principio de minimo privilegio establece que cada usuario, proceso o servicio debe tener exclusivamente los permisos necesarios para realizar su funcion, y no mas. Un servidor web no necesita ejecutarse como root. Un servicio de bases de datos no necesita acceso al sistema de archivos del servidor web. Un usuario de desarrollo no necesita privilegios de administracion del sistema.

### Aplicacion en servidores

Cada servicio debe ejecutarse bajo un usuario dedicado con los permisos minimos necesarios. Por ejemplo, un servidor web Apache o Nginx se ejecuta normalmente como el usuario `www-data`, que solo tiene permisos de lectura sobre los archivos estaticos y escritura sobre los directorios de logs y cache. Si un atacante logra comprometer el servidor web, el dano esta limitado al alcance de los permisos de ese usuario.

En terminos de acceso administrativo, se debe utilizar un usuario no privilegiado para las tareas habituales y recurrir a sudo o mecanismos similares unicamente cuando sea estrictamente necesario. Cada uso de privilegios elevados debe quedar registrado y auditado.

### Relacion con la superficie de ataque

El minimo privilegio reduce directamente la superficie de ataque. Cada permiso adicional es un vector potencial de explotacion. Si un atacante obtiene acceso a un servicio con privilegios de root, el dano potencial es total. Si el servicio opera con privilegios minimos, el atacante debe encontrar una via de escalada de privilegios adicionales, lo que incrementa significativamente la dificultud del ataque.

## Separacion de responsabilidades

Este principio establece que los distintos componentes del sistema deben estar claramente delimitados en cuanto a su funcion y alcance. Un servidor de bases de datos no debe ejecutar servicios web. Un servidor de desarrollo no debe tener acceso directo a datos de produccion. Un balanceador de carga no debe albergar logica de negocio.

La separacion facilita la escalabilidad, la seguridad y el mantenimiento. Cuando los componentes estan claramente delimitados, un fallo en un componente no afecta automaticamente a los demas, las actualizaciones se pueden realizar por componentes sin riesgo de efectos colaterales, y los permisos de acceso se pueden gestionar de forma granular.

## Consistencia y reproducibilidad

Un sistema bien disenado produce el mismo resultado partiendo del mismo punto de partida, sin importar cuando ni donde se ejecute. Este principio implica que los entornos de desarrollo, pruebas y produccion deben ser lo mas similares posible en su configuracion base, diferenciandose unicamente en dimensiones como la escala o los datos que contienen.

La consistencia se logra a traves de plantillas compartidas, configuraciones parametrizadas y pipelines de despliegue estandarizados. Cuando un entorno de staging difiere significativamente de produccion, las pruebas en staging pierden valor porque los fallos que detectan pueden no existir en produccion, y vice versa.

