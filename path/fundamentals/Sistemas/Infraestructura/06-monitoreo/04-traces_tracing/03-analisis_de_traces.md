# Analisis de traces

拥有一套 instrumentada y propagando contexto correctamente genera una cantidad masiva de trazas. Pero los datos crudos de trazas sin analisis son como un almacén lleno de cartas postal ordenadas: contienen toda la informacion necesaria, pero sin un proceso de lectura y sintesis son inutilizables. El analisis de trazas es la disciplina de extraer conocimiento accionable a partir de los patrones que emergen del examen de multiples trazas, identificando cuellos de botella, dependencias criticas y oportunidades de optimizacion que serian invisibles en metricas agregadas o logs individuales.

## Analisis de latencia

El analisis de latencia en tracing distribuido consiste en determinar cuanto tiempo consume cada servicio en el procesamiento de una peticion y, de forma mas especifica, que componente del sistema es responsable de la mayor parte del tiempo total de respuesta. A diferencia de las metricas de latencia por servicio (que muestran promedios o percentiles de cada servicio de forma aislada), el tracing permite descomponer la latencia total de una peticion en sus componentes individuales y determinar la contribucion de cada servicio al tiempo total.

Considera una peticion que tarda 800ms en total. Las metricas por servicio podrian mostrar que cada uno de los cinco servicios involucrados tiene una latencia promedio de 40ms. Esto pareceria saludable. Pero el tracing revela que la peticion paso por una secuencia de llamadas donde el tiempo se acumulo de forma secuencial: 40ms + 30ms + 50ms + 60ms + 20ms = 200ms de latencia nativa, pero la latencia total es 800ms porque hubo 600ms adicionales de waiting time entre servicios (latencia de red, cold starts, colas de mensajes). El tracing hace visible este tiempo oculto que las metricas por servicio completamente ocultan.

La forma mas directa de analizar latencia es ordenar los spans por duracion dentro de una traza y examinar los que consumen mas tiempo. Los backends de tracing como Jaeger o Grafana Tempo permiten buscar trazas cuya duracion total supere un umbral y agruparlas por servicio o operacion para identificar que componentes son consistentemente lentos.

> La latencia observada en un span no siempre refleja el tiempo de procesamiento real del servicio. Un span de 500ms puede incluir 400ms de espera por una respuesta de un servicio downstream. El analisis correcto requiere examinar la jerarquia de spans para distinguir entre tiempo de procesamiento propio y tiempo de espera en llamadas dependientes.

## Deteccion de cuellos de botella

Un cuello de botella en un sistema distribuido es un servicio o recurso que limita el throughput global del sistema porque su capacidad de procesamiento es insuficiente para la demanda recibida. El tracing distribuido es la herramienta mas directa para identificar estos cuellos porque muestra exactamente donde se acumula el tiempo en cada peticion.

Existen dos tipos fundamentales de cuellos de botella que el tracing puede revelar. El primero es el cuello de botella por latencia: un servicio que responde consistentemente mas lento que los demas, provocando que todas las peticiones que lo atraviesan experimenten una degradacion proporcional. El segundo es el cuello de botella por concurrencia: un servicio que opera correctamente en condiciones normales pero que bajo carga comienza a acumular peticiones encoladas, creando un efecto cascada que se propaga hacia los servicios upstream.

La forma mas efectiva de detectar cuellos de botella es buscar patrones temporales en las trazas. Si una operacion que normalmente toma 50ms periodicamente salta a 2.000ms, el tracing permite inspeccionar las trazas lentas para identificar el servicio exacto donde se produce el salto. Comparando las trazas rapidas con las lentas, se pueden encontrar las diferencias en la jerarquia de spans que explican la degradacion.

```text
Trazas rapidas (P50):  Gateway(10ms) → Usuarios(15ms) → DB(5ms) = 30ms
Trazas lentas (P99):   Gateway(10ms) → Usuarios(1800ms) → DB(5ms) = 1815ms
                                           ^
                                    Cuello de botella identificado
```

