# Monitoreo basico de servidores

Un servidor que no se monitorea es un servidor que fallara sin previo aviso. El monitoreo basico consiste en observar periodicamente los recursos fundamentales del sistema —CPU, memoria, disco y red— para establecer una linea base de comportamiento normal. Solo cuando se comprende que significa "normal" para un servidor concreto es posible detectar anomalias: un pico inesperado de uso de CPU, un proceso que consume memoria de forma desmedida o un disco que se llena gradualmente. Sin esa referencia, cada alerta es una suposicion y cada incidencia una sorpresa.

## Uso de CPU, memoria y disco

El uso de CPU no es simplemente un porcentaje: refleja la relacion entre tiempo de procesamiento util y tiempo de espera. Un valor alto puede indicar carga real, pero tambien problemas de bloqueo de E/O, context switching excesivo o procesos zombie. La herramienta `htop` ofrece una vision mucho mas rica que el clasico `top`: muestra arboles de procesos, uso por nucleo, tiempo de ejecucion y permite interactuar directamente con los procesos identificados. Diferenciar entre `%user`, `%system` y `%iowait` es clave para diagnosticar correctamente.

La memoria en Linux se gestiona con una filosofia contraintuitiva: la memoria "libre" es memoria desperdiciada, por lo que el kernel la usa activamente para cache. Para entender realmente el consumo, hay que observar `available` en vez de `free`. Herramientas como `htop` muestran la barrera entre buffers/cache y el uso real, lo que permite distinguir entre un sistema con poca memoria disponible y uno que simplemente esta aprovechando el cache de forma eficiente.

El disco es el recurso que con mas frecuencia genera incidentes silenciosos. `df -h` muestra el espacio disponible por particion, pero el problema real muchas veces no es el espacio total sino los inodos: un servidor con millones de archivos pequenos puede agotar inodos teniendo espacio libre en bloques. `iotop` complementa esta vision mostrando que procesos estan realizando operaciones de E/O y con que intensidad, lo que permite identificar un proceso que satur el disco sin necesidad de un agente externo.

```bash
# Estado general de CPU y procesos
htop

# Espacio en disco y uso de inodos
df -h
df -i

# Procesos con mayor actividad de E/O
sudo iotop -o
```

## Conexiones de red

El estado de la red de un servidor se refleja en las conexiones activas, los puertos que escuchan y el trafico que fluye por cada interfaz. `ss` ha reemplazado al clasico `netstat` y ofrece informacion mas detallada y rapida sobre sockets activos. Desde una perspectiva de monitoreo, saber que puertos estan abiertos y que procesos los utilizan es esencial tanto para diagnostico como para seguridad. Un numero inusual de conexiones en estado `TIME_WAIT` puede indicar un problema de configuracion o un ataque.

`iftop` proporciona una vista en tiempo real del trafico por interfaz de red, mostrando las conexiones mas activas y su ancho de banda consumido. A diferencia de herramientas como `nethogs` (que agrupa por proceso), `iftop` agrupa por conexion, lo que resulta mas util para identificar patrones de trafico anormales o conexiones no autorizadas.

```bash
# Conexiones activas y puertos en escucha
ss -tunlp

# Trafico de red en tiempo real por interfaz
sudo iftop -i eth0
```

> En servidores con multiples interfaces, es importante monitorear cada una por separado. Una interfaz de management con trafico inesperado puede indicar un intento de acceso no autorizado.

## Establecer una linea base

El verdadero proposito del monitoreo basico no es ejecutar comandos periodicamente, sino construir un modelo mental de como se comporta el servidor bajo condiciones normales. Un servidor web en hora punta tiene un patron de uso completamente diferente al mismo servidor a las tres de la madrugada. Sin conocer ambos, es imposible determinar si un 80% de uso de CPU a las dos de la manana es un problema o simplemente un proceso de backup programado.

Esto implica registrar metricas a lo largo del tiempo, no solo observar el instante presente. Un valor aislado nunca es suficiente: el contexto temporal es lo que transforma un dato en informacion util. Un disco al 70% de uso no es alarmante si el historial muestra un crecimiento estable del 2% mensual, pero es critico si ese 70% se alcanzo en un dia. El monitoreo basico es, en esencia, la disciplina de mirar el sistema con suficiente frecuencia y contexto como para notar cambios significativos antes de que se conviertan en fallos.

