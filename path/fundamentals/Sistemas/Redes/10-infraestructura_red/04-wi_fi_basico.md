# Wi-Fi basico

Wi-Fi es la tecnologia de red inalambrica mas extendida del mundo, basada en el estandar IEEE 802.11. Permite la conectividad movil sin necesidad de cables, lo que la convierte en la forma predominante de acceder a la red en hogares, oficinas y espacios publicos.

Comprender las frecuencias, canales, seguridad y configuracion de Wi-Fi es esencial para disenar redes inalambricas eficientes y seguras. Una mala configuracion Wi-Fi puede causar interferencias, cobertura insuficiente y vulnerabilidades de seguridad.

## Frecuencias: 2.4 GHz vs 5 GHz vs 6 GHz

Wi-Fi opera en bandas de frecuencia de radio que determinan la velocidad, el alcance y la capacidad:

**2.4 GHz:**

- Mayor alcance y mejor penetracion de paredes.
- Menos canales disponibles (1, 6, 11 no superpuestos).
- Mas interferencia (microondas, Bluetooth, dispositivos IoT).
- Velocidades maximas menores (hasta ~600 Mbps con 802.11n).

**5 GHz:**

- Menor alcance y peor penetracion de paredes.
- Mas canales disponibles (36, 40, 44, 48, 52, 56, 60, 64 y DFS).
- Menos interferencia.
- Velocidades mayores (hasta ~3.5 Gbps con 802.11ac).

**6 GHz (Wi-Fi 6E / Wi-Fi 7):**

- Canales adicionales sin superposicion.
- La banda mas limpia (menos dispositivos antiguos).
- Velocidades muy altas (hasta ~5.8 Gbps con Wi-Fi 7).
- Alcance similar a 5 GHz.

## Canales e interferencia

Los canales de Wi-Fi son rangos de frecuencia dentro de una banda. En 2.4 GHz, los canales se superponen significativamente: solo los canales 1, 6 y 11 son completamente independientes. Si dos redes cercanas utilizan canales superpuestos, se interfieren mutuamente.

```
Canal 1:  ████████████████████
Canal 2:   ████████████████████
Canal 3:    ████████████████████
Canal 6:         ████████████████████
Canal 11:                  ████████████████████
```

**Mejores practicas:**

- En 2.4 GHz: usar canales 1, 6 o 11 (los unicos no superpuestos).
- En 5 GHz: elegir canales con menor interferencia (verificar con herramientas de analisis).
- Usar herramientas como `iwlist scan`, `wavemon` o aplicaciones moviles para analizar el espectro.

## Seguridad: WPA2 y WPA3

**WPA2 (*Wi-Fi Protected Access 2*):** estandar de seguridad actual con cifrado AES-CCMP. Es seguro si se utiliza una contrasena fuerte (12+ caracteres, mezcla de tipos). Vulnerable al ataque KRACK (*Key Reinstallation Attack*) en implementaciones antiguas.

**WPA3:** estandar de seguridad reciente con mejoras significativas:

- **SAE (*Simultaneous Authentication of Equals*):** reemplaza al PSK de WPA2, protegiendo contra ataques de fuerza bruta offline.
- **Cifrado de 192 bits** en modo empresarial.
- **Forward secrecy:** si la contrasena se compromete, las comunicaciones pasadas no se pueden descifrar.

**Configuracion recomendada:**

```bash
# Configuracion tipica de Wi-Fi en un router
SSID: MiRed                    # Nombre de la red (no usar datos personales)
Seguridad: WPA3-Personal       # O WPA2-Personal si WPA3 no esta disponible
Contrasena:最少 12 caracteres   # Mezcla de mayusculas, minusculas, numeros, simbolos
Canal: Auto (o 1/6/11 en 2.4 GHz)
Ocultar SSID: No recomendado    # No aporta seguridad real
```

## Resolucion de problemas comunes

- **Cobertura deficiente:** reubicar el router en un punto central, elevarlo (no en el suelo), considerar un extensor o sistema mesh.
- **Interferencia:** cambiar de canal, eliminar fuentes de interferencia cercanas, migrar a 5 GHz o 6 GHz.
- **Velocidad baja:** verificar que no haya muchos dispositivos conectados, comprobar la velocidad del enlace con `iwconfig`, actualizar firmware.
- **Desconexion intermitente:** actualizar controladores Wi-Fi, comprobar energia del adaptador, revisar logs del router.

> Wi-Fi es la tecnologia de red mas conveniente pero tambien la mas propensa a problemas de rendimiento y seguridad. La calidad de la conexion depende de factores ambientales (distancia, obstaculos, interferencias), de la configuracion (canal, seguridad, firmware) y del hardware (calidad del AP y de los adaptadores de los clientes). Una buena red Wi-Fi requiere planificacion, no solo comprar el router mas caro.

