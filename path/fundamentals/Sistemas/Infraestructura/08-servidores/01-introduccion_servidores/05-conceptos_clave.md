# Conceptos clave

El lenguaje tecnico que rodea a los servidores se apoya en un conjunto de metricas y conceptos que determinan como se mide, se dimensiona y se optimiza un servicio. Comprender cada uno de ellos es necesario para tomar decisiones informadas sobre la arquitectura de una infraestructura y para comunicarse eficazmente con otros profesionales del sector.

## Uptime y disponibilidad

El uptime es el porcentaje de tiempo que un servidor permanece operativo durante un periodo determinado, normalmente medido en meses o anos. Se expresa con niveles de disponibilidad que siguen una convencion estandar: "nueve" seguidos de un punto decimal y un numero. Un uptime del 99,9% implica aproximadamente 8 horas y 45 minutos de inactividad al ano, mientras que un 99,99% reduce esa cifra a poco mas de 52 minutos. Cada decimal adicional suplica un incremento exponencial en el coste de la infraestructura necesaria para conseguirlo.

La disponibilidad no es solo una metrica tecnica; tiene implicaciones economicas directas. Para un comercio electronico que factura 10.000 euros por hora, cada minuto de inactividad representa una perdida de aproximadamente 167 euros. La estrategia de alta disponibilidad se construye sobre multiples pilares: redundancia hardware (fuentes duplicadas, RAID), redundancia software (clustering, replicacion), procesos operativos (actualizaciones con rolling restart) y planes de recuperacion ante desastres. El nivel de disponibilidad requerido determina que combinacion de estos pilares es necesaria, y cada uno incrementa la complejidad y el coste.

La diferencia entre uptime y disponibilidad es sutil pero importante. El uptime mide si el servidor esta encendido, pero la disponibilidad mide si el servicio que ofrece es accesible y funcional. Un servidor puede estar encendido y respondiendo pings pero con la base de datos caida, lo que lo convierte en un equipo con uptime perfecto pero sin disponibilidad del servicio. Las metricas de disponibilidad deben medir el servicio completo, no solo el estado del hardware.

## Latencia

La latencia es el tiempo que transcurre entre el momento en que un cliente envia una peticion y el momento en que recibe la respuesta. Se mide en milisegundos y su impacto varia radicalmente segun el tipo de servicio. En una peticion a una base de datos, una latencia de 10 milisegundos es habitual y acceptable, pero en un servicio de streaming en tiempo real, 100 milisegundos de latencia pueden provocar un retraso perceptible por el usuario.

La latencia se compone de multiples factores: el tiempo de propagacion de la senal por el medio fisico (cobre, fibra, aire), el tiempo de procesamiento en cada nodo intermedio (routers, switches, firewalls), el tiempo de encolamiento en los buffers de los dispositivos de red y el tiempo de procesamiento en el servidor destino. Cada componente suma su propio retardo, y en una ruta con varios saltos estos se acumulan de forma lineal.

La latencia no se puede reducir indefinidamente porque esta limitada por la velocidad de la luz en el medio de transmision. Un paquete que viaja de Madrid a Tokyo recorre aproximadamente 20.000 kilometros, lo que impone una latencia minima teorica de unos 66 milisegundos solo por el tiempo de propagacion, sin contar el procesamiento. Este limite fisico es la razon por la que la proximidad geografica importa en la arquitectura de servicios y por la que se despliegan servidores en multiples regiones para acercar el servicio al usuario final.

## Throughput

El throughput mide la cantidad de datos o peticiones que un servidor puede procesar en una unidad de tiempo. A diferencia de la latencia, que mide la rapidez de una operacion individual, el throughput cuantifica el volumen total de trabajo completado. Se expresa en distintas unidades segun el contexto: megabytes por segundo (MB/s) para transferencias de archivos, peticiones por segundo (req/s) para servidores web, o consultas por segundo (QPS) para bases de datos.

El throughput y la latencia estan interrelacionados pero no son inversamente proporcionales. Un servidor puede tener una latencia baja pero un throughput limitado si solo puede procesar unas pocas peticiones concurrentes, o puede tener un throughput alto con latencia elevada si procesa muchas peticiones en paralelo pero cada una tarda mucho. El escenario ideal es mantener ambos valores optimos simultaneamente, lo que generalmente requiere un equilibrio entre recursos CPU, memoria, disco y red.

En la practica, el throughput se ve afectado por cuellos de botella que no siempre son obvios. Un disco SSD con un throughput nominal de 500 MB/s puede verse reducido a 100 MB/s si la operacion es aleatoria en lugar de secuencial. Una tarjeta de red de 1 Gbps puede no alcanzar esa velocidad si el procesador no puede cifrar los datos a tiempo con TLS. La identificacion de estos cuellos de botella es una de las tareas mas importantes del administrador de sistemas.

## Escalabilidad vertical y horizontal

La escalabilidad vertical (scale-up) consiste en aumentar los recursos de un servidor existente: anadir mas RAM, un procesador mas potente o discos mas rapidos. Es el enfoque mas simple porque no requiere cambios en la arquitectura de la aplicacion ni en la configuracion de la red. Sin embargo, tiene un limite fisico: un servidor solo puede alojar un numero finito de slots de memoria y procesadores, y llega un punto en el que el coste del siguiente incremento de hardware supera el beneficio obtenido.

La escalabilidad horizontal (scale-out) consiste en anadir mas servidores al sistema y distribuir la carga entre ellos. Este enfoque es mas complejo porque implica resolver problemas de consistencia de datos, sesiones de usuario persistentes, balanceo de carga y coordinacion entre nodos. Sin embargo, su limite teorico es mucho mayor: siempre es posible anadir un servidor adicional, y los algoritmos de escalado automatico pueden hacerlo dinamicamente segun la demanda.

En la practica, la mayoria de servicios modernos combinan ambas estrategias. Un servidor se dimensiona con la mayor cantidad de recursos que resulte rentable (vertical) y luego se replica horizontalmente cuando la demanda supera la capacidad de una unica instancia. Las bases de datos suelen escalar verticalmente porque la replicacion complejiza la consistencia, mientras que los servidores web escalan horizontalmente con facilidad porque son basicamente estadoless: cada peticion puede ser atendida por cualquier instancia sin depender de las anteriores.

## Margen de operacion

El margen de operacion (headroom) es la diferencia entre la capacidad maxima de un sistema y la carga que efectivamente maneja. Un servidor con capacidad para 1.000 peticiones por segundo que normalmente recibe 400 opera con un margen del 60%. Este margen existe para absorber picos de demanda imprevistos, permitir el mantenimiento sin interrupcion del servicio y garantizar que el rendimiento no se degrada bajo carga.

Un margen demasiado elevado implica infraestructura pagada pero subutilizada, lo que representa un coste innecesario. Un margen demasiado bajo convierte cualquier anomalia — un ataque DDoS, un pico estacional, un proceso de mantenimiento — en un incidente que afecta a los usuarios. La definicion del margen adecuado depende del patron de trafico del servicio, su criticidad y la velocidad a la que se puede ampliar la capacidad. Los servicios con trafico predecible pueden operar con margenes ajustados, mientras que aquellos con fluctuaciones impredecibles requieren margenes generosos o mecanismos de escalado automatico.

