# Documentacion y estandares del repositorio

Un repositorio bien documentado es un repositorio que puede ser utilizado, mantenido y contribuido por cualquier persona sin necesidad de preguntar. Los archivos estandar del repositorio (`README.md`, `CONTRIBUTING.md`, `CODEOWNERS`, `LICENSE`) constituyen el contrato implicito entre los mantenedores y los colaboradores: explican que hace el proyecto, como contribuir y quien es responsable de que.

Estos archivos no son opcionales ni decorativos. Son la primera linea de defensa contra la desorganizacion y la desercion de colaboradores que no entienden el proyecto.

## Archivos esenciales

**README.md:** la puerta de entrada al proyecto. Debe incluir:
- Nombre y descripcion breve del proyecto.
- Como instalar y ejecutar.
- Ejemplo basico de uso.
- Enlace a la documentacion completa.
- Informacion de contacto o canales de soporte.

```markdown
# Mi Proyecto

Herramienta para gestionar inventarios de hardware en redes empresariales.

## Instalacion

```bash
pip install mi-proyecto
```

## Uso

```bash
mi-proyecto inventario scan --subnet 192.168.1.0/24
```

## Contribuir

Lee [CONTRIBUTING.md](CONTRIBUTING.md) para instrucciones.

## Licencia

MIT - Ver [LICENSE](LICENSE)
```

**CONTRIBUTING.md:** guia para colaboradores que incluye:
- Requisitos previos (herramientas, versiones).
- Como reportar errores.
- Como proponer cambios.
- Estilos de codigo y convenciones de commits.
- Proceso de revision y fusion.

**CODEOWNERS:** define automaticamente que personas o equipos deben revisar cambios en archivos o directorios especificos.

```
# CODEOWNERS
*.js       @equipo-frontend
*.py       @equipo-backend
/docs/     @equipo-documentacion
infra/     @equipo-devops
```

**LICENSE:** declara bajo que terminos se distribuye el codigo. Sin licencia, el codigo es propiedad exclusiva del autor por defecto.

## Plantillas para issues y PRs

Las plataformas permiten definir plantillas que se cargan automaticamente al crear una incidencia o solicitud de fusion.

```markdown
<!-- .github/ISSUE_TEMPLATE/bug_report.md -->
---
name: Reportar un error
about: Describe el problema encontrado
labels: bug
---

## Descripcion
[Describe el error]

## Pasos para reproducir
1. ...
2. ...

## Comportamiento esperado
[Que deberia ocurrir]
```

## Archivos complementarios

- **`.editorconfig`**: define estilos de indentacion y formato para que todos los colaboradores usen la misma configuracion.
- **`.gitignore`**: archivos que Git debe ignorar (dependencias, binarios, secretos).
- **`CHANGELOG.md`**: registro de cambios relevantes por version.
- **`SECURITY.md`**: politica de reporte de vulnerabilidades de seguridad.

> Un README incompleto es peor que no tener README: genera falsas expectativas. Un proyecto que explica claramente que hace, como se instala y como contribuir atrae mas colaboradores que uno con codigo brillante pero sin documentacion.

