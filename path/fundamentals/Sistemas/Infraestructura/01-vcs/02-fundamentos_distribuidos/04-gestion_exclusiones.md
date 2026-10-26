# Gestion de exclusiones

No todos los archivos de un proyecto deben ser controlados por versiones. Archivos generados por el compilador, dependencias instaladas automaticamente, archivos de configuracion local con secretos y archivos temporales deben ser excluidos del repositorio para mantenerlo limpio, seguro y eficiente.

Git utiliza archivos de exclusiones (`.gitignore`) para indicar que archivos o patrones no deben ser rastreados. Un buen `.gitignore` es esencial para evitar subir accidentalmente datos sensibles, archivos de gran tamano o archivos que generan conflictos innecesarios.

## Archivos que deben excluirse

- **Dependencias:** `node_modules/`, `vendor/`, `venv/`, `__pycache__/`. Se pueden regenerar con un gestor de paquetes.
- **Archivos generados:** `build/`, `dist/`, `*.o`, `*.class`, `*.pyc`. Se generan al compilar el codigo.
- **Secretos y credenciales:** `.env`, `*.pem`, `credentials.json`, `id_rsa`. Si se suben, cualquier persona con acceso al repositorio puede verlos.
- **Archivos de configuracion local:** `.vscode/settings.json`, `*.swp`, `*.swo`. Son especificos de cada desarrollador.
- **Archivos de sistema:** `.DS_Store` (macOS), `Thumbs.db` (Windows), `*.log`.

## Formato de .gitignore

El archivo `.gitignore` se coloca en la raiz del repositorio y contiene patrones de exclusion, uno por linea.

```gitignore
# Comentario: ignorar directorio de dependencias
node_modules/

# Ignorar archivos de build
build/
dist/
*.o
*.class

# Ignorar archivos de configuracion local
.env
*.pem
id_rsa

# Ignorar archivos de sistema
.DS_Store
Thumbs.db

# Ignorar logs
*.log
logs/
```

**Patrones avanzados:**

```gitignore
# Ignorar todos los .txt excepto README.txt
*.txt
!README.txt

# Ignorar archivos de logs pero mantener el directorio
logs/
!logs/.gitkeep

# Ignorar archivos en cualquier subdirectorio
**/temp/
```

## Crear un .gitignore

```bash
# Crear .gitignore basico
cat > .gitignore << 'EOF'
# Dependencias
node_modules/
venv/
__pycache__/

# Build
build/
dist/
*.o
*.pyc

# Config local
.env
*.pem
.vscode/
.idea/

# Sistema
.DS_Store
Thumbs.db
*.swp
EOF
```

Plantillas predefinidas para multiples lenguajes y frameworks estan disponibles en `github.com/github/gitignore`.

## Gestor global de exclusiones

Para exclusiones que aplican a todos los repositorios del usuario (archivos de sistema, editores):

```bash
# Configurar exclusiones globales
git config --global core.excludesfile ~/.gitignore_global

# Crear el archivo global
cat > ~/.gitignore_global << 'EOF'
.DS_Store
Thumbs.db
*.swp
*.swo
*~
.vscode/
.idea/
EOF
```

## Corregir archivos subidos accidentalmente

Si un archivo que deberia estar en `.gitignore` ya esta rastreado, simplemente agregarlo a `.gitignore` no lo elimina del historial. Se debe eliminar del rastreo preservando el archivo en disco:

```bash
# Eliminar del rastreo pero mantener en disco
git rm --cached archivo_secreto.env

# Eliminar del rastreo y del disco
git rm archivo_temporal.log

# Confirmar la eliminacion
git commit -m "Eliminar archivos que no deberian estar rastreados"
```

> Un `.gitignore` bien configurado es una measure de seguridad critica. Subir credenciales o secretos a un repositorio publico es uno de los errores mas comunes y peligrosos en el desarrollo de software. Herramientas como `git-secrets`, `truffleHog` y `gitleaks` pueden escanear el historial en busca de secretos subidos accidentalmente.

