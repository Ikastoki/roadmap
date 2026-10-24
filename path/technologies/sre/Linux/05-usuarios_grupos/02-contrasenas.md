# Administracion de contrasenas

La contrasena es el principal mecanismo de autenticacion en Linux. Historicamente los hashes de las contrasenas se almacenaban en `/etc/passwd`, un archivo legible por todos los usuarios del sistema. Cualquier usuario podia extraer los hashes y someterlos a ataques de fuerza bruta fuera de linea. Para mitigar este riesgo se creo `/etc/shadow`, un archivo de solo lectura para root que almacena los hashes de forma segura.

## passwd

El comando `passwd` gestiona las contrasenas de los usuarios. Cuando un usuario ejecuta `passwd` sin argumentos, cambia su propia contrasena. root puede cambiar la contrasena de cualquier usuario.

```bash
# Cambiar la contrasena del usuario actual
passwd

# Cambiar la contrasena de un usuario especifico (solo root)
sudo passwd maria

# Bloquear la cuenta de un usuario
sudo passwd -l juan

# Desbloquear la cuenta
sudo passwd -u juan

# Eliminar la contrasena (cuenta sin contrasena, solo login sin contrasena)
sudo passwd -d antonio

# Forzar expiracion de contrasena en el proximo inicio de sesion
sudo passwd -e maria

# Ver estado de la contrasena de un usuario
sudo passwd -S juan
```

La salida de `passwd -S` muestra informacion estructurada:

```
juan P 01/15/2026 0 99999 7 -1
```

Los campos indican: nombre de usuario, estado de la contrasena (`P` = usable, `L` = bloqueada, `NP` = sin contrasena), fecha del ultimo cambio, dias minimos entre cambios, dias maximos entre cambios, periodo de advertencia y dias de inactividad antes de deshabilitar la cuenta.

## /etc/shadow

El archivo `/etc/shadow` almacena los hashes de las contrasenas y las politicas de envejecimiento. Solo root y el grupo shadow tienen permiso de lectura. Cada linea representa una cuenta y contiene nueve campos separados por dos puntos:

```
juan:$y$j9T$3kJ2...:20000:0:99999:7:30::
```

| Campo | Significado |
|---|---|
| `juan` | Nombre de usuario |
| `$y$j9T$...` | Hash de la contrasena (algoritmo, salt y hash) |
| `20000` | Dias desde 1970-01-01 del ultimo cambio |
| `0` | Dias minimos entre cambios |
| `99999` | Dias maximos entre cambios |
| `7` | Dias de aviso antes de expirar |
| `30` | Dias de gracia tras expiracion |
| `` | Fecha de expiracion de la cuenta |
| `` | Campo reservado |

### Formatos de hash

El campo del hash puede tener diferentes prefijos que indican el algoritmo:

- `$y$` o `$7$` indica yescrypt (estandar actual en sistemas modernos).
- `$6$` indica SHA-512 (comun en distribuciones antiguas).
- `$5$` indica SHA-256.
- `$2y$` o `$2b$` indica Blowfish/bcrypt.
- `$1$` indica MD5 (obsoleto, inseguro).
- Un `!` o `*` al inicio indica que la cuenta esta bloqueada.

```bash
# Verificar el algoritmo de hash usado en el sistema
sudo head -1 /etc/shadow

# Ver la politica de contrasenas del sistema
sudo cat /etc/login.defs | grep -E "^PASS"
```

### chage

El comando `chage` (change age) proporciona una interfaz mas clara que editar `/etc/shadow` directamente para gestionar el envejecimiento de contrasenas:

```bash
# Ver informacion de envejecimiento de un usuario
sudo chage -l juan

# Forzar cambio de contrasena en el proximo inicio de sesion
sudo chage -d 0 maria

# Establecer fecha de expiracion de la cuenta
sudo chage -E 2026-12-31 temporal

# Establecer dias minimos entre cambios (no se puede cambiar hasta dentro de N dias)
sudo chage -m 7 juan

# Establecer dias maximos entre cambios
sudo chage -M 90 juan

# Establecer dias de aviso antes de la expiracion
sudo chage -W 14 juan
```

> Configurar una politica de envejecimiento de contrasenas (`PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`) en `/etc/login.defs` es mas eficiente que hacerlo usuario por usuario con `chage`.

