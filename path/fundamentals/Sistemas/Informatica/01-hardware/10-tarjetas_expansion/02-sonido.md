# Tarjetas de sonido dedicadas

Las tarjetas de sonido dedicadas son circuitos de expansión especializados en la captura, procesamiento y reproducción de audio de alta calidad. Aunque el audio integrado en las placas base modernas ha mejorado enormemente y es adecuado para la mayoría de los usuarios, las tarjetas de sonido dedicadas siguen siendo preferidas por profesionales del audio, músicos, productores y entusiastas que buscan una calidad de sonido superior, menor interferencia eléctrica y funcionalidades específicas que el hardware integrado no puede ofrecer.

La diferencia fundamental entre el audio integrado y una tarjeta dedicada radica en la separación física de los circuitos de audio del resto de la placa base, donde la proximidad a fuentes de interferencia electromagnética —procesador, memoria RAM, reguladores de voltaje— degrada la calidad de la señal de audio.

## Ventajas sobre el audio integrado

El audio integrado en la placa base comparte el espacio físico con el resto de los circuitos electrónicos, lo que lo expone a interferencias electromagnéticas que pueden manifestarse como zumbidos (*hum*), siseos (*hiss*) o distorsión en la señal de audio. Las tarjetas de sonido dedicadas, al estar físicamente separadas en su propio slot PCIe, eliminan gran parte de estas interferencias.

Además, las tarjetas de sonido dedicadas incluyen componentes de mayor calidad que el audio integrado: condensadores de audio de alta gama, osciladores de baja interferencia y diseños de circuito optimizados para la señal de audio. Estos componentes proporcionan una relación señal/ruido (*SNR*) significativamente superior y una distorsión armónica total (*THD*) menor.

Las ventajas prácticas incluyen:

- **Mejor calidad de salida:** mayor claridad, rango dinámico más amplio y mejor separación de canales.
- **Menor latencia de audio:** los drivers especializados optimizan la cadena de procesamiento de audio para reducir la latencia, crítica en aplicaciones de producción musical en tiempo real.
- **Entradas y salidas adicionales:** puertos S/PDIF ópticos, entradas de micrófono de alta impedancia, salidas balanceadas para monitores de estudio y conectores MIDI.
- **Procesamiento de hardware:** DSP (*Digital Signal Processor*) dedicado que ejecuta efectos de audio, ecualización y mezcla sin carga del procesador principal.

## DAC, SNR, amplificadores de auriculares

**DAC (*Digital-to-Analog Converter*):** el conversor digital-analógico es el componente que transforma la señal de audio digital (una secuencia de números) en una señal analógica continua que los altavoces o auriculares pueden reproducir. La calidad del DAC determina en gran medida la calidad del sonido producido. Los DAC integrados en el audio suelen ser de calidad básica, mientras que las tarjetas de sonido dedicadas incluyen DAC de referencia de fabricantes como ESS, Cirrus Logic o Burr-Brown.

**SNR (*Signal-to-Noise Ratio*):** la relación señal/ruido mide la diferencia en decibelios (dB) entre la señal de audio útil y el ruido de fondo del circuito. Un SNR de 100 dB significa que la señal es 100,000 veces más potente que el ruido. Las tarjetas de sonido de gama alta alcanzan SNR de 120-140 dB, mientras que el audio integrado típicamente ronda los 90-100 dB. Cada 6 dB de mejora en SNR corresponde a una reducción del ruido a la mitad.

**Amplificador de auriculares (*headphone amplifier*):** los auriculares de alta impedancia (250-600 ohmios) requieren una potencia de salida significativamente mayor que los auriculares estándar (32 ohmios). El amplificador de auriculares integrado en una tarjeta de sonido dedicada puede entregar suficiente potencia y voltaje para excitar auriculares de alta impedancia sin distorsión, algo que el amplificador integrado en la mayoría de placas base no puede lograr.

> Para la mayoría de usuarios que utilizan auriculares o altavoces estándar, el audio integrado moderno es más que suficiente. Las tarjetas de sonido dedicadas marcan una diferencia perceptible principalmente con auriculares de alta gama, sistemas de altavoces de estudio o en aplicaciones de producción profesional.

