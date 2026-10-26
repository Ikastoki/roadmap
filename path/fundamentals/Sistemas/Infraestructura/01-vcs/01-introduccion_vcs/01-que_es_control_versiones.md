# Qué es el control de versiones

El control de versiones (*Version Control System*, VCS) es un sistema que registra los cambios realizados en archivos o conjuntos de archivos a lo largo del tiempo, permitiendo recuperar versiones anteriores, comparar modificaciones y colaborar con multiples personas sin sobrescribir el trabajo de otros. Es una de las herramientas mas fundamentales en el desarrollo de software y la gestion de cualquier proyecto que involucre documentos que evolucionan.

Sin control de versiones, los proyectos se gestionan con copias manuales de seguridad, nombres de archivo como `informe_v2_final_real_final.docx`, y la协作 se realiza por correo electronico o mensajeria, creando confusion, perdida de trabajo y conflictos inevitables.

## Beneficios clave

**Historial auditado:** cada cambio queda registrado con un identificador unico, un autor, una marca temporal y un mensaje que explica el porque del cambio. Esto proporciona una trazabilidad completa del proyecto: quien cambio que, cuando y por que.

**Capacidad de reversión:** si un cambio introduce un error o un comportamiento no deseado, se puede volver a cualquier version anterior del proyecto con un simple comando. No hay miedo a experimentar porque siempre se puede volver atras.

**Colaboracion simultanea:** multiples personas pueden trabajar en el mismo proyecto al mismo tiempo, incluso en los mismos archivos, sin interferirse mutuamente. El sistema gestiona la combinacion de cambios y la resolucion de conflictos.

**Trazabilidad:** cada version del proyecto esta documentada, lo que permite entender como evoluciono el software, por que se tomaron ciertas decisiones y que impacto tuvieron los cambios en el sistema.

## Tipos de sistemas de control de versiones

### Local

Los VCS locales son los mas simples: el sistema almacena los cambios en una base de datos local sin necesidad de un servidor o conexion de red. RCS (*Revision Control System*) es el ejemplo clasico: guarda las diferencias (*deltas*) entre versiones consecutivas de un archivo.

**Ventaja:** no requiere infraestructura adicional.
**Desventaja:** no permite协作; solo una persona puede trabajar en el proyecto a la vez.

### Centralizado (CVCS)

Los sistemas centralizados utilizan un servidor que almacena el historial completo del proyecto. Los clientes se conectan al servidor para obtener la version actual y enviar cambios.

```
[Servidor Central]
    │    │    │
   [C1] [C2] [C3]  (clientes)
```

Ejemplos: CVS (*Concurrent Versions System*), Subversion (SVN), Perforce.

**Ventaja:** todos los usuarios trabajan con la misma version del proyecto; la administracion es centralizada.
**Desventaja:** si el servidor falla, nadie puede enviar cambios ni acceder al historial. La协作 se limita a la velocidad del servidor.

### Distribuido (DVCS)

Los sistemas distribuidos otorgan a cada colaborador una copia completa del repositorio, incluyendo todo el historial. No hay dependencia de un servidor central para trabajar localmente.

```
[Servidor Central]
    │    │    │
   [C1] [C2] [C3]  (cada uno tiene copia completa)
```

Ejemplos: Git, Mercurial.

**Ventaja:** trabajo offline, mayor fiabilidad (cada copia es un backup),协作 mas flexible.
**Desventaja:** mayor consumo de espacio en disco (cada clone contiene todo el historial), curva de aprendizaje mas pronunciada.

> Git ha revolucionado el control de versiones al combinar la potencia de un sistema distribuido con un rendimiento excepcional. Su modelo de datos basado en instantaneas (*snapshots*) en lugar de deltas, y su soporte para ramas ligeras, lo han convertido en el estandar de facto de la industria.

