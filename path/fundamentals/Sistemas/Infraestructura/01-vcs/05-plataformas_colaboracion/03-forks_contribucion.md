# Forks y contribucion a proyectos de terceros

Cuando no tienes acceso de escritura a un repositorio, la forma de contribuir es haciendo una bifurcacion (*fork*): una copia completa del proyecto en tu propia cuenta de la plataforma. Desde ahi, puedes modificar el codigo, enviar cambios y proponer su integracion al proyecto original mediante una solicitud de fusion.

El modelo de forks es la piedra angular de la协作 en proyectos de codigo abierto. Permite que cualquier persona del mundo contribuya sin necesidad de permisos especiales.

## Flujo completo de fork

**1. Bifurcar el repositorio desde la plataforma**

Desde GitHub, GitLab o Bitbucket, haz clic en el boton "Fork". Se crea una copia del repositorio en tu espacio de usuario.

**2. Clonar tu fork localmente**

```bash
git clone git@github.com:tuproject/proyecto.git
cd proyecto
```

**3. Configurar el repositorio original como upstream**

```bash
git remote add upstream git@github.com:original/proyecto.git

# Verificar remotos configurados
git remote -v
# origin    git@github.com:tuproject/proyecto.git (fetch)
# origin    git@github.com:tuproject/proyecto.git (push)
# upstream  git@github.com:original/proyecto.git (fetch)
# upstream  git@github.com:original/proyecto.git (push)
```

**4. Sincronizar con el upstream periodicamente**

```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

**5. Crear una rama para tu contribucion**

```bash
git checkout -b fix/error-validacion-email
```

**6. Hacer cambios y enviar a tu fork**

```bash
git add .
git commit -m "fix: corregir validacion de email con caracteres especiales"
git push origin fix/error-validacion-email
```

**7. Abrir PR desde tu fork al upstream**

Desde la interfaz de GitHub, seleccionas tu rama y la rama destino del proyecto original.

## Mantener tu fork sincronizado

El mayor problema de los forks es que se desactualizan. Si el proyecto original avanza y tu fork no, tu solicitud de fusion tendra conflictos y probablemente sera rechazada.

```bash
# Sincronizar periodicamente
git fetch upstream
git checkout main
git rebase upstream/main
git push origin main
```

**Regla:** antes de abrir una PR, siempre sincroniza con upstream para tener los cambios mas recientes.

## Convenciones para contribuidores

- **Leer las guias de contribucion** antes de empezar (`CONTRIBUTING.md`).
- **Abrir una incidencia** primero si el cambio es significativo, para debatir el enfoque antes de escribir codigo.
- **Ramas descriptivas**: `fix/bug-login` en lugar de `mi-rama`.
- **Commits limpios**: mensajes descriptivos, un commit logico por cambio.
- **Tamano razonable**: cambios pequenos y enfocados son mas faciles de revisar.

> El modelo de forks demuestra la filosofia del codigo abierto: cualquiera puede copiar, modificar y proponer cambios sin pedir permiso. La calidad del proyecto se mantiene porque cada propuesta pasa por revision antes de aceptarse, pero la puerta esta abierta para todos.

