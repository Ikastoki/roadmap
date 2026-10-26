# Autenticacion segura con SSH

SSH (Secure Shell) es el protocolo estandar para conectarse de forma segura a servidores remotos. En el contexto de Git, SSH elimina la necesidad de introducir credenciales manualmente en cada operacion de push/pull: un par de claves criptograficas reemplaza a las contraseñas, proporcionando autenticacion rapida, segura y sin interaccion del usuario.

La mayoria de plataformas (GitHub, GitLab, Bitbucket) soportan autenticacion por SSH y la recomiendan como metodo principal de acceso a repositorios.

## Por que SSH en lugar de HTTPS

| Aspecto | HTTPS | SSH |
|---|---|---|
| Credenciales | Usuario + contrasena/token | Par de claves criptograficas |
| Autenticacion | Cada operacion o token persistente | Una vez configurado, automatico |
| Seguridad | Depende de la fortaleza de la contrasena | Clave criptografica (practicamente inquebrantable) |
| Interaccion | Puede requerir 2FA o MFA | Sin interaccion despues de configurar |
| Servidores | Funciona en cualquier lugar | Requiere acceso al puerto 22 |

## Configuracion paso a paso

**1. Generar el par de claves**

```bash
# Generar clave Ed25519 (recomendada, mas rapida y segura que RSA)
ssh-keygen -t ed25519 -C "tu@email.com"

# Si el sistema no soporta Ed25519
ssh-keygen -t rsa -b 4096 -C "tu@email.com"
```

Esto crea dos archivos:
- `~/.ssh/id_ed25519` (clave privada: NUNCA compartirla).
- `~/.ssh/id_ed25519.pub` (clave publica: subir a la plataforma).

**2. Iniciar el agente SSH**

El agente SSH almacena la clave privada en memoria para no tener que introducir la contrasena en cada operacion.

```bash
# Iniciar el agente
eval "$(ssh-agent -s)"
# Agent pid 12345

# Agregar la clave privada al agente
ssh-add ~/.ssh/id_ed25519
```

**3. Copiar la clave publica**

```bash
# Linux/macOS
cat ~/.ssh/id_ed25519.pub | pbcopy    # macOS
cat ~/.ssh/id_ed25519.pub | xclip    # Linux

# Windows (PowerShell)
Get-Content ~/.ssh/id_ed25519.pub | Set-Clipboard
```

**4. Registrar en la plataforma**

- **GitHub:** Settings > SSH and GPG keys > New SSH key > Pegar la clave publica.
- **GitLab:** Preferences > SSH Keys > Pegar la clave publica.
- **Bitbucket:** Personal settings > SSH keys > Add key.

**5. Probar la conexion**

```bash
# GitHub
ssh -T git@github.com
# Hi usuario! You've successfully authenticated, but GitHub does not provide shell access.

# GitLab
ssh -T git@gitlab.com
# Welcome to GitLab, @usuario!
```

**6. Usar URLs SSH para Git**

```bash
# Cambiar el remoto de HTTPS a SSH
git remote set-url origin git@github.com:usuario/proyecto.git

# Clonar directamente con SSH
git clone git@github.com:usuario/proyecto.git
```

## Agente SSH y persistencia

```bash
# Configurar auto-add de claves (Linux)
# ~/.ssh/config
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519

# Para que el agente sobreviva al cerrar la terminal
# ~/.bashrc o ~/.zshrc
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null 2>&1
  ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi
```

## Solucion de problemas

**Permisos incorrectos:**

```bash
# La clave privada debe ser solo legible por el usuario
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# La carpeta .ssh debe tener permisos restrictivos
chmod 700 ~/.ssh
```

**Agente no activo:**

```bash
# Verificar si el agente esta corriendo
ssh-add -l

# Si no hay claves, reiniciar el agente
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

**Clave rechazada:**

```bash
# Verificar la conexion con debug
ssh -Tv git@github.com

# Causas comunes:
# - Clave publica no registrada en la plataforma.
# - Nombre de usuario incorrect en la URL.
# - Permisos de archivos incorrectos.
```

**Multiples claves (diferentes cuentas):**

```bash
# ~/.ssh/config
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

Host github-trabajo
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_trabajo

# Usar el alias en el remoto
git remote set-url origin git@github-personal:usuario/proyecto.git
```

> SSH es la autenticacion estandar en entornos profesionales. Una vez configurado, elimina la friccion de las contraseñas y tokens, y es mas seguro que cualquier alternativa basada en contraseñas. La inversion de 10 minutos en configurar SSH se traduce en anos de autenticacion transparente.

