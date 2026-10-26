# Que es CI/CD

El desarrollo de software moderno exige capacidad de entrega continua: las organizaciones que despliegan con frecuencia detectan errores antes, reducen el impacto de los fallos y responden con mayor agilidad a los cambios del mercado. CI/CD es el conjunto de practicas y herramientas que automatizan el camino del codigo fuente hasta su ejecucion en produccion, eliminando los cuellos de botella manuales que historicamente ralentizaban el ciclo de vida del software. Sin estas practicas, cada despliegue se convierte en un evento estresante y propenso a fallos, mientras que con CI/CD el despliegue rutinario se vuelve tan seguro como ejecutar un comando.

## Integracion Continua (CI)

La integracion continua consiste en que cada desarrollador fusione su codigo con la rama principal del repositorio con la mayor frecuencia posible, idealmente varias veces al dia. Cada una de esas fusiones automaticamente dispara un proceso que compila el proyecto, ejecuta los tests unitarios y de integracion, y reporta el resultado en cuestión de minutos. El proposito fundamental es detectar incompatibilidades entre cambios lo antes posible, cuando el conflicto aun es pequeno y facil de resolver. Si la integracion se realiza cada semana o cada mes, los conflictos se acumulan y convertir la fusion en una tarea compleja y riesgosa. Con la integracion continua, la integracion del codigo deja de ser un hito y pasa a ser un proceso continuo que ocurre sin friction.

## Despliegue Continuo (Continuous Delivery)

El despliegue continuo extiende la integracion continua al automatizar el resto del pipeline despues de que los tests pasan. Cuando el codigo supera todas las verificaciones (compilacion, tests, analisis de calidad), el sistema genera un artefacto listo para produccion y lo despliega automaticamente en un entorno de staging o pre-produccion. Sin embargo, la decision final de llevar ese artefacto a produccion requiere aprobacion humana. Este punto de decision es lo que distingue al delivery del deployment: el equipo mantiene el control sobre cuando y como se libera el software, pero el proceso tecnico que lo prepara ya esta completamente automatizado. Es como tener un avion que puede aterrizar solo, pero el piloto aprueba el aterrizaje final.

## Despliegue Automatico (Continuous Deployment)

El despliegue automatico va un paso mas alla: elimina completamente la decision humana del proceso de liberacion. Cada cambio que supera exitosamente todas las etapas del pipeline se despliega directamente en produccion sin intervencion de nadie. Esto requiere una confianza absoluta en la calidad de los tests, el monitoreo y los mecanismos de rollback. Las empresas que operan a escala como Netflix, Google o Amazon despliegan miles de veces al dia precisamente porque cada despliegue es seguro, atomico y reversible. El continuous deployment no es adecuado para todos los contextos: proyectos con requisitos regulatorios o ciclos de liberacion planificados pueden beneficiarse mas del continuous delivery con despliegue manual aprobado.

## La progresion natural

La evolucion natural de una organizacion en su adopcion de CI/CD sigue un camino progresivo. Primero se establece la integracion continua: los desarrolladores aprenden a fusionar con frecuencia y el equipo confia en que los tests automatizados detectan problemas. Despues se avanza al despliegue continuo: se automatiza la preparacion del artefacto y su despliegue en entornos intermedios, pero se mantiene la decision de liberacion en manos humanas. Finalmente, si la madurez del proceso lo permite, se adopta el despliegue automatico donde cada cambio validado llega a produccion sin demora. Saltar directamente al despliegue automatico sin pasar por las etapas anteriores es un error comun que generalmente termina en incidentes graves.

> Una analogia util es la linea de ensamblaje de una fabrica. Cada commit del desarrollador es una pieza que entra a la cadena. La integracion continua verifica que la pieza sea compatible con las demas antes de ensamblarla. El despliegue continuo lleva el producto terminado a la linea de empaquetado. Y el despliegue automatico lo coloca directamente en la estanteria del cliente. En cada estacion de la linea, si la pieza no cumple los estandares de calidad, se rechaza automaticamente y el problema se reporta de inmediato al operario que la fabrico.

