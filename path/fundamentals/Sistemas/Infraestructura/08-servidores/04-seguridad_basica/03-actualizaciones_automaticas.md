# Actualizaciones automaticas

Mantener un servidor actualizado es una de las medidas de seguridad mas efectivas y, paradójicamente, una de las mas descuidadas. Los desarrolladores de distribuciones lanzan parches de seguridad constantemente para corregir vulnerabilidades descubiertas en paquetes del sistema. Un servidor sin actualizar puede llevar semanas o meses con una vulnerabilidad conocida, tiempo suficiente para que un atacante la explote. Sin embargo, actualizar automaticamente implica un compromiso: un parche mal probado puede romper dependencias criticas y provocar caidas en servicios de produccion. La clave esta en configurar las actualizaciones de forma que se apliquen parches de seguridad sin alterar el comportamiento estable del sistema.

## unattended-upgrades (Debian/Ubuntu)

En sistemas Debian y Ubuntu, el paquete `unattended-upgrades` permite configurar actualizaciones automaticas de seguridad sin intervencion manual. El sistema descarga e instala paquetes marcados como actualizaciones de seguridad del repositorio oficial, respetando las restricciones de version para no saltar a una nueva release mayor que podria introducir cambios incompatibles.

La configuracion se encuentra en `/etc/apt/apt.conf.d/50unattended-upgrades`. Las opciones relevantes incluyen el nivel de automatizacion (solo descargar, descargar e instalar, o instalar y reiniciar si es necesario), las distribuciones de las que se aceptan actualizaciones, y la posibilidad de excluir paquetes especificos. Para activar el servicio:

```bash
dpkg-reconfigure -plow unattended-upgrades
```

El servicio se ejecuta periodicamente via `apt-daily-upgrade.timer` y genera logs en `/var/log/unattended-upgrades/`. Es recomendable configurar notificaciones por correo electronico para que el administrador este informado de cada actualizacion aplicada.

> unattended-upgrades solo gestiona actualizaciones de seguridad del repositorio configured. No actualiza paquetes de terceros ni los que se instalaron manualmente con `.deb` descargados de internet.

## dnf-automatic (RHEL/Fedora)

En distribuciones basadas en Red Hat, `dnf-automatic` ofrece funcionalidad similar. El paquete configura un timer systemd que ejecuta `dnf` periodicamente segun las directivas de `/etc/dnf/automatic.conf`.

La configuracion permite establecer si se descargan e instalan actualizaciones automaticamente, si se aplican solo parches de seguridad (`apply_updates = yes` con el tipo de upgrade restringido), o si solo se generan reportes sin instalar nada. Esta ultima opcion es util en entornos donde se prefiere un ciclo de revision manual pero se quiere mantener la informacion actualizada:

```bash
dnf install dnf-automatic
systemctl enable --now dnf-automatic.timer
```

## Equilibrio entre seguridad y estabilidad

La decision entre actualizaciones automaticas y manuales depende del contexto. En servidores de produccion con criticidad alta, muchos administradores prefieren revisar cada parche antes de aplicarlo, probarlo en un entorno de staging y planificar una ventana de mantenimiento. En servidores de desarrollo, laboratorios o sistemas donde la disponibilidad es menos critica, las actualizaciones automaticas de seguridad son casi siempre la mejor opcion: el riesgo de una vulnerabilidad explotada supera con creces el riesgo de un parche que pueda requerir un reinicio o una configuracion adicional.

Una estrategia comun es permitir actualizaciones automaticas de seguridad (que suelen ser parches menores y reversibles) pero mantener las actualizaciones mayores (nuevas versiones de kernel, bases de datos o stacks de aplicacion) bajo control manual. Esto se consigue restringiendo las fuentes de repositorio en la configuracion de `unattended-upgrades` o `dnf-automatic` para que solo apunten a repositorios de seguridad.

