# Flujo de trabajo con solicitudes de fusion

Las solicitudes de fusion (Pull Requests en GitHub, Merge Requests en GitLab) son el mecanismo central para proponer, revisar y aceptar cambios en un repositorio compartido. No son solo un boton para fusionar: son un espacio de discusion, revision y validacion que garantiza que los cambios cumplan los estandares del proyecto antes de integrarse con la rama principal.

El flujo completo sigue un patron estructurado que conecta el trabajo individual con la revision colectiva.

## Flujo paso a paso

**1. Crear una rama para el cambio**

```bash
git checkout -b feature/agregar-busqueda
```

**2. Hacer commits y enviar la rama**

```bash
git add .
git commit -m "feat: implementar busqueda de usuarios"
git push -u origin feature/agregar-busqueda
```

**3. Abrir la solicitud desde la plataforma**

La plataforma detecta el push y muestra un boton para abrir la solicitud. Seleccionas la rama de destino (generalmente `main` o `develop`) y añades descripcion del cambio.

**4. Revision por pares**

Los revisores examinan el codigo, dejan comentarios inline y pueden solicitar cambios. El autor responde, hace las modificaciones y vuelve a enviar.

```bash
# El autor corrige segun los comentarios
git add .
git commit -m "fix: aplicar sugerencias de revision"
git push
# La solicitud se actualiza automaticamente
```

**5. Verificacion automatica**

Los pipelines de CI ejecutan pruebas, linting y construccion. Si fallan, la solicitud no puede fusionarse hasta que se corrijan.

**6. Aprobacion y fusion**

Cuando los revisores aprueban y los checks pasan, se fusiona la solicitud. La plataforma ofrece opciones de fusion:

- **Merge commit**: conserva toda la historia (merge tradicional).
- **Squash and merge**: comprime todos los commits en uno solo.
- **Rebase and merge**: reescribe los commits sobre la rama destino.

**7. Limpiar la rama**

```bash
git checkout main
git pull origin main
git branch -d feature/agregar-busqueda
git push origin --delete feature/agregar-busqueda
```

## Reglas de proteccion

Las plataformas permiten configurar reglas que bloquean la fusion directa en ramas importantes:

- Requisito de aprobaciones minimas (ej. 2 revisores).
- Checks de CI obligatorios que deben pasar.
- Prohibicion de force push sobre la rama.
- Obligatoriedad de resolucion de todas las conversaciones.

## Convenciones para buenas solicitudes

- Titulo claro y conciso que describa el cambio.
- Descripcion con contexto: que problema resuelve, como se implemento, como probarlo.
- Enlazar la incidencia que resuelve (ej. `Closes #42`).
- Limitar el tamano: cambios pequenos son mas faciles de revisar que PRs de miles de lineas.

> Las solicitudes de fusion transforman el codigo de un producto individual en un producto colectivo. Sin ellas, la协作 se reduce a confiar en que cada persona envia codigo correcto. Con ellas, cada cambio pasa por un filtro de calidad que mejora el proyecto y forma al equipo.

