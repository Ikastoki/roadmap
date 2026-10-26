# Controladores de almacenamiento

Las tarjetas de control de almacenamiento son circuitos de expansión que proporcionan puertos de conexión para dispositivos de almacenamiento adicionales o funcionalidades de gestión avanzada que el controlador integrado en la placa base no ofrece. Estas tarjetas son especialmente relevantes en servidores, estaciones de trabajo y sistemas NAS donde se necesita una gran cantidad de puertos de almacenamiento, gestión RAID por hardware o soporte para interfaces de alta velocidad.

Aunque el controlador SATA integrado en la mayoría de placas base permite conectar entre 4 y 8 dispositivos de almacenamiento, las necesidades de los servidores y sistemas de almacenamiento empresariales suelen superar estas capacidades, requiriendo soluciones de expansión dedicadas.

## Tarjetas RAID dedicadas (HBA)

Las tarjetas RAID de hardware proporcionan gestión de conjuntos RAID con un procesador dedicado (*RAID-on-Chip*, ROC) que ejecuta los cálculos de paridad y striping sin carga del procesador del sistema. A diferencia del RAID por software, que compite por los ciclos de CPU con las aplicaciones del usuario, el RAID por hardware opera de forma independiente y no afecta al rendimiento general del sistema.

Los controladores RAID de hardware de gama alta (LSI/Avago, Adaptec, HighPoint) incluyen:

- **Procesador dedicado:** un chip ARM o x86 que gestiona todas las operaciones de RAID sin intervención del CPU del sistema.
- **Caché de escritura:** memoria caché de 256 MB a 4 GB que almacena temporalmente las escrituras antes de commitarlas al disco, mejorando significativamente el rendimiento de escritura aleatoria.
- **Protección de caché por batería/supercondensador:** garantiza que los datos almacenados en caché no se pierdan en caso de corte de energía, proporcionando la misma fiabilidad que las escrituras directas al disco.
- **Soporte para múltiples niveles de RAID:** RAID 0, 1, 5, 6, 10, 50, 60 y configuraciones personalizadas con gestión desde una interfaz dedicada (BIOS RAID o software de gestión).

**HBA (*Host Bus Adapter*):** es una variante de la tarjeta RAID que presenta los dispositivos de almacenamiento directamente al sistema operativo sin gestión RAID. Los HBA se utilizan cuando la gestión RAID se realiza por software (ZFS, mdadm) o cuando se necesita acceso directo a los discos individuales, como en entornos de virtualización con almacenamiento compartido.

> La elección entre un controlador RAID y un HBA depende de la arquitectura del sistema. Para RAID 5/6 por hardware con caché protegida, un controlador RAID es preferible. Para ZFS o RAID por software, un HBA proporciona la funcionalidad necesaria sin la complejidad innecesaria de un controlador RAID.

## Expansión de puertos SATA/SAS

Las tarjetas de expansión de puertos de almacenamiento añaden puertos SATA o SAS adicionales al sistema:

**Controladores SATA:** tarjetas que añaden 2-8 puertos SATA adicionales mediante un chip controlador (típicamente Marvell o ASMedia). Son económicas y fáciles de instalar, adecuadas para usuarios que necesitan conectar más discos de los que la placa base permite.

**Controladores SAS:** las tarjetas SAS (*Serial Attached SCSI*) proporcionan puertos que son compatibles tanto con discos SATA como con discos SAS de gama alta. Los discos SAS offer mayor fiabilidad (dual port), velocidades de hasta 12 Gbps y soporte para grandes cantidades de dispositivos en un solo bus. Las tarjetas SAS son estándar en servidores donde se requiere alta disponibilidad y escalabilidad.

**Expansión con backplane:** en servidores y NAS, las tarjetas de control de almacenamiento se conectan a *backplanes* —placas con múltiples puertos SATA/SAS que permiten la inserción y extracción caliente de discos sin necesidad de cableado manual. Los backplanes simplifican enormemente la gestión de almacenamiento en sistemas con muchos discos.

**Expansión M.2:** las tarjetas adaptadoras M.2 permiten instalar múltiples SSD M.2 en slots PCIe, añadiendo almacenamiento NVMe de alta velocidad a sistemas que no tienen suficientes slots M.2 integrados. Algunas tarjetas adaptadoras incluyen disipadores térmicos integrados para mantener los SSD NVMe dentro de sus rangos de temperatura operativos.

