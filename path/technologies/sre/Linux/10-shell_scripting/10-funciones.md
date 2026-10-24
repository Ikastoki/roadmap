# Funciones

Las funciones permiten agrupar bloques de codigo bajo un nombre y reutilizarlos sin duplicacion. Son el mecanismo basico de modularizacion en shell scripting y, usadas correctamente, transforman scripts largos y repetitivos en programas modulares, faciles de mantener y probar.

## Declaracion

Bash soporta dos sintaxis para declarar funciones. La mas portable y recomendada es usar `nombre() { ... }`. La palabra clave `function` es opcional y especifica de Bash:

```bash
# Forma estandar (portable)
saludar() {
    echo "Hola, $1"
}

# Forma alternativa (Bash-only)
function despedir {
    echo "Adios, $1"
}
```

La funcion se invoca escribiendo su nombre como si fuera un comando cualquiera, sin parentesis:

```bash
saludar "Juan"
despedir "Maria"
```

## Argumentos en funciones

Dentro de una funcion, `$1`, `$2`, etc. se refieren a los argumentos pasados a la funcion, no al script principal. `$#` es el numero de argumentos de la funcion, y `$@` la lista completa de esos argumentos.

```bash
suma() {
    local total=$(( $1 + $2 ))
    echo $total
}

resultado=$(suma 5 3)
echo "Resultado: $resultado"
```

## Variables locales

Por defecto, las variables en una funcion son globales y afectan al script principal. La palabra clave `local` restringe el ambito de una variable a la funcion y sus hijas:

```bash
#!/bin/bash
variable="global"

mi_funcion() {
    local variable="local"
    echo "Dentro: $variable"
}

mi_funcion
echo "Fuera: $variable"
# Salida:
# Dentro: local
# Fuera: global
```

El uso de `local` evita efectos colaterales y es una de las buenas practicas mas importantes en scripts con funciones. Una funcion bien escrita deberia declarar todas sus variables como locales, a menos que exista una razon explicita para modificarlas globalmente.

## Retorno de valores

Bash no permite devolver valores arbitrarios directamente. Existen tres mecanismos para obtener resultados de una funcion:

1. **`return`**: devuelve un codigo numerico entre 0 y 255, interpretable como codigo de salida. Analogamente a `exit` en el script, `0` indica exito y cualquier otro valor indica error.

```bash
es_par() {
    [ $(( $1 % 2 )) -eq 0 ] && return 0 || return 1
}

if es_par 4; then
    echo "Es par"
fi
```

2. **`echo`**: imprime el resultado, que puede capturarse con sustitucion de comandos `$()`.

```bash
cuadrado() {
    echo $(( $1 * $1 ))
}

valor=$(cuadrado 5)
echo "El cuadrado es $valor"
```

3. **Variables globales o de referencia**: se modifican variables definidas fuera de la funcion. Este enfoque es menos puro pero util cuando se necesita devolver multiples valores.

```bash
dividir() {
    resultado=$(( $1 / $2 ))
    resto=$(( $1 % $2 ))
}

dividir 10 3
echo "Cociente: $resultado, Resto: $resto"
```

## Modularizacion

Las funciones se pueden agrupar en archivos separados e importarse con `source` (o `.`):

```bash
# utilidades.sh
log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1"
}
log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2
}
```

```bash
#!/bin/bash
source ./utilidades.sh

log_info "Iniciando proceso"
log_error "Fallo en la conexion"
```

`source` ejecuta el contenido del archivo en el shell actual, haciendo que las funciones y variables definidas esten disponibles. Es el equivalente a `import` en otros lenguajes.

## Buenas practicas con funciones

- Cada funcion debe hacer una sola cosa (principio de responsabilidad unica).
- Nombrar las funciones con verbos: `calcular_media`, `validar_usuario`, `limpiar_temporales`.
- Declarar siempre las variables como `local` dentro de la funcion.
- Documentar el proposito y los parametros con comentarios al inicio de cada funcion.
- Mantener las funciones cortas (menos de 50 lineas).
- Usar `return` para indicar exito/fallo y `echo` para devolver datos.

