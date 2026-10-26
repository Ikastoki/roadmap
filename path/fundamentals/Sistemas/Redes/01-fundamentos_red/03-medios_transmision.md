# Medios de transmision

Los medios de transmision son los canales fisicos a traves de los cuales viajan las senales que transportan datos de un dispositivo a otro. La eleccion del medio de transmision determina la velocidad maxima, la distancia, la inmunidad al ruido y el coste de la infraestructura de red. Los medios se clasifican en guiados (donde la senal viaja por un conductor fisico) y no guiados (donde la senal viaja por el aire o el vacio).

Comprender las caracteristicas de los medios de transmision es fundamental para disenar una red que cumpla con los requisitos de rendimiento y fiabilidad. Un medio inadecuado puede convertirse en un cuello de botella que limite todo el rendimiento del sistema.

## Ancho de banda, latencia y jitter

**Ancho de banda (*bandwidth*):** es la capacidad maxima de transmision de datos de un medio, medida en bits por segundo (bps). En la practica, se utilizan multiples de bits: Mbps (megabits), Gbps (gigabits) o Tbps (terabits). El ancho de banda no es la velocidad real de transferencia, sino la capacidad maxima teorica del canal.

**Latencia (*latency*):** es el tiempo que tarda un bit en viajar desde el emisor hasta el receptor. Incluye la latencia de propagacion (tiempo fisico de viaje de la senal), la latencia de transmision (tiempo necesario para poner todos los bits en el medio) y la latencia de procesamiento (tiempo que tardan los dispositivos intermedios en reenviar los datos). La latencia se mide en milisegundos (ms).

**Jitter:** es la variacion de la latencia a lo largo del tiempo. Un jitter alto es perjudicial para aplicaciones en tiempo real como VoIP o videollamadas, donde las variaciones de latencia causan cortes y distorsion. Un jitter estable (bajo) es mas importante que una latencia baja para ciertas aplicaciones.

## Medios guiados

Los medios guiados son conductores fisicos por los que viajan las senales electricas o opticas. Proporcionan un camino definido para la senal, lo que ofrece mayor control sobre la propagacion y menor interferencia con el exterior.

**Par trenzado:** el medio mas comun en las redes LAN. Consiste en pares de cables de cobre trenzados entre si para reducir la interferencia electromagnetica. Se presenta en variantes UTP (*Unshielded Twisted Pair*, sin blindaje) y STP (*Shielded Twisted Pair*, con blindaje). Las categorias mas comunes son Cat5e (1 Gbps), Cat6 (1-10 Gbps) y Cat6a (10 Gbps a 100 metros).

**Fibra optica:** utiliza pulsos de luz para transmitir datos a traves de un nucleo de vidrio o plastico. Ofrece ancho de banda extremadamente alto, inmunidad total a interferencias electromagneticas y largas distancias (decenas de kilometros). Se presenta en variantes monomodo (un solo rayo de luz, larga distancia) y multimodo (multiples rayos, corta distancia).

**Coaxial:** medio historico que se utilizaba en redes Ethernet tempranas (10BASE2, 10BASE5). Consiste en un conductor central rodeado de aislante, una malla conductora y una cubierta protectora. Actualmente se usa principalmente para television por cable y conexiones de internet por cable (DOCSIS).

## Medios no guiados

Los medios no guiados transmiten senales por el aire, el agua o el espacio sin un conductor fisico. Son esenciales para comunicaciones moviles y en areas donde el cableado es impracticable.

**Radiofrecuencia:** la tecnologia base de Wi-Fi, Bluetooth, LTE y 5G. Las ondas de radio se propagan en todas las direcciones y pueden atravesar obstaculos no metalicos. La frecuencia determina el alcance y la capacidad: 2.4 GHz ofrece mayor alcance pero menor velocidad; 5 GHz ofrece mayor velocidad pero menor alcance.

**Microondas:** ondas de alta frecuencia (1-300 GHz) que se propagan en linea recta. Se utilizan para enlaces punto a punto a larga distancia (entre edificios, entre ciudades). Requieren lineas de vista entre emisor y receptor y son sensibles a las condiciones meteorologicas.

**Infrarrojo:** ondas electromagneticas de frecuencia inferior a la de la luz visible. Se utilizan para comunicaciones corta distancia (controles remotos, algunos dispositivos de red legacy). No atraviesan obstaculos opacos y tienen un alcance limitado.

## Relacion senal/ruido (SNR)

La relacion senal/ruido (*Signal-to-Noise Ratio*, SNR) es una medida de la calidad de la senal en un canal de comunicacion. Se expresa en decibelios (dB) y representa la relacion entre la potencia de la senal util y la potencia del ruido que la acompana.

Un SNR alto indica una senal limpia y fuerte en comparacion con el ruido; un SNR bajo indica una senal debil que puede ser dificil de decodificar correctamente. La capacidad maxima de un canal esta determinada por el teorema de Shannon: C = B * log2(1 + SNR), donde C es la capacidad en bps y B es el ancho de banda en Hz.

> La eleccion del medio de transmision debe equilibrar cuatro factores: capacidad (ancho de banda), distancia, coste y resistencia al ruido. La fibra optica ofrece la mejor combinacion de capacidad y distancia pero es mas cara de instalar. El par trenzado es economico y versatil pero tiene limitaciones de distancia y velocidad. Las redes inalambricas son moviles pero sensibles al ruido y las interferencias.

