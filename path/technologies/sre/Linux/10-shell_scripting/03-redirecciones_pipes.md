# Redirecciones y tuberias

Una de las caracteristicas mas potentes del shell Unix es la capacidad de redirigir flujos de datos entre comandos, archivos y dispositivos. Todo proceso en Unix tiene tres flujos estandar: **stdin** (entrada estandar, descriptor 0), **stdout** (salida estandar, descriptor 1) y **stderr** (salida de error, descriptor 2). Por defecto, stdin proviene del teclado, stdout se muestra en la terminal y stderr tambien se muestra en la terminal. Las redirecciones alteran este comportamiento.

## Redireccion de salida

El operador `>` redirige stdout a un archivo, creandolo si no existe o sobrescribiendolo si existe. El operador `>>` anade la salida al final del archivo sin sobrescribirlo.

```bash
ls -l > listado.txt          # guarda el listado en el archivo
echo "nueva entrada" >> log.txt  # anade una linea al final
```

`2>` redirige stderr a un archivo. `2>>` anade stderr al final. `&>` redirige tanto stdout como stderr a un mismo destino.

```bash
grep "root" /etc/shadow 2> errores.log  # solo stderr al archivo
find / -name "*.conf" &> busqueda.log   # stdout y stderr juntos
```

> El orden de las redirecciones importa. `command 2>&1 > archivo` redirige stderr a stdout (la terminal) y luego stdout al archivo. `command > archivo 2>&1` redirige stdout al archivo y luego stderr a donde apunte stdout (el archivo). La segunda forma es la correcta para unificar ambos flujos.

## Redireccion de entrada

El operador `<` redirige stdin desde un archivo. Es util para pasar contenido a comandos que aceptan entrada estandar.

```bash
sort < palabras.txt                # ordena el contenido del archivo
mail -s "asunto" user@example.com < mensaje.txt  # cuerpo del correo
```

El **here document** (`<<`) permite incrustar bloques de texto directamente en el script sin usar archivos externos:

```bash
cat << EOF > config.txt
server {
    listen 80;
    server_name example.com;
}
EOF
```

## Tuberias (pipes)

El pipe (`|`) conecta el stdout de un comando con el stdin del siguiente. Cada comando en la cadena se ejecuta simultaneamente en un proceso independiente, y los datos fluyen entre ellos sin escribir en disco.

```bash
ps aux | grep apache | awk '{print $2}'
cat /var/log/syslog | grep "ERROR" | cut -d' ' -f1-3 | sort | uniq -c
```

La composicion de comandos mediante pipes es la materializacion de la filosofia Unix: herramientas pequeñas que hacen una cosa bien, combinadas para resolver tareas complejas. No hay un comando "matar procesos de Apache del usuario www-data que consumen mas de 100 MB"; se construye combinando `ps`, `grep`, `awk` y `kill`.

## Combinacion de operadores

Los operadores pueden combinarse en una misma linea para lograr comportamientos complejos:

```bash
./script.sh > salida.log 2>&1              # todo a un archivo
./script.sh 2>&1 | tee salida.log          # pantalla y archivo
./script.sh > /dev/null 2>&1               # descartar toda la salida
```

`/dev/null` es el agujero negro del sistema: todo lo que se escribe en el se descarta. Es util para suprimir mensajes que no interesan. Comprender y dominar las redirecciones y pipes es lo que separa a un usuario novato de uno experimentado en el shell.

