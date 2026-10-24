# Que es coverage

El code coverage (cobertura de codigo) es una metrica que indica que porcentaje del codigo fuente de una aplicacion ha sido ejecutado durante la ejecucion de los tests. Se expresa tipicamente como un porcentaje que va desde 0 % (ninguna linea ejecutada) hasta 100 % (todas las lineas alcanzadas). Herramientas como JaCoCo, Cobertura o IntelliJ se encargan de instrumentar el bytecode o el codigo fuente para registrar que instrucciones se disparan en cada ejecucion de test.

La utilidad del coverage no reside en el numero en si mismo, sino en lo que revela sobre las zonas inexploradas del codigo. Un modulo con un 30 % de coverage indica que existen ramas, condiciones o flujos completos que nunca se verifican. Esa informacion permite dirigir los esfuerzos de testing hacia las areas de mayor riesgo. No obstante, convertir el coverage en un objetivo numerico ciego genera comportamiento perverso: escribir tests que toquen lineas sin verificarlas realmente, o peor aun, eliminar codigo valioso para inflar el porcentaje.

> La cobertura dice que lineas se ejecutaron, no que se probaron. Un test que invoca un metodo sin assert alguno sigue contando como cobertura, pero no aporta valor.

El coverage como metrica no es objetiva porque dos implementaciones distintas del mismo algoritmo pueden producir porcentajes radicalmente diferentes siendo igualmente correctas. Ademas, el coverage no mide la calidad de las aserciones, la profundidad de los escenarios ni la deteccion de bugs logicos. Una base de codigo con 100 % de cobertura puede seguir teniendo errores graves si los tests solo repiten la misma logica sin cuestionarla.

En la practica, el code coverage es una herramienta de diagnostico, no un fin. Equipos maduros lo usan para identificar puntos ciegos, no para establecer bonus o penalizaciones. La combinacion de coverage con tecnicas como mutation testing ofrece una imagen mucho mas realista de la efectividad de la suite de tests.
