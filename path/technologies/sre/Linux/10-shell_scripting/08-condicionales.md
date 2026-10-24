# Condicionales

Las estructuras condicionales permiten que un script tome decisiones en funcion del estado del sistema, los valores de las variables o el resultado de comandos previos. Sin condicionales, un script ejecutaria siempre las mismas instrucciones, incapaz de adaptarse a entornos cambiantes.

## if, elif, else

La estructura basica es `if ... then ... elif ... then ... else ... fi`. La condicion que sigue a `if` se evalúa como verdadera si su codigo de salida es 0 (exito), y como falsa en cualquier otro caso.

```bash
if [ -f "$archivo" ]; then
    echo "El archivo existe"
elif [ -d "$archivo" ]; then
    echo "Es un directorio"
else
    echo "No existe"
fi
```

La indentacion no es obligatoria en Bash, pero mejora drasticamente la legibilidad. Cada `then` debe ir en su propia linea o precedido de punto y coma (`; then`).

```bash
if [ $edad -ge 18 ]; then
    echo "Mayor de edad"
    puede_votar=true
fi
```

## Condiciones compuestas

Dentro de `[ ]` se usan `-a` (AND) y `-o` (OR). Dentro de `[[ ]]` se usan `&&` y `||`, que resultan mas legibles:

```bash
# Con [ ]
if [ "$1" = "root" -a "$UID" -eq 0 ]; then
    echo "Eres root y el argumento es root"
fi

# Con [[ ]]
if [[ "$1" = "root" && $UID -eq 0 ]]; then
    echo "Eres root y el argumento es root"
fi
```

Las condiciones se pueden agrupar con parentesis (escapados en `[ ]`, naturales en `[[ ]]`):

```bash
if [ "$user" = "admin" ] && ( [ "$mode" = "debug" ] || [ "$mode" = "test" ] ); then
    echo "Acceso concedido"
fi
```

## Evaluacion de comandos directamente

No es necesario limitarse a `test`. Cualquier comando puede actuar como condicion, ya que `if` evalua el codigo de salida:

```bash
if grep -q "root" /etc/passwd; then
    echo "El usuario root existe"
fi

if ping -c 1 -W 1 8.8.8.8 &>/dev/null; then
    echo "Hay conectividad"
else
    echo "Sin conexion"
fi
```

> La opcion `-q` de grep suprime la salida, haciendo que grep se comporte como una condicion pura: codigo 0 si encuentra el patron, 1 si no. Esta tecnica se aplica a muchos comandos que devuelven codigos de salida significativos.

## case

La estructura `case` reemplaza multiples `if/elif` cuando se compara una misma variable contra varios valores posibles. Es mas legible y ligeramente mas eficiente:

```bash
case "$1" in
    start)
        echo "Iniciando servicio..."
        ;;
    stop)
        echo "Deteniendo servicio..."
        ;;
    restart|reload)
        echo "Reiniciando servicio..."
        ;;
    *)
        echo "Uso: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

Los patrones en `case` pueden usar comodines glob (`*`, `?`, `[a-z]`) y tuberias para multiples opciones. El `*` final actua como "captura todo" (por defecto o error). Cada bloque termina con `;;`, que indica el final de ese patron.

## Doble parentesis en condicionales

Las expresiones aritmeticas con `(( ))` tambien pueden actuar como condicion directamente, sin necesidad de `test`:

```bash
a=10
b=20
if (( a + b > 25 )); then
    echo "La suma supera 25"
fi
```

La combinacion de `(( ))` con `if` produce un codigo mas limpio para comprobaciones numericas que usar `[ "$a" -gt "$b" ]`.

## Buenas practicas

- Entrecomillar siempre las variables en `[ ]`: `[ "$var" = "valor" ]` (evita errores si la variable esta vacia).
- Preferir `[[ ]]` sobre `[ ]` cuando Bash este garantizado.
- Usar `case` en lugar de cadenas de `elif` cuando se compara contra patrones fijos.
- Terminar el script con `exit 0` si todo es correcto, `exit 1` (o mayor) si hubo errores.

