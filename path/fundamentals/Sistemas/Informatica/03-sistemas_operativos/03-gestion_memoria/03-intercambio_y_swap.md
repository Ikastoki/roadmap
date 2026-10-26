# Intercambio y swap

El intercambio (*swapping*) es el mecanismo por el cual el sistema operativo mueve procesos completos o páginas individuales de memoria RAM al disco y viceversa. Cuando la memoria física se llena y se necesita asignar espacio para nuevos procesos o páginas, el SO selecciona víctimas en la memoria y las escribe en un área de intercambio (*swap space*) en el disco, liberando marcos de memoria para otros usos.

El swap es esencial para la multiprogramación eficiente: permite que el sistema ejecute más procesos de los que caben en RAM simultáneamente. Sin embargo, el acceso al disco es órdenes de magnitud más lento que el acceso a RAM (milisegundos frente a nanosegundos), por lo que un uso excesivo de swap degrada significativamente el rendimiento del sistema.

## Swap space

El *swap space* es una partición especial del disco o un archivo reservado que se utiliza como extensión de la memoria RAM. En Linux, se puede crear un archivo de intercambio con `mkswap` y activarlo con `swapon`:

```bash
# Crear archivo de swap de 4 GB
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Verificar el estado del swap
swapon --show
free -h
```

La cantidad de swap recomendada depende del uso del sistema. Para servidores con poca memoria, se recomienda al menos el doble de la RAM. Para sistemas de escritorio modernos con suficiente RAM (16 GB o más), un swap pequeño (2-4 GB) puede ser suficiente.

## Algoritmos de reemplazo de páginas

Cuando la memoria está llena y se necesita un marco libre, el SO debe decidir qué página reemplazar. Los algoritmos más comunes incluyen:

**FIFO (*First-In, First-Out*):** reemplaza la página más antigua (la que llegó primero a la memoria). Es simple pero no tiene en cuenta el patrón de acceso: puede reemplazar una página muy usada solo porque lleva mucho tiempo en memoria.

**LRU (*Least Recently Used*):** reemplaza la página que lleva más tiempo sin ser accedida. Es una buena aproximación de la localidad temporal: las páginas recientemente usadas probablemente se usarán de nuevo pronto. Sin embargo, la implementación exacta es costosa (requiere un timestamp o contador para cada acceso).

**Clock (algoritmo del reloj):** es una aproximación eficiente de LRU utiliza un bit de referencia para cada página. Un puntero circular recorre las páginas; si una página tiene el bit de referencia en 1, se pone en 0 y avanza; si está en 0, se reemplaza. Es el algoritmo más utilizado en la práctica por su buen balance entre rendimiento y eficacia.

**LFU (*Least Frequently Used*):** reemplaza la página con menor conteo de accesos. Es útil cuando las páginas con mayor frecuencia de uso deben permanecer en memoria, pero tiene el problema de las páginas antiguas que acumularon muchos accesos pero ya no se usan.

## Thrashing (intercambio excesivo)

El *thrashing* ocurre cuando el sistema dedica más tiempo a intercambiar páginas entre memoria y disco que a ejecutar instrucciones útiles. Se produce cuando la suma de losWorking Sets de todos los procesos activos excede significativamente la memoria física disponible.

El *Working Set* de un proceso es el conjunto de páginas que está usando activamente en un momento dado. Si el sistema puede mantener en memoria al menos el Working Set de cada proceso, el rendimiento es aceptable. Cuando la memoria disponible es insuficiente para todos los Working Sets, los procesos experimentan un alto número de page faults y comienza el thrashing.

Las señales de thrashing incluyen alta utilización de CPU combinada con alta actividad de disco, y un rendimiento del sistema dramáticamente degradado. Las estrategias para combatirlo incluyen reducir el grado de multiprogramación (matar procesos) o aumentar la memoria física.

> El rendimiento del sistema depende críticamente de la cantidad de RAM disponible. Cuando la memoria física es insuficiente para contener los Working Sets de los procesos activos, el sistema degrada rápidamente al thrashing, donde la mayoría del tiempo se gasta en mover páginas entre RAM y disco. Un diagnóstico de rendimiento siempre debe incluir la revisión del uso de swap con herramientas como `vmstat`, `sar` o `free`.

