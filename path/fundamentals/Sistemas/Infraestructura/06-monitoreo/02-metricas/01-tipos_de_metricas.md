# Tipos de metricas

El correcto modelado de datos de monitorizacion depende enteramente de comprender la naturaleza de lo que se mide. No todas las magnitudes se comportan igual: algunas solo crecen, otras fluctuan libremente y otras representan distribuciones estadisticas de poblaciones completas. Elegir el tipo de metrica equivocado genera dashboards inutiles, alertas falsas y, peor aun, la illusion de que un sistema esta saludable cuando en realidad esta degradandose. Los sistemas de monitorizacion modernos como Prometheus, Datadog o InfluxDB implementan tipos primitivos bien definidos, cada uno con reglas de agregacion y almacenamiento propias.

## Contadores (Counters)

Un contenedor es una metrica que solo crece o se reinicia a cero cuando el proceso que la emite se reinicia. Representa una cantidad acumulada de eventos a lo largo de la vida de un servicio: peticiones HTTP recibidas, bytes transferidos, errores encontrados, transacciones procesadas. El valor absoluto de un contenedor en un instante concreto carece de significado por si solo; lo relevante es la tasa de crecimiento calculada como la derivada entre dos muestras consecutivas.

La analogia mas precisa es el odometro de un coche: solo avanza hacia delante, nunca retrocede. Si el odometro marca 50.000 km y al dia siguiente 50.100 km, sabemos que se recorrieron 100 km. Pero si el coche se desguaza y se compra uno nuevo, el odometro vuelve a cero, igual que un contenedor tras un reinicio del proceso.

> Los contadores son la base para calcular tasas (rates). Prometheus expone funciones como `rate()` y `increase()` que derivan automaticamente un contenedor para obtener valores interpretables por segundo o por intervalo. Sin esta transformacion, un contenedor puro es poco util en dashboards.

El error comun es mostrar un contenedor crudo en un grafico de linea, lo que genera una grafica siempre creciente que oculta cualquier patron real. Siempre se debe graficar la tasa derivada del contenedor.

## Medidores (Gauges)

Un medidor representa un valor que puede fluctuar en cualquier direccion: subir, bajar o mantenerse estable. Captura el estado actual de un recurso en un momento dado. Ejemplos habituales incluyen el uso de CPU, la memoria consumida, el numero de conexiones activas, la longitud de una cola de mensajes o la temperatura de un servidor.

A diferencia del contenedor, el medidor es directamente interpretable sin transformaciones adicionales. Si un medidor indica 78% de uso de CPU, eso es exactamente lo que significa en ese instante. La analogia clasica es el deposito de combustible de un coche: el nivel sube cuando se llena y baja cuando se consume, y en cualquier momento puedes leer el nivel exacto.

Los medidores son ideales para dashboards de estado en tiempo real y para alertas que responden a umbrales absolutos. Sin embargo, suelen requerir menor frecuencia de muestreo que los contadores porque representan snapshots del estado, no acumulaciones.

> Un medidor no es suitable para calcular tasas de forma natural. Si necesitas medir la velocidad de cambio de un medidor (por ejemplo, cuanto crece la memoria en bytes por segundo), deberas calcular la derivada manualmente entre dos puntos, lo cual introduce error por el intervalo de muestreo.

## Histogramas (Histograms)

Un histograma agrupa muestras de una metrica en buckets predefinidos de rango de valores. Para cada peticion, el histograma incrementa el contador del bucket correspondiente. El resultado es una distribucion de frecuencias que revela como se reparten los valores dentro de una poblacion, no solo su promedio.

Imaginemos que un profesor quiere entender las calificaciones de un examen de 100 alumnos. El promedio de 6.2 no dice nada sobre si hay un grupo de alumnos suspendidos y otro con nota alta. Un histograma agruparia las notas en rangos (0-2, 2-4, 4-6, 6-8, 8-10) y mostraria cuantos alumnos caen en cada rango, revelando la forma real de la distribucion.

En contextos de monitorizacion, los histogramas son especialmente valiosos para metricas de latencia. Un servicio con latencia promedio de 50ms puede estar atendiendo el 99% de peticiones en 30ms y el 1% restante en 2.000ms. Un promedio simple ocultaria este problema criticamente importante. El histograma lo haria visible al mostrar que el bucket de 1.000-5.000ms tiene muestras.

```text
Bucket        | Count
0-10ms        | ████ 4.200
10-50ms       | ████████ 8.100
50-100ms      | ███ 3.000
100-500ms     | █ 800
500-1000ms    | ▎ 120
1000-5000ms   | ▏ 30
```

El compromiso es la precision: los buckets son aproximaciones. La definicion de los limites de cada bucket determina la resolucion de la distribucion. Prometheus permite configurar buckets personalizados mediante `histogram_bucket` o usar defaults de 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10 segundos para latencias HTTP.

> Los histogramas consumen mas memoria y almacenamiento que contadores o medidores porque almacenan multiples contadores por muestra. Un histograma con 10 buckets genera 10 series de time series por metrica, lo que debe considerarse en entornos con alta cardinalidad.

## Resumenes (Summaries)

Un resumen es conceptualmente similar a un histograma pero calcula los percentiles en el lado del emisor (el cliente que genera la metrica) en lugar de hacerlo en el servidor de monitorizacion. Emiten directamente el valor del percentil, la media y el conteo total, sin necesidad de que el backend procese los buckets.

La diferencia fundamental con el histograma es donde ocurre el calculo estadistico. En un histograma, el backend (Prometheus) recibe los buckets y puede calcular cualquier percentil, quantil o percentil sobre la marcha. En un resumen, el emisor ya ha calculado el percentil y solo envia el resultado final.

> Los resumenes no son agregables entre instancias. Si tienes tres réplicas de un servicio, cada una emitiendo su propio P99, no puedes calcular el P99 global sumando los tres valores. Los histogramas si permiten esta agregacion porque sus buckets si son sumables.

## Criterios de seleccion

La decision entre tipos depende de la naturaleza de la magnitud y de como se consumira la informacion. Si algo se acumula sin limite, es un contenedor. Si representa un estado actual fluctuante, es un medidor. Si necesitas entender la distribucion de una poblacion de valores, es un histograma o resumen. El histograma es preferible cuando se necesita agregacion entre multiples instancias; el resumen cuando se quiere minimizar la carga del backend y cada instancia se analiza de forma independiente.

