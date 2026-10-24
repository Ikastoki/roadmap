# Permisos de archivos

El modelo de permisos de Linux es uno de los pilares de su seguridad. Cada archivo y directorio lleva asociado un conjunto de bits que determinan quien puede leerlo, escribirlo o ejecutarlo. Este sistema de control de acceso discrecional (DAC) permite que el propietario de un recurso decida los privilegios que otorga al resto del sistema, sentando las bases para la contencion de usuarios y procesos en entornos multiusuario.

## Desglose de los 9 bits de permiso

Cuando se ejecuta `ls -l`, la primera columna muestra una cadena de 10 caracteres. El primer caracter indica el tipo de archivo, y los 9 siguientes representan los permisos en tres grupos de tres bits cada uno.

```
-rwxr-xr--
```

- Posicion 1: tipo de archivo (`-`, `d`, `l`, etc.)
- Posiciones 2-4: permisos del **usuario propietario** (u)
- Posiciones 5-7: permisos del **grupo propietario** (g)
- Posiciones 8-10: permisos de **otros usuarios** (o)

Cada trio de bits representa, en orden: lectura (r), escritura (w), ejecucion (x). Si el permiso no esta concedido, aparece un guion `-`.

| Bit | Usuario (u) | Grupo (g) | Otros (o) |
|-----|-------------|-----------|-----------|
| r   | `r--` (4)   | `r--` (4) | `r--` (4) |
| w   | `-w-` (2)   | `-w-` (2) | `-w-` (2) |
| x   | `--x` (1)   | `--x` (1) | `--x` (1) |

Para un directorio, el significado de los bits varia ligeramente:
- **r (lectura)**: permite listar el contenido del directorio.
- **w (escritura)**: permite crear, renombrar o eliminar archivos dentro del directorio.
- **x (ejecucion/atravesar)**: permite acceder a archivos dentro del directorio y atravesarlo (necesario para usar `cd`).

> En un directorio, tener permiso de lectura sin ejecucion permite ver los nombres de los archivos pero no acceder a sus metadatos ni a su contenido. Tener ejecucion sin lectura permite acceder a archivos conocidos por su nombre pero no listar el contenido del directorio. Ambos son necesarios para un uso normal del directorio.

## Cambio de permisos con chmod

`chmod` (change mode) modifica los permisos de archivos y directorios. Acepta dos formas de notacion: octal y simbolica.

### Notacion octal

Cada grupo de tres bits se representa con un digito octal (0-7), resultado de sumar los valores de los bits activados:

| Octal | Binario | Permisos  | Significado             |
|-------|---------|-----------|-------------------------|
| 7     | 111     | `rwx`     | Lectura, escritura, ejecucion |
| 6     | 110     | `rw-`     | Lectura y escritura     |
| 5     | 101     | `r-x`     | Lectura y ejecucion     |
| 4     | 100     | `r--`     | Solo lectura            |
| 3     | 011     | `-wx`     | Escritura y ejecucion   |
| 2     | 010     | `-w-`     | Solo escritura          |
| 1     | 001     | `--x`     | Solo ejecucion          |
| 0     | 000     | `---`     | Sin permisos            |

El modo completo se expresa con tres digitos: `chmod 755 archivo`. El primer digito corresponde al usuario, el segundo al grupo, el tercero a otros.

```bash
chmod 755 script.sh    # rwxr-xr-x (usuario: todo, grupo/otros: r-x)
chmod 644 documento.txt # rw-r--r-- (usuario: rw, grupo/otros: r)
chmod 700 directorio    # rwx------ (solo el propietario)
chmod 600 clave.pem     # rw------- (clave privada SSH)
```

Combinaciones tipicas:
- `755`: ejecutables publicos (scripts, binarios del sistema).
- `644`: archivos de texto, configuracion, codigo fuente.
- `700`: directorios personales (solo el propietario accede).
- `600`: archivos sensibles (claves, tokens).
- `777`: todos los permisos para todos (desaconsejado por seguridad).

### Notacion simbolica

La notacion simbolica utiliza operadores para modificar permisos de forma relativa:

```
chmod [referencia][operador][permiso] archivo
```

