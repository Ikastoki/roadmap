# Concepto de rama

Una rama (*branch*) en Git es un puntero movil a una confirmacion (*commit*) especifica. Cada rama representa una linea independiente de desarrollo que puede evolucionar sin afectar a las demas. Las ramas son la herramienta fundamental para el desarrollo paralelo: permiten que multiples personas trabajen en funcionalidades diferentes simultaneamente sin interferirse.

El concepto de rama es lo que transforma a Git de un simple sistema de control de versiones en una plataforma de协作 potente. Sin ramas, todo el trabajo se realizaria en una unica linea, creando conflictos constantes y limitando la capacidad de experimentar.

## Como funcionan las ramas

En Git, una rama es simplemente un archivo que contiene el hash SHA-1 de un commit. Cuando haces un commit en una rama, el puntero de la rama se actualiza automaticamente para apuntar al nuevo commit.

```
main:  a1b2c3d ← e4f5g6h ← i7j8k9l (HEAD)
                            ↑
                         puntero de rama 'main'
```

Crear una nueva rama es instantaneo: solo crea un nuevo puntero. No copia archivos ni crea duplicados del repositorio. Esto hace que las ramas en Git sean extremadamente ligeras y rapidas.

## Rama principal

Por convencion, la rama principal se llama `main` (o `master` en repositorios antiguos). Es la rama que contiene la version mas estable y reciente del proyecto. En repositorios con despliegue continuo, `main` es tipicamente la rama que se despliega automaticamente a produccion.

```bash
# Ver ramas actuales
git branch

# Crear nueva rama
git branch feature/login

# Cambiar a la nueva rama
git checkout feature/login
# O con el comando moderno:
git switch feature/login

# Crear y cambiar a la vez
git checkout -b feature/login
git switch -c feature/login

# Eliminar rama (solo si ya fue fusionada)
git branch -d feature/login

# Eliminar rama sin verificar fusion
git branch -D feature/temporal
```

## Cabeza (HEAD)

`HEAD` es un puntero especial que indica en que rama (y commit) te encuentras actualmente. Cuando haces un commit, HEAD y la rama activa se actualizan para apuntar al nuevo commit.

```bash
# Ver HEAD actual
git log --oneline -1
# a1b2c3d (HEAD -> main) Ultimo commit

# HEAD simbolico: apunta a la rama
cat .git/HEAD
# ref: refs/heads/main
```

## Rama vs etiqueta (tag)

Una **rama** es un puntero que se mueve con cada commit nuevo. Una **etiqueta** (*tag*) es un puntero fijo a un commit especifico, tipicamente utilizado para marcar versiones de lanzamiento (v1.0.0, v2.3.1).

```bash
# Etiqueta ligera (puntero simple)
git tag v1.0.0

# Etiqueta anotada (objeto completo con metadatos)
git tag -a v1.0.0 -m "Version 1.0.0 - Primera liberacion"
```

> Las ramas son la herramienta mas poderosa de Git para el desarrollo colaborativo. Su ligereza y velocidad hacen que no haya excusa para no usar ramas: cada funcionalidad, cada correccion y cada experimento debe vivir en su propia rama hasta que este lista para ser fusionada.

