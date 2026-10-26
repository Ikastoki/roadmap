# Autenticacion y autorizacion

La seguridad de un sistema operativo comienza con la verificacion de identidad y el control de acceso. La autenticacion verifica quien es el usuario; la autorizacion determina que puede hacer una vez autenticado. Estos dos conceptos son la base de la seguridad en cualquier sistema multiusuario, desde un servidor Linux hasta una estacion de trabajo.

Un fallo en la autenticacion puede comprometer todo el sistema, independientemente de lo robustos que sean los demas mecanismos de seguridad. Si un atacante obtiene credenciales validas, toda la infraestructura de defensa se vuelve irrelevante.

## Autenticacion basada en contrasenas

La forma mas comun de autenticacion es la contrasena. Cuando un usuario introduce su contrasena, el sistema la compara con un hash almacenado en `/etc/shadow` (nunca en texto plano). El proceso utiliza un salt (cadena aleatoria) para proteger contra ataques de tabla arcoiris.

```bash
# Formato de /etc/shadow (solo root puede leerlo)
usuario:$6$salt$hash:18000:0:99999:7:::

# Crear usuario con contrasena encriptada
useradd -m -p $(openssl passwd -6 "contrasena") nuevo_usuario

# Cambiar contrasena
passwd usuario
```

La seguridad de las contrasenas depende de varios factores: longitud minima recomendada de 12 caracteres, complejidad (mayusculas, minusculas, numeros, simbolos), caducidad periodica y no reutilizacion. Herramientas como `pam_pwquality` en Linux fuerzan estas politicas a nivel del sistema.

## PAM (Pluggable Authentication Modules)

PAM es un framework que permite configurar la autenticacion de forma modular y flexible. En lugar de codificar la logica de autenticacion en cada aplicacion, PAM centraliza la verificacion en archivos de configuracion en `/etc/pam.d/`.

Cada modulo PAM realiza una funcion especifica:

- **pam_unix.so:** autenticacion basada en contrasena Unix.
- **pam_ldap.so:** autenticacion contra un servidor LDAP/Active Directory.
- **pam_google_authenticator.so:** autenticacion de dos factores con TOTP.
- **pam_cracklib.so:** verifica la fortaleza de las contrasenas.
- **pam_limits.so:** establece limites de recursos por usuario.

```
# /etc/pam.d/sshd - ejemplo de configuracion
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
account    required     pam_nologin.so
account    password-auth
password   include      password-auth
session    required     pam_selinux.so close
session    required     pam_loginuid.so
```

Los flags de PAM controlan el comportamiento: `required` (el modulo debe pasar, pero otros modulos se ejecutan), `requisite` (el modulo debe pasar y se detiene si falla), `sufficient` (si pasa y no hay modulos required fallidos, la autenticacion es exitosa) y `optional` (contribuye pero no es decisivo).

## Control de acceso basado en roles (RBAC)

RBAC (*Role-Based Access Control*) asigna permisos a roles en lugar de a usuarios individuales. Los usuarios se asignan a roles, y los roles tienen permisos predefinidos. Esto simplifica la gestion de permisos en entornos grandes.

SELinux y AppArmor son implementaciones de control de acceso obligatorio (*MAC*) en Linux que utilizan conceptos similares a RBAC:

- **SELinux:** cada proceso y archivo tiene una *etiqueta de seguridad* que define que puede hacer. Un proceso etiquetado como `httpd_t` solo puede acceder a archivos etiquetados como `httpd_sys_content_t`.
- **AppArmor:** define perfiles de aplicaciones que especifican que archivos y recursos puede acceder cada programa. Los perfiles pueden estar en modo `enforce` (bloquea violaciones) o `complain` (solo registra).

## Autenticacion de dos factores (2FA)

La autenticacion de dos factores combina dos factores de verificacion:

1. **Algo que sabes:** la contrasena.
2. **Algo que tienes:** un dispositivo (telefono con app de autenticacion, llavero USB FIDO2).
3. **Algo que eres:** biometria (huella dactilar, reconocimiento facial).

En Linux, la autenticacion TOTP (Time-based One-Time Password) se implementa con `pam_google_authenticator`:

```bash
# Instalar Google Authenticator PAM
sudo apt install libpam-google-authenticator

# Configurar para el usuario
google-authenticator

# Añadir a /etc/pam.d/sshd
auth required pam_google_authenticator.so
```

> La autenticacion es la puerta de entrada a la seguridad del sistema. Una contrasena debil anula todas las demas medidas de seguridad. La implementacion de 2FA, la politica de contrasenas robustas y la auditoria de accesos son componentes esenciales de una politica de seguridad de identidad.

