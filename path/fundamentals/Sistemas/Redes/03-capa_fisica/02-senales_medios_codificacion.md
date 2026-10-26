# Senales, medios y codificacion

La capa fisica se ocupa de como representar bits como senales fisicas, que medios utilizan para transportar esas senales y como codificar la informacion de forma eficiente. Estos tres elementos determinan la velocidad, la distancia y la fiabilidad de la transmision en una red.

Las senales pueden ser analogicas (ondas continuas) o digitales (estados discretos). Los medios pueden ser guiados (cables) o no guiados (aire). Y la codificacion determina como se asignan bits a las variaciones de la senal.

## Senales analogicas vs digitales

**Senales analogicas:** son ondas continuas que pueden tomar cualquier valor dentro de un rango. La voz humana, la musica y la senal de television analogica son ejemplos. En una senal analogica, la informacion se codifica en las variaciones de amplitud, frecuencia y fase.

**Senales digitales:** son senales que solo toman valores discretos (tipicamente dos: 0 y 1). Los ordenadores trabajan con senales digitales, pero las redes de telecomunicaciones historicas (telefono, television) trabajaban con senales analogicas. La conversion entre ambos mundos se realiza con moduladores/demoduladores (modems).

**Amplitud:** la altura de la onda. En una senal electrica, representa la intensidad del voltaje. En una senal de radio, representa la potencia de la senal.

**Frecuencia:** el numero de ciclos por segundo de la onda, medido en Hertz (Hz). Una frecuencia alta permite transmitir mas informacion por unidad de tiempo pero tiene mayor atenuacion a largas distancias.

**Fase:** el punto del ciclo en el que se encuentra la onda en un momento dado. La modulacion por fase es utilizada en tecnologias como Wi-Fi.

## Ancho de banda

El **ancho de banda** (*bandwidth*) es el rango de frecuencias que un canal puede transmitir. Se mide en Hertz (Hz) para canales analogicos y en bits por segundo (bps) para canales digitales. En la practica, cuando se dice que una conexion tiene un ancho de banda de 1 Gbps, se refiere a la capacidad maxima de transmision en bits por segundo.

El teorema de Nyquist establece que la tasa maxima de transmision sin errores en un canal de ancho de banda B Hz es 2B simbolos por segundo. El teorema de Shannon anade la capa de ruido: C = B * log2(1 + SNR), donde C es la capacidad en bps, B es el ancho de banda y SNR es la relacion senal/ruido.

## Ruido, atenuacion e interferencia

**Ruido:** senales no deseadas que se mezclan con la senal util, degradando su calidad. El ruido puede ser termico (generado por el movimiento termico de electrones), electromagneticas (proveniente de dispositivos cercanos) o de fondo (resente en todos los canales).

**Atenuacion:** la reduccion de la amplitud de la senal a medida que viaja por el medio. Cuanto mas larga es la distancia, mayor es la atenuacion. Los amplificadores y repetidores regeneran la senal para contrarestar la atenuacion.

**Interferencia:** perturbaciones causadas por fuentes externas. En cables de cobre, puede provenir de dispositivos electronicos cercanos o de otros cables. En fibra optica, la interferencia es practicamente inexistente, lo que la convierte en el medio mas inmune a perturbaciones externas.

## Codificacion digital

La codificacion digital determina como se representan bits como senales electricas en un canal. Las tecnicas mas comunes incluyen:

**NRZ (*Non-Return to Zero*):** el bit 1 se representa con un voltaje alto y el 0 con un voltaje bajo. Simple pero problematico: una secuencia larga de ceros o unos causa una deriva de continua que dificulta la sincronizacion.

**Manchester:** cada ciclo de reloj se divide en dos mitades; un 0 se representa con una transicion de alto a bajo en el centro del ciclo, y un 1 de bajo a alto. Proporciona sincronizacion integrada pero duplica el ancho de banda necesario.

**4B/5B:** cada 4 bits de datos se codifican como 5 bits, garantizando que no hay mas de tres ceros consecutivos. Se combina con NRZI para proporcionar sincronizacion eficiente. Util en Fast Ethernet (100BASE-TX).

**8B/10B:** cada 8 bits se codifican como 10 bits, proporcionando transiciones suficientes para la sincronizacion y deteccion de errores basicos. Util en Gigabit Ethernet y Fibre Channel.

## Modulacion

La modulacion es el proceso de variar una senal portadora para codificar informacion:

- **ASK (*Amplitude Shift Keying*):** varia la amplitud de la senal para representar bits. Simple pero susceptible al ruido.
- **FSK (*Frequency Shift Keying*):** varia la frecuencia. Mas robusta que ASK pero requiere mas ancho de banda.
- **PSK (*Phase Shift Keying*):** varia la fase. Eficiente en ancho de banda; utilizada en Wi-Fi.
- **QAM (*Quadrature Amplitude Modulation*):** combina ASK y PSK para lograr alta densidad de bits. QAM-256 (8 bits por simbolo) se usa en Wi-Fi 5 y Ethernet de alta velocidad.

## Multiplexacion

La multiplexacion permite que multiples senales compartan el mismo canal:

- **TDM (*Time Division Multiplexing*):** divide el tiempo en ranuras asignadas a diferentes fuentes. Cada fuente transmite durante su ranura.
- **FDM (*Frequency Division Multiplexing*):** divide el ancho de banda en subbandas, cada una dedicada a una fuente. Util en television por cable y radio.
- **WDM (*Wavelength Division Multiplexing*):** multiplexacion por colores de luz en fibra optica. DWDM (*Dense WDM*) puede transportar cientos de canales en un solo hilo de fibra.

> La capa fisica es donde la innovacion tecnologica tiene mayor impacto en el rendimiento de las redes. Los avances en modulacion QAM, codificacion LDPC y multiplexacion DWDM han permitido multiplicar la capacidad de las redes opticas en las ultimas decadas, acercandose a los limites teoricos establecidos por Shannon.

