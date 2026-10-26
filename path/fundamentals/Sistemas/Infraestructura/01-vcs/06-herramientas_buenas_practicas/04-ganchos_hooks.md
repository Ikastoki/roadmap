# Ganchos (Hooks) del sistema

Los ganchos de Git son scripts que se ejecutan automaticamente antes o despues de ciertas acciones del ciclo de vida: antes de un commit, despues de un checkout, antes de un push. Son el mecanismo de automatizacion local mas poderoso de Git, permitiendo validar codigo, ejecutar pruebas, formatear archivos y bloquear operaciones que no cumplan estandares, todo sin intervencion manual.

Cada repositorio contiene sus ganchos en `.git/hooks/`. Son scripts ejecutables (bash, python, ruby, cualquier lenguaje) que Git invoca en el momento oportuno.

## Ganchos principales

**pre-commit:** se ejecuta antes de que un commit se registre. Si el script termina con un codigo distinto de cero, el commit se cancela.

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Verificar que no haya console.log en codigo JavaScript
if git diff --cached --name-only | xargs grep -l "console.log" 2>/dev/null; then
  echo "Error: se encontro console.log en el codigo. Eliminar antes de commitear."
  exit 1
fi

# Ejecutar linter
npm run lint
if [ $? -ne 0 ]; then
  echo "Error: el linter encontro problemas. Corregir antes de commitear."
  exit 1
fi
```

**commit-msg:** valida el mensaje de commit despues de que el usuario lo escribe.

```bash
#!/bin/sh
# .git/hooks/commit-msg

MSG=$(cat "$1")

# Verificar formato Conventional Commits
if ! echo "$MSG" | grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf|ci)(\(.+\))?: .{1,}"; then
  echo "Error: el mensaje debe seguir Conventional Commits"
  echo "Ejemplo: feat(auth): agregar login con OAuth"
  exit 1
fi
```

**pre-push:** se ejecuta antes de enviar cambios al remoto. Util para ejecutar pruebas rapidas antes de push.

```bash
#!/bin/sh
# .git/hooks/pre-push

# Ejecutar tests antes de push
npm test
if [ $? -ne 0 ]; then
  echo "Error: las pruebas fallaron. Corregir antes de push."
  exit 1
fi
```

**post-merge:** se ejecuta despues de una fusion exitosa. Ideal para instalar dependencias automaticamente.

```bash
#!/bin/sh
# .git/hooks/post-merge

# Actualizar dependencias si cambio package.json
CHANGED=$(git diff HEAD@{1} --name-only | grep "package.json" || true)
if [ -n "$CHANGED" ]; then
  echo "package.json cambio, ejecutando npm install..."
  npm install
fi
```

## Ganchos compartidos del equipo

Los ganchos en `.git/hooks/` no se comprometen con el repositorio (Git ignora `.git/`). Para compartir ganchos con el equipo, se usa la carpeta del proyecto:

```bash
# .githooks/pre-commit (comprometer con el repositorio)
#!/bin/sh
npm run lint && npm test

# Configurar Git para usar esta carpeta
git config core.hooksPath .githooks
```

## Frameworks de ganchos

**Husky** (para proyectos Node.js):

```bash
# Instalar husky
npm install husky --save-dev
npx husky init

# Configurar pre-commit
echo "npm run lint" > .husky/pre-commit

# Configurar commit-msg
echo "npx commitlint --edit \$1" > .husky/commit-msg
```

**pre-commit** (framework multi-lenguaje):

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
  - repo: https://github.com/psf/black
    rev: 24.1.0
    hooks:
      - id: black
```

> Los ganchos transforman las buenas practicas de recomendacion a obligacion. Un equipo que configura pre-commit con linting y tests nunca recibe codigo que no pase la verificacion basica. La automatizacion local es la primera linea de defensa contra la degradacion de la calidad.

