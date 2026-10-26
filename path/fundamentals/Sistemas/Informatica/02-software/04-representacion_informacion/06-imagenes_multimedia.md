# Representación de imágenes y multimedia

La representación digital de imágenes, audio y vídeo es uno de los campos más complejos y fascinantes de la informática. Transformar la experiencia visual y auditiva del mundo real en secuencias de bits que una computadora pueda almacenar, procesar y reproducir requiere una combinación de matemáticas, física de la percepción humana y optimización de ingeniería. Los formatos multimedia equilibran calidad, tamaño de archivo y coste de procesamiento de formas que reflejan tanto las limitaciones del hardware como las expectativas de los usuarios.

## Imágenes

Una imagen digital es una cuadrícula (*grid*) de píxeles (*picture elements*), donde cada píxel representa un punto de color. La resolución de la imagen se define por el número de píxeles: una imagen de 1920×1080 contiene 2,073,600 píxeles (aproximadamente 2 megapíxeles).

### Modelos de color

**RGB (*Red, Green, Blue*):** el modelo aditivo utilizado en pantallas y monitores. Cada píxel se define por la intensidad de sus componentes roja, verde y azul, típicamente con valores de 0-255 por canal. La combinación de los tres canales produce el espectro completo de colores visibles.

**CMYK (*Cyan, Magenta, Yellow, Key/Black*):** el modelo sustractivo utilizado en impresión. A diferencia de RGB, que añade luz, CMYK absorbe luz. El negro (K, *Key*) se añade porque la combinación de cian, magenta y amarillo no produce un negro perfecto en la práctica.

### Profundidad de color

La profundidad de color (*color depth*) determina cuántos bits se utilizan para representar el color de cada píxel:

- **8 bits (256 colores):** paleta limitada, utilizada en formatos GIF y sistemas antiguos.
- **16 bits (65,536 colores):** *High Color*, suficiente para muchas aplicaciones básicas.
- **24 bits (16.7 millones de colores):** *True Color*, el estándar actual para fotografía y vídeo.
- **30-48 bits (*Deep Color*):** utilizado en fotografía profesional y edición de vídeo para preservar matices de color.

### Formatos de imagen

**Bitmap (raster):** las imágenes se almacenan como una matriz de píxeles. Cada píxel ocupa un número fijo de bits, lo que proporciona una representación precisa pero que escala mal con el tamaño: una imagen raster de alta resolución puede ocupar muchos megabytes.

**Vectorial:** las imágenes se describen mediante primitivas matemáticas (líneas, curvas, formas) en lugar de píxeles. Los formatos vectoriales (SVG, PDF) escalan sin pérdida de calidad, ya que se renderizan a la resolución deseada en tiempo de ejecución. Son ideales para logotipos, iconos y gráficos técnicos.

### Compresión

**Con pérdida (JPEG):** elimina información que el ojo humano percibe mal, reduciendo drásticamente el tamaño del archivo a costa de una ligera degradación de la calidad. JPEG es ideal para fotografías donde la compresión agresiva es aceptable.

**Sin pérdida (PNG, GIF):** preserva toda la información original de la imagen. PNG es ideal para gráficos con transparencia y texto, donde la pérdida de calidad sería perceptible. GIF soporta animación y paletas de 256 colores.

## Audio

El audio digital se representa mediante *samples* (muestras) de la onda sonora a intervalos regulares. La tasa de muestreo (*sample rate*) determina cuántas muestras se toman por segundo:

- **8 kHz:** calidad telefónica, suficiente para voz hablada.
- **44.1 kHz:** estándar CD de audio, cubre el rango audible humano (20-20,000 Hz).
- **48 kHz:** estándar de vídeo profesional.
- **96-192 kHz:** audio de alta resolución para producción musical.

Cada sample se cuantiza a una resolución de bits determinada (típicamente 16 o 24 bits), que define la precisión de la amplitud de la onda. Un audio de 16 bits y 44.1 kHz ocupa aproximadamente 10 MB por minuto en estéreo sin comprimir.

Los formatos de audio comprimido (MP3, AAC, OGG) utilizan modelos perceptuales para eliminar información que el oído humano no percibe, reduciendo el tamaño del archivo en 10-20 veces con pérdida de calidad mínimamente perceptible.

## Vídeo

El vídeo es esencialmente una secuencia de imágenes (*frames*) mostradas rápidamente para crear la ilusión de movimiento. Los parámetros clave son:

- **Resolución:** el tamaño de cada frame (1920×1080 para Full HD, 3840×2160 para 4K).
- **Tasa de frames (*frame rate*):** cuántos frames se muestran por segundo (24, 30, 60 fps).
- **Tasa de bits (*bitrate*):** la cantidad de datos por unidad de tiempo (Mbps).

La compresión de vídeo es significativamente más compleja que la de imágenes estáticas, ya que aprovecha la redundancia tanto espacial (dentro de cada frame) como temporal (entre frames consecutivos). Los códecs modernos (H.264/AVC, H.265/HEVC, AV1, VP9) utilizan técnicas como predicción de movimiento, transformada coseno discreta (DCT) y compresión entrópica para reducir enormemente el tamaño de archivo sin pérdida de calidad perceptible.

> La elección del códec de vídeo tiene un impacto significativo tanto en la calidad como en los requisitos de hardware. H.265 ofrece una compresión 40-50% mejor que H.264 a cambio de un coste de codificación significativamente mayor. AV1, el estándar abierto más reciente, ofrece compresión aún mejor con coste de codificación comparable a H.265.

