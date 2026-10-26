# Logs y alertas

Los registros del sistema son la memoria historica de un servidor. Cada peticion HTTP, cada inicio de sesion, cada error de disco queda plasmado en algun archivo de log. Cuando un servidor funciona correctamente, los logs son simplemente evidencia de operacion normal. Cuando algo falla, se convierten en la primera herramienta de diagnostico. Pero los logs solo son utiles si se gestionan correctamente: un servidor sin rotacion de logs puede llenar el disco con gigabytes de texto antiguo, y un sistema de alertas sin criterio puede ahogar al administrador en notificaciones irrelevantes.

## Gestion de logs con logrotate

El problema fundamental de los logs es que crecen sin limite. Un servidor web busy puede generar gigabytes de registros diarios. Sin un mecanismo de rotacion, el archivo de log crece hasta agotar el espacio en disco, lo que puede provocar fallos en servicios que dependen de escribir registros o, peor aun, en servicios que necesitan espacio de disco para operar. `logrotate` resuelve este problema rotando, comprimiendo y eliminando logs segun una politica configurable.

`logrotate` funciona como un cron job que se ejecuta diariamente (o con la frecuencia configurada). Para cada archivo gestionado, define que hacer: rotar el archivo, comprimir las versiones anteriores, mantener un numero determinado de copias y ejecutar un comando post-rotacion (generalmente enviar una senal al servicio para que reabra el archivo de log). La configuracion se encuentra en `/etc/logrotate.conf` y en `/etc/logrotate.d/`, donde cada servicio puede tener su propio archivo de configuracion.

```bash
# Configuracion basica de logrotate para un servicio web
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        /bin/kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

> La rotacion de logs no es opcional: es un requisito de operacion. Un servidor con logs sin rotacion es una bomba de tiempo que eventualmente agotara el espacio en disco y provocara un fallo en cascada.

Los parametros clave de logrotate incluyen `daily`/`weekly`/`monthly` para la frecuencia de rotacion, `rotate N` para el numero de copias a mantener, `compress` para comprimir versiones anteriores con gzip, y `size` para rotar basado en tamano en lugar de tiempo. El parametro `copytruncate` es particularmente util para servicios que no pueden reabrir sus archivos de log: en lugar de mover el archivo y crear uno nuevo, crea una copia y trunca el original, evitando la necesidad de una senal al proceso.

## Sistema de alertas

El monitoreo sin alertas es solo un dashboard bonito. Las alertas transforman la informacion en accion: notifican al administrador cuando algo sale del rango esperado y requiere atencion. Un sistema de alertas efectivo no solo detecta problemas, sino que los comunica con el contexto suficiente para que la respuesta sea rapida y adecuada.

Los canales de notificacion mas comunes son el correo electronico y plataformas de mensajeria como Slack o Discord. El correo electronico es universal pero lento y propenso a ser ignorado en buzones saturados. Slack o Discord permiten crear canales dedicados donde las alertas llegan con contexto inmediato y donde el equipo puede discutir la respuesta en tiempo real. La eleccion del canal depende del flujo de trabajo del equipo: lo critico es que la alerta llegue donde realmente se mira.

Los umbrales de alerta deben basarse en el conocimiento del comportamiento normal del sistema. Una alerta de "CPU al 90%" puede ser perfectamente normal en un servidor de procesamiento por lotes durante la ejecucion de un job, pero critica en un servidor de bases de datos en horario pico. Los mejores sistemas de monitoreo permiten definir umbrales por periodo del dia o por estado del servicio, evitando falsos positivos que generan desconfianza en el sistema.

## Fatiga de alertas

Uno de los fenomenos mas destructivos en operaciones es la fatiga de alertas: cuando un sistema genera demasiadas notificaciones, el administrador aprende a ignorarlas. Esto ocurre gradualmente: cada falso positivo erosiona la confianza en el sistema, hasta que una alerta genuina se pierde entre el ruido. Es el equivalente operativo del cuento del lobo: si el sistema grita "fuego" cada vez que la temperatura sube un grado, nadie acudira cuando haya un incendio real.

Prevenir la fatiga de alertas requiere criterio en la definicion de umbrales, clasificacion por severidad y agrupacion de eventos relacionados. Una buena estrategia es definir tres niveles: informativo (algo ha cambiado pero no requiere accion inmediata), warning (se acerca a un umbral problematico) y critico (requiere intervencion inmediata). Solo el nivel critico debe generar notificaciones intrusivas como SMS o llamadas; los demas pueden esperar revision periodica. Ademas, las alertas deben incluir contexto: no solo "disco al 95%", sino "disco /var al 95%, crecimiento promedio 2% diario, espacio agotado en 3 dias".

La revision periodica de la efectividad de las alertas es parte esencial del monitoreo. Alguna alerta nunca se ha disparado? Quizas el umbral es demasiado alto. Alguna alerta se dispara diariamente sin que haya una accion correspondiente? Quizas no es una alerta sino un reporte programado. El objetivo no es tener menos alertas, sino tener las alertas correctas.

