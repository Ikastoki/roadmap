# PDU y analogia practica

La *Protocol Data Unit* (PDU) es el termino generico que designa la unidad de datos que cada capa del modelo de red transporta. Cada capa tiene su propia PDU con un nombre especifico que refleja su funcion y estructura. Conocer las PDU es esencial para comunicarse con precision sobre problemas de red y para comprender como se transforman los datos a lo largo del proceso de comunicacion.

El concepto de PDU refuerza la idea de que cada capa opera sobre un formato de datos determinado: la capa de transporte trabaja con segmentos, la de red con paquetes, la de enlace con tramas. Cuando un profesional dice "el paquete se perdio", se esta refiriendo especificamente a una PDU de capa 3.

## PDU en cada capa

| Capa | PDU | Descripcion |
|---|---|---|
| 7 - Aplicacion | Mensaje / Datos | Datos originales de la aplicacion |
| 6 - Presentacion | Mensaje / Datos | Datos formateados/cifrados |
| 5 - Sesion | Mensaje / Datos | Datos con control de sesion |
| 4 - Transporte | Segmento (TCP) / Datagrama (UDP) | Datos con puertos y control de flujo |
| 3 - Red | Paquete (*packet*) | Segmento con direcciones IP |
| 2 - Enlace | Trama (*frame*) | Paquete con direcciones MAC y FCS |
| 1 - Fisica | Bits | Trama convertida en senales electricas/opticas |

Es importante distinguir entre estas PDU:

- **Segmento (TCP):** incluye cabecera con puertos, secuencia y acuse de recibo. Orientado a conexion.
- **Datagrama (UDP):** incluye cabecera minima con puertos. Sin conexion.
- **Paquete:** la PDU mas comunmente referida en la practica. Cuando se habla de "paquete de red", se refiere a una PDU de capa 3.
- **Trama:** la PDU de capa 2. Incluye direcciones MAC para la comunicacion en la red local.

## Analogia: enviar un correo electronico

Para ilustrar el encapsulamiento, imagina que envias un correo electronico a un amigo en otra ciudad:

**Capa de aplicacion (tu mensaje):** escribes el texto del correo: "Hola, nos vemos el lunes". Este es el payload original.

**Capa de presentacion (formato):** el correo electronico se codifica en UTF-8, se comprime y se cifra si utilizas cifrado de extremo a extremo.

**Capa de sesion (conexion):** se establece la sesion con el servidor de correo. Se autentica tu identidad y se gestiona el dialogo cliente-servidor.

**Capa de transporte (puerto):** el correo se dirige al puerto 587 (SMTP) del servidor de correo. Se asigna un identificador de transaccion.

**Capa de red (IP):** se anhade la IP de tu servidor de correo como origen y la IP del servidor de correo del destinatario como destino. El paquete viaja a traves de multiples routers.

**Capa de enlace (MAC):** en cada salto de la red, el paquete se encapsula en una trama con las direcciones MAC del dispositivo origen y el siguiente salto (router) en la red local.

**Capa de fisica (bits):** la trama se convierte en senales electricas (cable Ethernet) o electromagneticas (Wi-Fi) que viajan por el medio fisico hasta el siguiente dispositivo.

En el receptor, el proceso se invierte: cada capa elimina su cabecera y entrega los datos a la capa superior hasta que tu amigo recibe y lee el mensaje original.

## Nota sobre la terminologia en la practica

En el dia a dia de las redes, los profesionales no siempre usan la terminologia estricta de PDU. Es comun referirse a "paquetes" de forma general para cualquier unidad de datos en transmision, independientemente de la capa. Sin embargo, en contextos tecnicos y de diagnostico, la precision en la terminologia es importante: saber si un problema ocurre en capa 2 (tramas), capa 3 (paquetes) o capa 4 (segmentos) permite aislar la causa del fallo rapidamente.

> Las PDU no son solo un concepto academico: son la base del diagnostico de redes. Herramientas como Wireshark permiten capturar y analizar PDU de multiples capas simultaneamente, mostrando desde la cabecera Ethernet hasta los datos de la aplicacion. Comprender la estructura de cada PDU es esencial para interpretar los resultados de un analisis de trafico.

