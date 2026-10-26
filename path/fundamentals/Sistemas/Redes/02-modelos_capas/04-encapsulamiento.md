# Encapsulamiento de datos

El encapsulamiento es el proceso por el cual cada capa del modelo de red anhade su propia cabecera (y a veces una cola) a los datos que recibe de la capa superior, creando una unidad de datos completa que puede ser transmitida por la red. Este proceso es fundamental para el funcionamiento de los modelos de capas, ya que permite que cada capa opere de forma independiente sobre su propio formato de datos.

Cuando los datos viajan desde la capa de aplicacion hacia abajo (emision), cada capa anhade informacion de control. Cuando los datos llegan al receptor y viajan hacia ar_RECEPCION), cada capa elimina la cabecera que le corresponde y procesa los datos antes de pasarselos a la capa superior.

## Proceso de encapsulamiento

El flujo de datos en el emisor es el siguiente:

1. La **capa de aplicacion** genera los datos originales (mensaje HTTP, comando SSH, etc.).
2. La **capa de transporte** divide los datos en segmentos (TCP) o datagramas (UDP), anhadiendo cabeceras con puertos origen/destino, numeros de secuencia y otros campos de control.
3. La **capa de red** anhade cabeceras con direcciones IP origen y destino, creando paquetes.
4. La **capa de enlace** anhade cabeceras con direcciones MAC y un FCS (Frame Check Sequence), creando tramas.
5. La **capa fisica** convierte la trama en una secuencia de bits y la transmite por el medio fisico.

```
Aplicacion:    [       Datos        ]
                    ↓ + TCP/UDP header
Transporte:    [H][      Datos      ]  → Segmento
                    ↓ + IP header
Red:           [H][H][    Datos     ]  → Paquete
                    ↓ + MAC header + FCS
Enlace:        [H][H][H][ Datos ][F] → Trama
                    ↓
Fisica:        0110101101001010...      → Bits
```

## Proceso de desencapsulamiento

En el receptor, el proceso se invierte. Cada capa recibe la PDU de la capa inferior, elimina su cabecera, procesa los datos y los pasa a la capa superior:

1. La **capa fisica** recibe los bits y los reconstruye como trama.
2. La **capa de enlace** verifica la integridad (FCS), elimina la cabecera MAC y extrae el paquete.
3. La **capa de red** verifica la direccion IP, elimina la cabecera IP y extrae el segmento.
4. La **capa de transporte** utiliza el puerto para identificar el proceso destino, elimina la cabecera TCP/UDP y entrega los datos a la aplicacion.
5. La **capa de aplicacion** procesa el mensaje original.

## Cabeceras y payload

Cada cabecera contiene campos especificos que la capa correspondiente utiliza para su funcion:

- **Cabecera TCP:** puertos origen (16 bits) y destino (16 bits), numero de secuencia (32 bits), numero de acuse de recibo (32 bits), ventana de recepcion, flags (SYN, ACK, FIN, etc.).
- **Cabecera IP:** version, longitud de cabecera, longitud total, TTL (*Time To Live*), protocolo, checksum, IP origen (32 bits), IP destino (32 bits).
- **Cabecera Ethernet:** MAC destino (48 bits), MAC origen (48 bits), tipo/longitud, FCS (32 bits).

El **payload** es la porcion util de datos que se transporta. La cabecera es overhead necesario para que la red pueda entregar los datos correctamente, pero no contiene informacion util para la aplicacion. A medida que los datos descienden por las capas, el overhead de cabeceras se acumula, reduciendo la eficiencia de la transmision.

> El encapsulamiento es lo que hace posible la interoparabilidad en las redes: una aplicacion web no necesita conocer si la red subyacente es Ethernet, Wi-Fi o fibra optica; solo necesita enviar datos HTTP. Cada capa se encarga de adaptar los datos al medio que le corresponde, ocultando la complejidad a las capas superiores.

