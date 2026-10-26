# Flujo de aire

El flujo de aire dentro de un chasis de PC es el factor que determina si la refrigeración por aire funciona de forma efectiva. Un disipador de CPU de alta gama con ventiladores potentes puede ser ineficiente si el chasis no proporciona un flujo de aire adecuado que renueve el aire caliente y suministre aire fresco. La configuración de los ventiladores del chasis, la presión de aire y la gestión de cables son elementos que influyen directamente en la temperatura de los componentes.

Diseñar un flujo de aire eficiente requiere comprender cómo se mueve el aire dentro del chasis, dónde se acumulan las bolsas de aire caliente y cómo las presiones positiva y negativa afectan al patrón de flujo. Un sistema con un flujo de aire bien diseñado puede reducir las temperaturas entre 5-15°C respecto a uno con una configuración deficiente, sin cambiar ningún componente de refrigeración.

## Configuración de ventiladores

La configuración estándar de ventiladores en un chasis de PC sigue un patrón de entrada y salida:

**Entrada (*intake*):** ventiladores montados en la parte frontal o inferior del chasis que introducen aire fresco del exterior. El aire fresco debe llegar directamente a los componentes que más calor generan: la GPU y la CPU.

**Salida (*exhaust*):** ventiladores montados en la parte trasera o superior del chasis que expulsan el aire caliente del interior. El aire caliente tiende a subir por convección, lo que hace que los ventiladores superiores sean particularmente efectivos.

Una configuración típica para un chasis ATX incluye:

- 2 ventiladores frontales de entrada (120-140 mm)
- 1 ventilador trasero de salida (120-140 mm)
- 1-2 ventiladores superiores de salida (120-140 mm)

Esta configuración crea un flujo de aire que entra por la frontal, pasa sobre los componentes y sale por la trasera y superior. La zona más crítica es la del compartimento de la CPU/GPU, donde los ventiladores deben proporcionar aire fresco directamente a los disipadores.

> La instalación de demasiados ventiladores no siempre mejora la refrigeración. Un exceso de ventiladores de entrada puede crear turbulencias que interfieran con el flujo de aire del disipador de la CPU, y un exceso de ventiladores de salida puede crear zonas de baja presión que atraigan polvo por las aberturas no filtradas del chasis.

## Presión positiva

Un sistema tiene presión positiva cuando la cantidad de aire introducido es mayor que la cantidad de aire extraído. Esto provoca que el aire sea empujado hacia el exterior del chasis a través de todas las aberturas, creando un flujo unidireccional que dificulta la entrada de polvo.

La presión positiva se logra instalando más ventiladores de entrada que de salida, o configurando los ventiladores de entrada con un CFM mayor que los de salida. Los beneficios incluyen:

- **Reducción de polvo:** el aire es empujado hacia afuera por las aberturas del chasis, dificultando la entrada de polvo por las grietas.
- **Flujo predecible:** el patrón de flujo es más consistente y fácil de gestionar.
- **Protección de componentes:** los componentes que no tienen disipadores propios (como la placa base y la memoria) reciben aire fresco del exterior.

## Presión negativa

Un sistema tiene presión negativa cuando se extrae más aire del que se introduce. Esto crea una depresión que succiona aire del exterior a través de todas las aberturas disponibles, incluyendo las no filtradas.

La presión negativa se logra instalando más ventiladores de salida que de entrada, o configurando los ventiladores de salida con un CFM mayor que los de entrada. Los beneficios incluyen:

- **Evacuación rápida del aire caliente:** el aire caliente se expulsa más eficientemente, particularmente de la zona superior del chasis.
- **Refrigeración direccional:** puede ser útil en configuraciones donde la GPU es el componente más crítico y se desea maximizar el flujo de aire sobre ella.

Sin embargo, la presión negativa tiene un inconveniente significativo: el aire entra por todas las aberturas del chasis, incluyendo las no filtradas, lo que aumenta la acumulación de polvo en los componentes.

## Presión neutra

La presión neutra ocurre cuando la cantidad de aire introducido es igual a la cantidad de aire extraído. En teoría, esto equilibra los beneficios de ambas configuraciones, pero en la práctica, la presión neutra es difícil de mantener exactamente debido a las diferencias en la resistencia al flujo de aire de los ventiladores y las obstrucciones del chasis.

La mayoría de los fabricantes de chasis recomiendan configuraciones que tienden ligeramente hacia la presión positiva, ya que los beneficios de la reducción de polvo suelen ser preferibles a la ligera mejora en la evacuación del aire caliente que ofrece la presión negativa.