**Referencias**: `u` (usuario), `g` (grupo), `o` (otros), `a` (todos, por defecto).

**Operadores**: `+` (anade permiso), `-` (quita permiso), `=` (asigna exactamente).

```bash
chmod u+x script.sh       # Anade ejecucion al usuario
chmod g-w archivo.txt     # Quita escritura al grupo
chmod o-r clave.pem       # Quita lectura a otros
chmod a+x script.sh       # Anade ejecucion a todos
chmod u=rw,g=r,o= archivo # Asigna permisos exactos (igual que 640)
chmod g+w,o-r documento   # Combinacion de operaciones
```

La notacion simbolica es preferible cuando solo se desea modificar un permiso concreto sin redefinir el resto, mientras que la notacion octal es mas adecuada para establecer el modo completo de forma precisa.

## Permisos especiales: SUID, SGID y sticky bit

Ademas de los 9 bits basicos, existen tres bits de permiso especial que alteran el comportamiento de ejecucion y herencia.

### SUID (Set User ID)

Cuando un archivo ejecutable tiene el bit SUID activado, se ejecuta con los permisos del *propietario* del archivo, no del usuario que lo ejecuta. Esto permite que usuarios sin privilegios ejecuten tareas que requieren permisos elevados de forma controlada.

```bash
chmod u+s /usr/bin/programa    # Notacion simbolica
chmod 4755 /usr/bin/programa   # Notacion octal (4 delante)
```

El ejemplo mas clasico es `/usr/bin/passwd`, que necesita acceso privilegiado para modificar `/etc/shadow`:

```bash
ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 nov 12  2023 /usr/bin/passwd
```

La `s` en la posicion de ejecucion del usuario (en lugar de `x`) indica SUID activado.

> SUID es un mecanismo peligroso si se aplica incorrectamente. Un binario con SUID root que contenga una vulnerabilidad puede permitir a un usuario sin privilegios escalar a superusuario. Por esta razon, el numero de binarios con SUID en un sistema debe minimizarse.

### SGID (Set Group ID)

En un archivo ejecutable, SGID funciona igual que SUID pero heredando los permisos del *grupo* propietario. En un directorio, SGID hace que los nuevos archivos y subdirectorios creados dentro hereden el grupo del directorio padre, en lugar del grupo primario del usuario que los crea.

```bash
chmod g+s /compartido/proyecto     # Notacion simbolica
chmod 2775 /compartido/proyecto    # Notacion octal (2 delante)
```

```bash
ls -ld /compartido/proyecto
drwxrwsr-x 2 root desarrolladores 4096 jul  1 12:00 /compartido/proyecto
```

La `s` en la posicion de ejecucion del grupo indica SGID activado. Este comportamiento es util en directorios compartidos por multiples usuarios de un mismo equipo de trabajo.

### Sticky bit

El sticky bit en un directorio restringe la eliminacion de archivos: solo el propietario del archivo, el propietario del directorio o el superusuario pueden eliminar o renombrar archivos dentro de ese directorio, independientemente de los permisos de escritura del directorio.

```bash
chmod +t /tmp            # Notacion simbolica
chmod 1777 /tmp          # Notacion octal (1 delante)
```

```bash
ls -ld /tmp
drwxrwxrwt 20 root root 4096 jul  1 12:00 /tmp
```

La `t` en la posicion de ejecucion de otros indica sticky bit activado. El directorio `/tmp` es el ejemplo por excelencia: todos los usuarios tienen permiso de escritura, pero no pueden borrar los archivos de otros usuarios.

### Visualizacion de permisos especiales

Los bits especiales aparecen en la salida de `ls -l` en las posiciones de ejecucion:
- SUID: `s` en lugar de `x` en la terna de usuario (`rwsr-xr-x`).
- SGID: `s` en lugar de `x` en la terna de grupo (`rwxr-sr-x`).
- Sticky: `t` en lugar de `x` en la terna de otros (`rwxrwxrwt`).

Si el permiso de ejecucion correspondiente no esta activado, la letra se muestra en mayuscula (`S` o `T`), indicando que el bit especial esta activo pero el permiso de ejecucion base no lo esta (caso inusual).

