<h1 align="center">Ramas</h1>

<h2>📑 Contenido</h2>

- [¿Que son las ramas?](#que-son-las-ramas)
  - [Ventajas](#ventajas)
- [Operaciones básicas](#operaciones-básicas)
  - [Crear Ramas](#crear-ramas)
  - [Visualizar Ramas](#visualizar-ramas)
  - [Cambiarse de Rama](#cambiarse-de-rama)
  - [Combinar/Fusionar Ramas](#combinarfusionar-ramas)
  - [Eliminar Rama](#eliminar-rama)
  - [Renombrar Rama](#renombrar-rama)
  - [Enviar Rama(remoto)](#enviar-ramaremoto)
  - [Eliminar Rama(remoto)](#eliminar-ramaremoto)

## ¿Que son las ramas?

Una rama en Git es una bifurcación del flujo de desarrollo del proyecto.
Cada repositorio tiene, como mínimo, una rama, en la que almacena los commits realizados en dicho flujo de trabajo.

El uso de ramas nos permite establecer varios flujos de desarrollo, separando el trabajo en curso, de las versiones ya estables de nuestro proyecto.

### Ventajas

- **Aislamiento de cambios:** Cada rama es una copia independiente del proyecto en un estado específico. Esto significa que puedes desarrollar nuevas características, solucionar problemas o realizar experimentos sin afectar directamente a la rama principal del proyecto.
- **Desarrollo paralelo:** Las ramas permiten que múltiples colaboradores trabajen en diferentes aspectos del proyecto al mismo tiempo. Cada colaborador puede crear su propia rama para trabajar en sus cambios, y luego esos cambios se pueden fusionar de nuevo en la rama principal cuando estén listos.
- **Experimentación y pruebas:** Puedes utilizar ramas para probar nuevas ideas o enfoques sin arriesgar la estabilidad de la rama principal. Si una idea no funciona, simplemente puedes descartar la rama sin afectar al código principal.
- **Despliegue controlado:** Las ramas son útiles para implementar un control de calidad riguroso. Puedes crear una rama específica para las pruebas y la validación antes de fusionar los cambios en la rama principal, lo que ayuda a evitar la introducción de errores en el código principal.
- **Versionado de características:** Puedes utilizar ramas para implementar y versionar características específicas. Por ejemplo, puedes tener una rama para una característica A y otra para una característica B, lo que facilita la gestión y el seguimiento de las versiones individuales.

## Operaciones básicas

### Crear Ramas

```bash
 git branch <nombre_de_rama>
```

### Visualizar Ramas

```bash
 git branch
```

### Cambiarse de Rama

```bash
git checkout <nombre_de_rama>
```

### Combinar/Fusionar Ramas

```bash
git merge <nombre_de_rama>
```

### Eliminar Rama

```bash
git branch -D <nombre_de_rama>
```

### Renombrar Rama

Buscar

### Enviar Rama(remoto)

```bash
git push origin  <nombre_de_rama>
```

### Eliminar Rama(remoto)

```bash
git push origin --delete <nombre_de_rama>
```


