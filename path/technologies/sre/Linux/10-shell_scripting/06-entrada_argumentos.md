# Entrada del usuario y argumentos

Un script que no recibe datos externos tiene una utilidad limitada. Bash proporciona dos mecanismos principales para que un script interactue con el usuario o reciba datos: los argumentos posicionales y el comando `read`.

## Argumentos posicionales

Cuando se invoca un script, los argumentos escritos en la linea de comandos quedan disponibles en las variables especiales `$1`, `$2`, `$3`... hasta `$9`. Para acceder al decimo argumento o superior se usan llaves: `${10}`.

```bash
#!/bin/bash
echo "Script: $0"
echo "Destino: $1"
echo "Origen: $2"
```

Si se ejecuta `./copia.sh /backup /home/usuario/docs`, `$1` contiene `/backup` y `$2` contiene `/home/usuario/docs`. El comportamiento esperado es que el usuario conozca el orden de los argumentos. Para documentarlo, el script debe mostrar un mensaje de uso cuando los argumentos no son suficientes.

## shift

El comando `shift` desplaza los argumentos posicionales una posicion a la izquierda: `$2` pasa a ser `$1`, `$3` pasa a `$2`, y `$1` se descarta. Es util para procesar argumentos en un bucle cuando no se conoce el numero exacto de parametros.

```bash
#!/bin/bash
while [ $# -gt 0 ]; do
    echo "Procesando: $1"
    shift
done
```

Cada `shift` reduce `$#` en uno. Cuando se combina con opciones, permite implementar parsing manual de argumentos estilo `-f archivo -v`.

## El comando read

`read` captura la entrada del usuario desde stdin y la asigna a una o mas variables. Si no se especifica ninguna variable, los datos se guardan en la variable predeterminada `REPLY`.

```bash
read -p "Introduce tu nombre: " nombre
echo "Hola, $nombre"
```

Opciones utiles de `read`:

- `-p "texto"` muestra un prompt antes de leer.
- `-s` modo silencioso (no muestra lo que se escribe, util para contrasenas).
- `-t N` timeout de N segundos.
- `-n N` lee solo N caracteres.
- `-r` modo raw (no interpreta barras invertidas como escapes).

```bash
read -s -p "Contrasena: " pass
echo
read -t 5 -p "Confirmar en 5 segundos (s/n): " -n 1 confirm
```

La combinacion de `read` con una sola variable captura toda la linea. Si se pasan multiples variables, el shell divide la entrada por el separador IFS (Internal Field Separator, que por defecto incluye espacio, tabulador y nueva linea):

```bash
echo "uno dos tres" | read a b c
# a=uno, b=dos, c=tres
```

## Validacion de entrada

Los scripts robustos deben verificar que los datos recibidos son validos antes de usarlos. La validacion basica comprueba si una variable esta vacia, si es un numero o si un archivo existe:

```bash
if [ -z "$1" ]; then
    echo "Error: falta el argumento"
    exit 1
fi

if ! [[ "$2" =~ ^[0-9]+$ ]]; then
    echo "Error: el segundo argumento debe ser un numero"
    exit 1
fi

if [ ! -f "$3" ]; then
    echo "Error: el archivo $3 no existe"
    exit 1
fi
```

> Una buena practica en scripts profesionales es incluir siempre una funcion `uso()` que muestre la sintaxis esperada y llame a `exit 1` cuando los argumentos no sean correctos. Esto proporciona retroalimentacion inmediata al usuario sin necesidad de leer el codigo fuente.

## Procesamiento combinado

Los argumentos posicionales y `read` no son excluyentes. Un script tipico acepta argumentos para modo batch y usa `read` para modo interactivo:

```bash
#!/bin/bash
if [ $# -eq 0 ]; then
    read -p "Archivo a procesar: " archivo
else
    archivo="$1"
fi
echo "Procesando: $archivo"
```

