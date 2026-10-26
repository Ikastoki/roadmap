# Etiquetado (Tags)

Las etiquetas (*tags*) marcan puntos especificos en el historial de Git como importantes. A diferencia de las ramas (que son punteros moviles), las etiquetas son punteros fijos que nunca se mueven. Se utilizan tipicamente para marcar versiones de lanzamiento (v1.0.0, v2.3.1), hitos del proyecto o puntos de referencia para auditoria.

El etiquetado es una practica esencial para la gestion de versiones de software: permite a los usuarios identificar que commit corresponde a una version especifica y recuperar el codigo exacto de ese punto en el tiempo.

## Etiquetas ligeras vs anotadas

**Etiqueta ligera (*lightweight tag*):** es simplemente un puntero a un commit, sin metadatos adicionales. Es como una rama que nunca se mueve.

```bash
# Crear etiqueta ligera
git tag v1.0.0

# Crear etiqueta en un commit especifico
git tag v1.0.0 abc1234
```

**Etiqueta anotada (*annotated tag*):** es un objeto completo de Git que incluye el nombre, el mensaje, el autor, la fecha y un hash. Puede ser firmada con GPG para garantizar la autenticidad.

```bash
# Crear etiqueta anotada
git tag -a v1.0.0 -m "Version 1.0.0 - Primera liberacion estable"

# Crear etiqueta anotada con firma GPG
git tag -s v1.0.0 -m "Version 1.0.0 - Firmada criptograficamente"
```

**Recomendacion:** usar siempre etiquetas anotadas para versiones de lanzamiento, ya que contienen mas informacion y pueden ser verificadas.

## Listar y ver etiquetas

```bash
# Listar todas las etiquetas
git tag
# v1.0.0
# v1.1.0
# v2.0.0

# Listar con mensaje
git tag -n

# Ver detalles de una etiqueta
git show v1.0.0

# Buscar etiquetas por patron
git tag -l "v1.*"
```

## Enviar etiquetas al repositorio remoto

Las etiquetas no se envian automaticamente con `git push`. Deben enviarse explicitamente:

```bash
# Enviar una etiqueta especifica
git push origin v1.0.0

# Enviar todas las etiquetas locales
git push origin --tags

# Eliminar etiqueta remota
git push origin --delete v1.0.0
```

## Nomenclatura de versiones (Semantic Versioning)

La nomenclatura recomendada para versiones sigue Semantic Versioning (*SemVer*):

```
MAJOR.MINOR.PATCH

v1.0.0
│ │ │
│ │ └─ PATCH: correcciones de bugs, no cambia la API
│ └─── MINOR: nuevas funcionalidades, retrocompatible
└───── MAJOR: cambios que rompen compatibilidad hacia atras
```

```bash
# Ejemplos
git tag -a v1.0.0 -m "Primera liberacion estable"
git tag -a v1.1.0 -m "Agregar filtro de busqueda"
git tag -a v1.1.1 -m "Corregir bug en validacion"
git tag -a v2.0.0 -m "API v2 - cambios incompatibles"
```

> Las etiquetas son la forma de anclar el historial de Git a puntos concretos y significativos. En un proyecto bien gestionado, cada release tiene una etiqueta anotada que permite recuperar el codigo exacto de esa version, verificar la integridad y generar changelogs automaticos.

