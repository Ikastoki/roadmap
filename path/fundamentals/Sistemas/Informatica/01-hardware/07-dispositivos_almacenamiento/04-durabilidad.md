# Métricas de durabilidad

La durabilidad de un SSD es una de las preocupaciones más frecuentes entre los usuarios que consideran la transición desde HDD. A diferencia de los discos duros, cuya vida útil está determinada por el desgaste mecánico de componentes móviles, la durabilidad de un SSD está directamente relacionada con la cantidad de datos escritos a lo largo de su vida. Las métricas de durabilidad permiten estimar cuánto tiempo funcionará un SSD de forma fiable bajo un patrón de uso determinado, y son especialmente importantes para servidores y sistemas donde la escritura es intensiva y continua.

Comprender estas métricas no solo ayuda a elegir el SSD adecuado, sino que también permite diseñar estrategias de mantenimiento y reemplazo proactivo que evitan la pérdida de datos por degradación de las celdas de memoria flash.

## Métricas

Las métricas de durabilidad de un SSD se expresan típicamente mediante dos indicadores principales que complementan la información sobre la vida útil esperada del dispositivo:

**TBW (*Total Bytes Written*):** representa la cantidad total de datos que se pueden escribir en el SSD durante su vida útil antes de que las celdas de memoria comiencen a degradarse de forma significativa. Se expresa en terabytes (TB) y varía enormemente según el modelo y la capacidad del SSD. Un SSD de 256 GB puede tener un TBW de 150 TB, mientras que uno de 2 TB puede superar los 2,500 TB. El TBW es la métrica más utilizada por los fabricantes para especificar la durabilidad de sus productos.

**DWPD (*Drive Writes Per Day*):** indica cuántas veces se puede escribir la capacidad completa del SSD cada día durante el período de garantía. Un SSD de 1 TB con 1 DWPD y una garantía de 5 años permite escribir 1 TB de datos al día durante 5 años sin exceder la durabilidad especificada. Un DWPD de 3 implica que se pueden escribir 3 TB diarios, lo que es típico de SSD de servidor.

La conversión entre ambas métricas es directa:

```
TBW = DWPD x Capacidad (TB) x 365 días x Años de garantía

Ejemplo: 1 DWPD x 1 TB x 365 días x 5 años = 1,825 TBW
```

Para un usuario doméstico promedio que escribe entre 10-40 GB al día, un SSD moderno de 1 TB con 600 TBW de durabilidad podría durar más de 40 años. Esta cifra ilustra que, para el uso doméstico típico, la degradación por escritura no debería ser una preocupación práctica durante la vida útil del producto.

> Los fabricantes de SSD ofrecen herramientas de monitorización (Samsung Magician, Crucial Storage Executive, WD Dashboard) que muestran el estado de salud del SSD basándose en las métricas S.M.A.R.T., incluyendo el porcentaje de vida útil restante calculado a partir del TBW acumulado.

## Durabilidad

La durabilidad de un SSD se refiere a su capacidad para resistir factores ambientales y de uso que podrían causar pérdida de datos o fallo del dispositivo:

**Resistencia a golpes y vibraciones:** al no tener partes mecánicas, los SSD son significativamente más resistentes que los HDD a impactos, caídas y vibraciones. Un SSD puede soportar aceleraciones de 1,500G durante operación, mientras que un HDD puede dañarse con caídas de apenas 30-60 cm. Esta resistencia los hace ideales para portátiles, entornos industriales y cualquier aplicación donde la integridad física no está garantizada.

**Rango de temperatura:** los SSD operan típicamente entre 0°C y 70°C, aunque el rango de almacenamiento puede extenderse de -40°C a 85°C. A diferencia de los HDD, que son sensibles a temperaturas extremas (especialmente durante el arranque), los SSD son tolerantes a una gama más amplia de condiciones ambientales.

**Retención de datos sin energía:** las celdas NAND retienen los datos durante un período que varía según la temperatura de almacenamiento y el nivel de desgaste de las celdas. Un SSD nuevo puede retener datos durante más de 10 años a 30°C, pero un SSD altamente desgastado puede retenerlos solo durante unos meses a temperaturas elevadas. Los fabricantes especifican típicamente un período de retención de 1 año a 30°C para SSD en estado no operativo.

**Resistencia electromagnética:** al no basarse en magnetización para almacenar datos, los SSD son inmunes a la interferencia electromagnética que podría afectar a los HDD. Sin embargo, las descargas electrostáticas (ESD) pueden dañar los componentes electrónicos del SSD, por lo que las precauciones estándar de manejo de hardware siguen siendo aplicables.

## Vida útil

La vida útil de un SSD se define como el período durante el cual el dispositivo puede funcionar de forma fiable sin pérdida de datos ni degradación significativa del rendimiento. Varios factores influyen en la vida útil real:

**Patrón de escritura:** la vida útil está directamente relacionada con el volumen de datos escritos. Un SSD utilizado exclusivamente como unidad de sistema para navegación y ofimática durará significativamente más que uno utilizado como destino de grabación de vídeo continuo o como disco de swap intensivo.

**Temperatura de operación:** las temperaturas elevadas aceleran la degradación de las celdas NAND. Mantener el SSD dentro del rango de temperatura recomendado (típicamente 30-50°C) maximiza su vida útil. Los SSD NVMe de alta gama pueden alcanzar temperaturas elevadas bajo carga sostenida, lo que puede provocar *thermal throttling* (reducción automática de velocidad para evitar daños por calor).

**Type of NAND:** como se ha mencionado, SLC ofrece la mayor durabilidad (100,000 ciclos), seguida de MLC (10,000), TLC (1,000-3,000) y QLC (100-1,000). Para aplicaciones de escritura intensiva, los SSD SLC o MLC siguen siendo preferibles a pesar de su mayor coste.

## Copias de seguridad

Ninguna métrica de durabilidad elimina la necesidad de copias de seguridad. Los SSD, aunque fiables, pueden fallar de forma abrupta sin los síntomas de advertencia que a menudo preceden al fallo de un HDD. Un HDD con sectores defectuosos suele degradar su rendimiento gradualmente, dando tiempo al usuario para reaccionar. Un SSD, en cambio, puede pasar de funcionar perfectamente a fallar completamente en cuestión de horas si un controlador falla o las celdas alcanzan su límite de escritura.

Una estrategia de copias de seguridad adecuada para un SSD incluye:

- **Copias de seguridad regulares** en un medio externo o en la nube, con una frecuencia determinada por la criticidad de los datos.
- **Monitorización del estado de salud** mediante herramientas S.M.A.R.T. que reportan el porcentaje de vida útil restante y el TBW acumulado.
- **Reemplazo proactivo** del SSD cuando el porcentaje de vida útil restante cae por debajo del 10-20%, antes de que la degradación de las celdas pueda causar pérdida de datos.
- **Evitar la escritura innecesaria** configurando el sistema para minimizar operaciones de escritura superfluas, como desactivar la desfragmentación (innecesaria y perjudicial en SSD) y optimizar la configuración de paginación.

> La velocidad de los SSD puede enmascarar una realidad importante: al contrario de lo que muchos usuarios asumen, los SSD no son infalibles. La tecnología NAND tiene un número finito de ciclos de escritura, y los fallos de controlador pueden ser catastróficos e impredecibles. La copia de seguridad sigue siendo la única garantía real de no perder datos.

