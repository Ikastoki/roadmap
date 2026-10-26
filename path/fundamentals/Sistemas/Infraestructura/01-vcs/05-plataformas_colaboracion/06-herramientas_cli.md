# Herramientas de linea de comandos de la plataforma

Las plataformas de colaboracion (GitHub, GitLab, Bitbucket) ofrecen herramientas de linea de comandos que permiten interactuar directamente desde la terminal: crear solicitudes de fusion, gestionar incidencias, revisar codigo, gestionar secretos y ejecutar pipelines. Estas CLI eliminan la necesidad de cambiar entre la terminal y el navegador para tareas repetitivas, acelerando el flujo de trabajo y permitiendo la automatizacion en scripts.

## GitHub CLI (gh)

La herramienta oficial de GitHub para la terminal.

```bash
# Instalacion (Linux)
sudo apt install gh
# O: brew install gh (macOS)

# Autenticacion
gh auth login

# Crear una solicitud de fusion
gh pr create --title "feat: nueva busqueda" --body "Implementa busqueda avanzada"

# Ver solicitudes abiertas
gh pr list

# Ver detalles de una PR
gh pr view 42

# Fusionar una PR
gh pr merge 42 --squash

# Crear una incidencia
gh issue create --title "Bug: login falla" --body "Descripcion del error"

# Ver incidencias abiertas
gh issue list

# Ver estado de checks de CI
gh run list
gh run view 12345
```

## GitLab CLI (glab)

```bash
# Instalacion
brew install glab

# Autenticacion
glab auth login

# Crear MR
glab mr create --title "feat: nueva funcionalidad"

# Listar MRs
glab mr list

# Crear incidencia
glab issue create --title "Bug report"
```

## Bitbucket CLI (bb)

```bash
# Instalacion
pip install bitbucket-cli

# Crear PR
bb pr create --title "Mi cambio"

# Listar PRs
bb pr list
```

## Uso en scripts y automatizacion

Las CLI son especialmente utiles en scripts de automatizacion:

```bash
#!/bin/bash
# Script para crear PR con descripcion automatica
BRANCH=$(git branch --show-current)
DESCRIPTION="Cambios en la rama $BRANCH"

gh pr create \
  --title "feat: $BRANCH" \
  --body "$DESCRIPTION" \
  --base main \
  --head "$BRANCH"
```

```bash
# Verificar si hay PRs pendientes antes de desplegar
OPEN_PRS=$(gh pr list --json number --jq '. | length')
if [ "$OPEN_PRS" -gt 0 ]; then
  echo "Hay $OPEN_PRS PRs pendientes de revision"
  exit 1
fi
```

## Diferencia con la interfaz web

| Aspecto | CLI | Interfaz web |
|---|---|---|
| Velocidad | Rapido para tareas repetitivas | Mas lento (navegar, hacer clic) |
| Visualizacion | Limitada | Excelente para revision de codigo |
| Automatizacion | Ideal para scripts | No apta para automatizacion |
| Aprendizaje | Requiere memorizar comandos | Intuitivo |
| Accesibilidad | Funciona en servidores sin GUI | Requiere navegador |

> Las CLI de las plataformas completan el ecosistema Git en la terminal. Para un desarrollador que ya usa Git desde la linea de comandos, estas herramientas eliminan el contexto switching y permiten gestionar todo el ciclo de vida del codigo sin salir del terminal.

