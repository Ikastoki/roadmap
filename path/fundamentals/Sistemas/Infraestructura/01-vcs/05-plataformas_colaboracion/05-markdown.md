# Markdown

Markdown es un lenguaje de marcado ligero disenado para ser legible en texto plano y convertido facilmente a HTML y otros formatos. Se ha convertido en el estandar universal para documentacion en repositorios, wikis, incidencias, solicitudes de fusion y hasta en la documentacion tecnica de proyectos. Su simplicidad lo hace accesible para cualquier persona sin conocimientos de HTML, pero suficientemente potente para estructurar documentos complejos.

Las plataformas de colaboracion renderizan Markdown automaticamente, por lo que los archivos `.md` se visualizan con formato enriquecido directamente en la interfaz.

## Sintaxis basica

**Encabezados:**

```markdown
# Titulo principal (H1)
## Seccion (H2)
### Subseccion (H3)
#### Cuarto nivel (H4)
```

**Texto y formato:**

```markdown
Texto en *cursiva* y en **negrita**.
Texto ~~tachado~~ y `codigo en linea`.
Enlace a [Google](https://google.com).
Imagen: ![Alt text](imagen.png)
```

**Listas:**

```markdown
- Elemento sin orden
- Otro elemento
  - Subelemento anidado

1. Primer elemento
2. Segundo elemento
   1. Subelemento
```

**Bloques de codigo:**

````markdown
```python
def funcion():
    return "codigo con resaltado de sintaxis"
```
````

**Citas:**

```markdown
> Esta es una cita textual.
> Puede ocupar multiples lineas.
```

**Tablas:**

```markdown
| Columna 1 | Columna 2 | Columna 3 |
|-----------|-----------|-----------|
| Dato A    | Dato B    | Dato C    |
| Dato D    | Dato E    | Dato F    |
```

**Tareas:**

```markdown
- [x] Tarea completada
- [ ] Tarea pendiente
- [ ] Otra tarea pendiente
```

**Separadores:**

```markdown
---
```

## Usos de Markdown en plataformas

- **README.md**: documentacion principal del proyecto.
- **Incidencias**: reportar bugs, proponer mejoras.
- **Solicitudes de fusion**: descripcion de cambios.
- **Wikis**: documentacion extensa del proyecto.
- **Comentarios en codigo**: documentacion inline que se renderiza en la interfaz.
- **Changelogs**: registro de cambios por version.

## Herramientas de Markdown

- **Editores**: VS Code (con vista previa), Typora, Obsidian.
- **Convertidores**: `pandoc` convierte Markdown a PDF, HTML, DOCX.
- **Linters**: `markdownlint` verifica consistencia de formato.
- **Generadores de documentacion**: MkDocs, Docusaurus, Jekyll.

## Mejores practicas

- Usar encabezados en orden jerarquico (no saltar de H1 a H4).
- Incluir un `README.md` en la raiz de cada repositorio.
- Usar bloques de codigo con el lenguaje especificado para resaltado de sintaxis.
- Mantener las lineas a una longitud razonable (80-100 caracteres).
- Usar enlaces relativos para referenciar otros archivos del repositorio.

> Markdown es el lenguaje de la documentacion tecnica moderna. Su poder esta en su simplicidad: con solo caracteres de formato, puedes crear documentos estructurados, legibles en texto plano y renderizados automaticamente por cualquier plataforma. Dominar Markdown es tan importante como dominar el lenguaje de programacion que usas.

