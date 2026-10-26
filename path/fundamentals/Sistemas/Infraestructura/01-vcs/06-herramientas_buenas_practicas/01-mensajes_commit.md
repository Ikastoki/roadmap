# Mensajes de commit efectivos

Un commit sin mensaje descriptivo es un commit inutil para el historial. Los mensajes de commit son la documentacion viva del proyecto: explican que cambio, porque cambio y cual era el contexto. Un historial con mensajes claros permite encontrar cambios especificos, entender la evolucion del proyecto y diagnosticar errores introducidos en commits anteriores.

Los mensajes de commit no son solo notas para ti mismo. Son comunicacion asincrona con todo el equipo, incluyendo a las personas que se uniran al proyecto en el futuro.

## Estructura recomendada

**Asunto (primera linea):**
- Maximo 50 caracteres.
- En modo imperativo ("Agregar" no "Agregado").
- Sin punto al final.
- Resumir el cambio en una frase.

**Cuerpo (lineas adicionales):**
- Separado del asunto por una linea en blanco.
- Explicar el porque y el como, no el que (el que ya lo dice el asunto).
- Maximo 72 caracteres por linea.
- Puede incluir referencias a incidencias.

```bash
git commit -m "Agregar validacion de email en formulario de registro

El formulario aceptaba emails sin dominio valido, causando
errores en el servidor de correo. Se añadio regex para
validar formato antes de enviar.

Fixes #127"
```

## Conventional Commits

Estandar de la industria que estructura los mensajes con prefijos que indican el tipo de cambio:

```
<tipo>(<ambito>): <asunto>

[opcional: cuerpo]

[opcional: footer]
```

**Tipos principales:**

| Tipo | Uso | Ejemplo |
|---|---|---|
| `feat` | Nueva funcionalidad | `feat(auth): agregar login con OAuth` |
| `fix` | Correccion de error | `fix(api): corregir timeout en peticiones` |
| `docs` | Solo documentacion | `docs: actualizar README con ejemplos` |
| `style` | Formato (sin cambio de logica) | `style: aplicar indentacion a config.js` |
| `refactor` | Reestructuracion sin cambio de funcionalidad | `refactor: extraer funcion de validacion` |
| `test` | Agregar o corregir pruebas | `test: agregar tests para servicio de email` |
| `chore` | Tareas de mantenimiento | `chore: actualizar dependencias` |
| `perf` | Mejora de rendimiento | `perf: optimizar consulta de base de datos` |
| `ci` | Cambios en configuracion de CI | `ci: agregar pipeline de testing` |

```bash
git commit -m "feat(search): agregar busqueda avanzada por filtros

Implementa busqueda con filtros combinables:
- Por nombre, categoria y fecha de creacion
- Soporte para rangos de fechas
- Paginacion de resultados

Closes #89"
```

## Mensajes a evitar

```bash
# MAL: vago, no explica que cambio
git commit -m " arreglo"

# MAL: no usa imperativo
git commit -m "Agregada validacion de email"

# MAL: solo dice "update"
git commit -m "update"

# BIEN: especifico, imperativo, con contexto
git commit -m "fix(auth): corregir timeout en login con LDAP

El servidor LDAP respondia lentamente bajo carga, causando
timeouts de 30s. Se redujo el timeout a 5s y se añadio
retry con backoff exponencial.

Fixes #234"
```

## Automatizacion con hooks

Puedes usar ganchos (*hooks*) para validar mensajes automaticamente:

```bash
# .git/hooks/commit-msg
#!/bin/sh
# Verificar que el mensaje empiece por un tipo valido
if ! grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf|ci):" "$1"; then
  echo "Error: el mensaje debe empezar por feat|fix|docs|style|refactor|test|chore|perf|ci"
  exit 1
fi
```

> Un buen mensaje de commit es una inversion en la mantenibilidad del proyecto. Cuando un colega (o tu mismo dentro de seis meses) busque "quien cambio la validacion de email y porque", un historial claro responde esa pregunta sin necesidad de revisar el codigo.

