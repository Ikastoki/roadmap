# Estrategias de ramificacion

Las estrategias de ramificacion (*branching models*) definen como se organizan las ramas en un proyecto, que nombres tienen, que tipo de cambios van en cada rama y como se fusionan. Elegir una estrategia adecuada es importante para mantener un historial limpio, facilitar la协作 y integrar cambios de forma segura.

No existe una estrategia universalmente correcta: la eleccion depende del tamano del equipo, la frecuencia de despliegue, la complejidad del proyecto y la cultura del equipo.

## Feature Branches (ramas por funcionalidad)

La estrategia mas comun y recomendada: cada funcionalidad, correccion o tarea vive en su propia rama temporal que se crea desde la rama principal y se fusiona de vuelta cuando esta completa.

```bash
# Flujo basico de feature branch
git checkout -b feature/nueva-funcionalidad
# ... trabajar y hacer commits ...
git checkout main
git merge --no-ff feature/nueva-funcionalidad
git branch -d feature/nueva-funcionalidad
```

**Ventajas:**

- Aisla el trabajo: los bugs o errores de una feature no afectan a la rama principal.
- Facilita la revision de codigo: el PR/MR contiene solo los cambios de esa feature.
- Permite experimentar sin riesgo para el codigo estable.

**Convenciones de nombres:**

- `feature/descripcion`: nuevas funcionalidades
- `bugfix/descripcion`: correccion de bugs
- `hotfix/descripcion`: correccion urgente en produccion
- `release/v1.0.0`: preparacion de un lanzamiento

## Git Flow

Git Flow es un modelo de ramificacion con reglas estrictas que define ramas con propositos especificos y ciclos de vida claros.

```
main (produccion)
  │
  ├── develop (desarrollo integrado)
  │     │
  │     ├── feature/login
  │     ├── feature/dashboard
  │     │
  │     ├── release/v1.0.0
  │     │
  │     └── hotfix/corregir-bug-critico
  │
  └── v1.0.0 (tag)
```

**Ramas en Git Flow:**

- **main:** produccion. Solo recibe merges de release y hotfix.
- **develop:** integracion de todas las features. Base para releases.
- **feature/*:** funcionalidades nuevas. Se crean desde develop y se fusionan de vuelta.
- **release/*:** preparacion de un lanzamiento. Correcciones menores, actualizacion de version.
- **hotfix/*:** correcciones urgentes para produccion. Se crean desde main y se fusionan a main y develop.

Git Flow es adecuado para proyectos con ciclos de lanzamiento definidos y despliegues manuales o poco frecuentes.

## Trunk-Based Development (desarrollo en tronco)

La estrategia opuesta a Git Flow: todo el trabajo se realiza en la rama principal (`main` o `trunk`) con ramas de corta duracion que se fusionan rapidamente (idealmente en menos de un dia).

```
main
  │
  ├── feature/  (1-2 commits, fusionada en horas)
  ├── feature/  (1-2 commits, fusionada en horas)
  ├── bugfix/   (1 commit, fusionada en horas)
  │
  └── tags: v1.0.0, v1.1.0
```

**Ventajas:**

- Historial lineal y simple.
- Integracion continua natural.
- Menos conflictos de fusion (las ramas viven poco tiempo).

**Requisitos:**

- Tests automaticos robustos (para no romper main).
- Integracion continua (CI) que valide cada push.
- Equipo disciplinado en la revision de codigo.

## Comparacion

| Aspecto | Git Flow | Trunk-Based |
|---|---|---|
| Complejidad | Alta | Baja |
| Ramas largas | Si | No (cortas) |
| Despliegue | Periodico | Continuo |
| Conflictos | Frecuentes | Pocos |
| Ideal para | Proyectos con releases | CI/CD, startups |

> La tendencia actual en la industria es Trunk-Based Development combinado con CI/CD. Git Flow sigue siendo valido para proyectos con ciclos de lanzamiento largos, pero para la mayoria de los equipos modernos, feature branches cortas con integracion continua es la estrategia mas efectiva.

