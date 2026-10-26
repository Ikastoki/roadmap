# Ramas y Fusion

La forma en que un equipo organiza sus ramas de codigo determina la velocidad y la fiabilidad de su pipeline de integracion continua. Un modelo de ramas incorrecto puede crear cuellos de botella, conflictos de fusion frecuentes y un ciclo de despliegue que se mide en semanas en lugar de horas. La integracion continua, en su definicion original, implica que multiples desarrolladores integren su trabajo en una rama compartida con la mayor frecuencia posible. Cualquier estrategia de ramas que aleje ese momento de integracion esta en conflicto con el proposito de CI.

La eleccion del modelo de branching no es solo una decision tecnica, es una decision organizativa que afecta como el equipo colabora, como resuelve conflictos y como mantiene la calidad del codigo en el tiempo. No existe un modelo universalmente correcto: la mejor estrategia depende del tamano del equipo, la frecuencia de despliegue, la criticidad del sistema y la madurez de las practicas de CI/CD.

## Trunk-based development

El desarrollo basado en tronco (trunk-based development) es el modelo mas alineado con la integracion continua. En este modelo, todos los desarrolladores trabajan en una unica rama (tipicamente `main` o `trunk`) y los commits se integran directamente en ella despues de una revision rapida. Las ramas de caracteristicas, si se usan, son de muy corta duracion (horas, no dias) y se fusionan al tronco tan pronto como el cambio esta listo.

La velocidad de integracion es la ventaja fundamental de este modelo. Al fusionar multiples veces al dia, los conflictos de merge son pequenos y frecuentes, lo cual es significativamente mejor que conflictos grandes e infrecuentes. Cuando un desarrollador no integra su cambio durante una semana, acumula diferencias con el tronco que dificultan la fusion y aumentan el riesgo de regresiones. La integracion continua funciona mejor cuando las integraciones son pequenas y continuas.

La implementacion de trunk-based development requiere practicas complementarias que compensan la ausencia de ramas de larga duracion. Feature flags permiten que codigo incompleto se fusione al tronco sin afectar a usuarios finales: el codigo se despliega pero se encuentra desactivado hasta que este completo. Los cambios atomicos garantizan que cada commit sea un cambio pequeno y autocontenido que no rompe la compilacion ni las pruebas existentes. Las ramas de release son ramas temporales creadas desde un commit especifico del tronco para gestionar despliegues a produccion, pero se eliminan despues de su uso.

> Trunk-based development no significa que todo se fusione sin verificacion. Significa que la verificacion es rapida (CI pipeline) y la fusion es frecuente, no que se omita la revision de codigo.

## GitFlow

GitFlow es un modelo de branching estructurado que organiza el trabajo en ramas con propositos especificos: `main` (produccion), `develop` (integracion), `feature/*` (nuevas funcionalidades), `release/*` (preparacion de version) y `hotfix/*` (correcciones urgentes). Fue disenado para proyectos con ciclos de release predefinidos y versiones multiples mantenidas en paralelo.

El modelo proporciona claridad sobre el estado de cada cambio: una rama `feature` esta en desarrollo, una rama `release` esta en validacion, un `hotfix` es una correccion urgente. Esta claridad es valiosa en equipos grandes con multiples proyectos activos y ciclos de release largos. Sin embargo, la complejidad del modelo crea overhead significativo: la cantidad de ramas activas, las fusiones frecuentes entre ramas de desarrollo y la necesidad de mantener multiples ramas actualizadas ralentizan el ciclo de desarrollo.

GitFlow entra en tension directa con la integracion continua. La existencia de una rama `develop` separada de `main` crea un punto de integracion secundario que no es el tronco de produccion. Los cambios pueden permanecer en `develop` durante dias o semanas antes de fusionarse a `main`, lo cual contradice el principio de integrar con frecuencia. Para equipos con despliegues continuos, GitFlow añade una capa de complejidad que no aporta valor.

## GitHub Flow

GitHub Flow es un modelo simplificado que solo utiliza la rama `main` y ramas de caracteristicas. Cada cambio se realiza en una rama derivada de `main`, se sube al repositorio remoto y se abre una solicitud de fusion (pull request). El pull request es el punto de revision, discusion y verificacion antes de que el codigo se fusione a `main`. Una vez fusionado, el codigo se despliega automaticamente.

Este modelo es ideal para equipos con despliegues continuos que trabajan en aplicaciones con una unica linea de version. La simplicidad de solo dos tipos de ramas reduce la complejidad cognitiva y elimina las fusiones frecuentes entre ramas de desarrollo que caracterizan a GitFlow. El pull request funciona como la puerta de calidad: el codigo debe pasar todas las verificaciones automaticas y la revision humana antes de integrarse.

La debilidad de GitHub Flow radica en proyectos que requieren mantener multiples versiones en produccion o que tienen ciclos de release planificados. Sin ramas de release dedicadas, la gestion de versiones paralelas requiere convenciones adicionales o herramientas externas.

## Pull requests como puertas de calidad

El pull request (o merge request en terminologia de GitLab) es el mecanismo central donde convergen la revision de codigo, las verificaciones automaticas y la decision de fusion. Un pull request no es solo un pedido para fusionar ramas: es un contrato que establece que el codigo propuesto cumple con los estandares del proyecto antes de integrarse al tronco.

Las verificaciones obligatorias antes de la fusion son la columna vertebral del pull request como puerta de calidad. Estas verificaciones incluyen:

- Compilacion exitosa del proyecto
- Ejecucion de todas las pruebas unitarias y de integracion
- Analisis estatico de codigo sin errores criticos
- Cobertura de codigo por encima del umbral establecido
- Revision por al menos un miembro del equipo
- Resolucion de todos los comentarios pendientes

Los sistemas de integracion continua como GitHub Actions, GitLab CI o Jenkins se integran directamente con el sistema de control de versiones para bloquear la fusion si alguna verificacion falla. Esta integracion es lo que transforma al pull request de un formulario de solicitud en una puerta de calidad automatica e inflexible.

> Un pull request sin verificaciones obligatorias es una formalidad que se puede ignorar. Las verificaciones automaticas son lo que le dan poder real al mecanismo.

## Resolucion de conflictos en CI

Los conflictos de merge son inevitables cuando multiples desarrolladores modifican el mismo archivo simultaneamente. En un entorno de CI, la resolucion de conflictos es un proceso que debe ser rapido, correcto y trazable. La frecuencia de los conflictos es inversamente proporcional a la frecuencia de las integraciones: integrar mas a menudo produce conflictos mas pequenos y mas faciles de resolver.

Cuando un conflicto ocurre, el sistema de control de versiones requiere intervencion manual para determinar que cambios prevalecen. En el contexto de CI, esta resolucion debe ocurrir antes de que las verificaciones automaticas se ejecuten sobre el codigo fusionado. Algunos flujos de trabajo resuelven el conflicto localmente antes de subir, mientras que otros usan herramientas de resolucion en la interfaz web del repositorio.

La mejor estrategia para minimizar conflictos es combinar integraciones frecuentes con cambios atomicos. Un commit pequeno que modifica un solo modulo tiene menos probabilidades de conflicto que un commit grande que toca multiples archivos. La disciplina de hacer pull del tronco antes de empezar a trabajar, y de empujar cambios con frecuencia, reduce dramaticamente la frecuencia y el tamano de los conflictos.