## Analisis de dependencias

El tracing distribuido construye implicitamente un grafo de dependencias del sistema. Cada vez que un servicio llama a otro, la jerarquia padre-hijo del span registra esa dependencia. Al analizar muchas trazas, se puede reconstruir el mapa completo de que servicios dependen de cuales otros, con que frecuencia y bajo que condiciones.

Este analisis es fundamental por varias razones. En primer lugar, permite identificar servicios criticos: aquellos que son dependencia de multiples servicios upstream y cuya caida afectaria a una porcion significativa del sistema. Un servicio que es padre en el 80% de las trazas es un punto unico de fallo critico que requiere monitorizacion y redundancia prioritarias. En segundo lugar, permite descubrir dependencias no documentadas: llamadas a servicios que no estaban en la arquitectura original pero que aparecieron durante el desarrollo y que pueden representar riesgos ocultos.

El analisis de dependencias tambien revela ciclos de dependencia y dependencias translativas que no son evidentes en diagramas estaticos. Un servicio A puede parecer independiente de B, pero el tracing revela que A llama a C, que llama a D, que llama a B, creando una cadena de dependencia indirecta de cuatro saltos que debe entenderse para evaluar el impacto real de una caida.

## Vista de cascada (Waterfall view)

La waterfall view es la visualizacion mas comun y intuitiva de una traza individual. Presenta los spans en un diagrama donde el eje horizontal representa el tiempo y cada span se dibuja como una barra horizontal cuya longitud es proporcional a su duracion. Los spans hijos se colocan indentados bajo su span padre, creando una estructura escalonada que revela la secuencia temporal y las relaciones de dependencia.

Leer un diagrama de cascada es analogo a leer un diagrama de Gantt de gestion de proyectos. Cada barra representa una tarea (un span), su posicion horizontal indica cuando ocurre en el tiempo, su longitud indica cuanto dura, y las barras anidadas indican subtareas que ocurren dentro de la tarea principal. La clave para interpretar un waterfall es identificar las barras mas largas (que consumen mas tiempo) y determinar si esa duracion se debe a procesamiento interno o a espera en llamadas downstream.

```text
Tiempo ────────────────────────────────────────────────>

API Gateway     [══════════════════════════════════] 800ms
  Usuarios      [═════════════════]                     400ms
    Cache       [═══]                                    30ms
    DB          [═══════════════]                       370ms
  Pagos         [════════]                              200ms
    Stripe      [════════]                              200ms
```

En el ejemplo anterior, el waterfall revela inmediatamente que la consulta a la base de datos en el Servicio Usuarios es el mayor consumidor de tiempo. Pero el analisis mas profundo muestra que el tiempo de la DB (370ms) es casi todo tiempo de ejecucion (la barra no tiene spans hijos significativos), mientras que el tiempo del Servicio Pagos (200ms) esta completamente determinado por la llamada a Stripe (200ms), que es tiempo de espera en un servicio externo. Estas dos situaciones requieren estrategias de optimizacion completamente distintas: optimizar la query SQL en un caso, y evaluar caching o asincronia en el otro.

Las vistas de cascada mas avanzadas muestran simultaneamente multiples trazas que comparten una estructura similar, permitiendo ver la variabilidad temporal de cada span a traves de muchas ejecuciones. Esto es especialmente util para identificar spans con alta varianza: operaciones que normalmente son rapidas pero que periodicamente experimentan picos de latencia, un patron tipico de problemas de contention de recursos, cold starts o interferencia de garbage collection en lenguajes gestionados.

> La waterfall view es ideal para analizar trazas individuales, pero no es practica para analizar tendencias a gran escala. Para identificar patrones sistemicos se recurre a metricas derivadas de trazas, como el percentil 99 de latencia por operacion, el throughput de spans por servicio o la tasa de spans con estado ERROR, que se almacenan como series temporales regulares.

