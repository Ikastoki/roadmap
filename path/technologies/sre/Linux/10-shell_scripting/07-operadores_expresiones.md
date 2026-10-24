# Operadores y expresiones

Bash proporciona varios mecanismos para evaluar expresiones aritmeticas, comparar valores y comprobar condiciones. Cada uno tiene una sintaxis y un proposito especifico, y usarlos correctamente es esencial para escribir scripts logicos y funcionales.

## Aritmetica con (( ))

La construccion `(( ))` evalua expresiones aritmeticas enteras. Soporta los operadores clasicos: `+`, `-`, `*`, `/`, `%` (modulo), `**` (potencia), ademas de operadores de incremento/decremento (`++`, `--`) y asignacion compuesta (`+=`, `-=`, etc.).

```bash
a=5
b=3
(( suma = a + b ))
(( producto = a * b ))
(( exponente = a ** b ))
echo "Suma: $suma"
echo "Producto: $producto"
(( a++ ))
echo "Ahora a vale: $a"
```

Dentro de `(( ))` las variables no necesitan el prefijo `$`. El resultado de la expresion final puede usarse como condicion: si el resultado es distinto de cero, se considera verdadero; si es cero, falso.

```bash
if (( a > 10 )); then
    echo "a es mayor que 10"
fi
```

## Comparaciones con test y [ ]

El comando `test` evalua expresiones condicionales. Su sintaxis alternativa con corchetes `[ ]` es semanticamente identica y mucho mas utilizada. La condicion se evalúa y devuelve un codigo de salida: 0 si es verdadera, 1 si es falsa.

```bash
if [ "$a" -eq "$b" ]; then
    echo "Iguales"
fi
```

### Comparaciones numericas

- `-eq` igual que (`equal`)
- `-ne` distinto de (`not equal`)
- `-lt` menor que (`less than`)
- `-le` menor o igual que
- `-gt` mayor que (`greater than`)
- `-ge` mayor o igual que

### Comparaciones de cadenas

- `=` o `==` igualdad de cadenas (dentro de `[ ]`)
- `!=` desigualdad
- `<` y `>` menor/mayor alfabeticamente (requieren escapado `\<` en `[ ]`, o usar `[[ ]]`)
- `-z` cadena vacia (longitud cero)
- `-n` cadena no vacia

```bash
[ "$nombre" = "root" ] && echo "Es root"
[ -z "$1" ] && echo "No hay argumentos"
```

### Comprobaciones de archivos

- `-f` existe y es un archivo regular
- `-d` existe y es un directorio
- `-e` existe (cualquier tipo)
- `-s` existe y no esta vacio
- `-r` existe y tiene permiso de lectura
- `-w` existe y tiene permiso de escritura
- `-x` existe y tiene permiso de ejecucion
- `-L` existe y es un enlace simbolico

```bash
if [ -f "/etc/passwd" ]; then
    echo "El archivo existe"
fi
if [ ! -d "$HOME/backups" ]; then
    mkdir "$HOME/backups"
fi
```

## La version extendida [[ ]]

`[[ ]]` es una version mejorada de `[ ]` disponible en Bash (no en POSIX). Sus ventajas incluyen:

- No requiere escapado de `<` y `>` para comparaciones lexicograficas.
- Soporta `&&` y `||` como operadores logicos internos (en `[ ]` se usan `-a` y `-o`).
- Permite patrones glob y expresiones regulares con `=~`.
- Maneja correctamente cadenas vacias sin necesidad de entrecomillar.

```bash
[[ "$archivo" = *.txt ]] && echo "Es un archivo de texto"
[[ $edad -ge 18 && $edad -le 65 ]] && echo "Adulto en edad laboral"
[[ "$email" =~ ^[a-z]+@[a-z]+\.[a-z]{2,}$ ]] && echo "Email valido"
```

> Aunque `[[ ]]` es mas potente y seguro, los scripts que deben ser portables a otras shells (como Dash) deben usar `[ ]`. En entornos donde Bash esta garantizado, `[[ ]]` es siempre la opcion recomendada.

## Operadores logicos

- `!` negacion
- `&&` Y logico (AND)
- `||` O logico (OR)

```bash
if [ ! -d "$DIR" ] || [ -z "$(ls -A "$DIR")" ]; then
    echo "Directorio no existe o esta vacio"
fi
```

