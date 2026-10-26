# Colisiones y buenas practicas de red

Cuando multiples personas trabajan en el mismo repositorio, es inevitable que se produzcan colisiones: dos personas modifican el mismo archivo, una envia cambios que otra ya habia modificado localmente, o el historial remoto y local divergen. Estas colisiones son normales y se resuelven con los mecanismos de fusion de Git, pero la clave esta en minimizarlas y resolverlas de forma ordenada.

Las buenas practicas de red en Git son simples pero criticas: sincronizar antes de trabajar, comunicar cambios al equipo y evitar operaciones destructivas en ramas compartidas.

## Causas comunes de colisiones

- **Dos personas modifican la misma linea** de un archivo en ramas diferentes.
- **Una persona envia cambios** mientras otra esta trabajando sin sincronizar.
- **Ramas que viven mucho tiempo** sin ser fusionadas, acumulando divergencias.
- **Commits en `main`** que no estan en la rama de trabajo (o viceversa).

## Prevencion de colisiones

**Sincronizar antes de cada tarea:**

```bash
# Siempre antes de empezar a trabajar
git checkout main
git pull origin main
git checkout -b mi-tarea
```

**Fusionar ramas frecuentemente:** las ramas que viven mucho tiempo acumulan cambios que generan conflictos. Fusionar regularmente (o rebase con main) mantiene las ramas actualizadas y reduce la probabilidad de conflictos.

**Comunicar cambios al equipo:** si vas a modificar archivos criticos, avisa a los demas para que eviten los mismos archivos.

**Usar pull antes de push:**

```bash
# Siempre antes de enviar
git pull --rebase origin main
git push origin main
```

## Resolucion de conflictos

Cuando Git no puede fusionar automaticamente dos cambios, marca el conflicto en el archivo:

```python
def buscar_usuario(nombre):
<<<<<<< HEAD
    resultados = db.query("SELECT * FROM usuarios WHERE nombre = ?", nombre)
    return resultados
=======
    resultados = db.query("SELECT * FROM usuarios WHERE nombre LIKE ?", f"%{nombre}%")
    return [r for r in resultados if r.activo]
>>>>>>> feature/busqueda
```

**Proceso de resolucion:**

1. Identificar el conflicto (busca los marcadores `<<<<<<<`, `=======`, `>>>>>>>`).
2. Decidir que cambio conservar (o combinar ambos).
3. Eliminar los marcadores de conflicto.
4. Guardar el archivo.
5. Marcar como resuelto: `git add archivo.py`.
6. Finalizar la fusion: `git commit`.

```bash
# Ver archivos en conflicto
git status

# Herramienta grafica de resolucion
git mergetool
```

## Force push: cuando y como usarlo

`git push --force` sobrescribe el historial remoto. Es peligroso en ramas compartidas pero a veces necesario.

**Usos legitimos:**

- Corregir commits en una rama feature que solo tu estas usando.
- Revertir un push accidental de datos sensibles.

**Nunca usar:**

- En `main` o cualquier rama compartida.
- Sin comunicarlo al equipo.

```bash
# Alternativa mas segura
git push --force-with-lease origin feature
```

`--force-with-lease` verifica que el historial remoto no ha cambiado desde tu ultima obtencion. Si alguien mas envio commits, el push falla en lugar de sobrescribirlos.

> Las colisiones en Git no son fallos del sistema; son la consecuencia natural de la协作 en paralelo. Un equipo que sincroniza frecuentemente, comunica cambios y resuelve conflictos de forma ordenada puede trabajar de forma eficiente incluso con multiples personas modificando los mismos archivos.

