# Cobertura minima

Establecer un umbral minimo de cobertura es una decision arquitectonica que afecta a la dinamica del equipo, la velocidad de desarrollo y la calidad del producto. No existe un numero magico que funcione para todos los proyectos, pero si hay rangos aceptados por la industria que sirven como referencia.

## 80 % - Alto

Un 80 % de line coverage se considera un nivel saludable. Implica que la mayoria del codigo de produccion se ejecuta durante los tests y que las ramas principales estan cubiertas. Proyectos con este nivel suelen tener una bateria de tests unitarios solida y practicas de testing consolidadas. Mantener este umbral exige disciplina, especialmente en code review, para que cada nueva funcionalidad llegue con sus tests correspondientes.

## 60 % - Aceptable

El 60 % es un minimo razonable para proyectos que comienzan a introducir tests de forma sistematica o para codigo legacy que se esta cubriendo progresivamente. En este rango aun existen zonas importantes sin probar, y el equipo debe priorizar los modulos de mayor riesgo antes que perseguir un numero global. Es preferible un 60 % con buenos tests en modulos criticos que un 80 % repartido uniformemente sin profundidad.

## 40 % - Bajo

Por debajo del 40 % la situacion es preocupante. Significa que mas de la mitad del codigo nunca se ejecuta durante los tests. En proyectos legacy esto es habitual, pero debe activar una alerta: cualquier cambio en zonas no cubiertas conlleva un riesgo alto de regression. El objetivo inmediato no es subir el numero, sino identificar los modulos core y establecer un plan para cubrirlos.

## Cubrir paths criticos

La cobertura global puede enganar. Un proyecto puede tener un 80 % de coverage global pero dejar sin probar el modulo de facturacion, la logica de autenticacion o la capa de persistencia. La estrategia correcta consiste en identificar los paths criticos del negocio y garantizar que esten cubiertos al 100 %, aunque el resto del proyecto tenga menos cobertura.

```java
// Path critico: logica de transferencia bancaria
void transferir(Origen origen, Destino destino, Monto monto) {
    if (!origen.tieneFondos(monto)) {
        throw new FondosInsuficientes();
    }
    origen.debitar(monto);
    destino.acreditar(monto);
    registrarTransaccion(origen, destino, monto);
}
```

Cada linea de este metodo debe ejecutarse y verificarse. No basta con que el test pase; hay que probar el caso sin fondos, el caso exitoso y los escenarios limite (montos cero, montos negativos, cuentas bloqueadas).

> Un 100 % de cobertura sobre codigo trivial no protege contra errores en la logica de negocio. La cobertura debe dirigirse a donde el fallo tiene mayor impacto.

La decision final sobre el umbral debe tomarse en equipo y revisarse periodicamente. Una vez establecido, se incorpora como regla en JaCoCo check y se verifica en CI. Si el build falla por cobertura, el equipo debe decidir si anade tests o ajusta la regla, pero nunca debe ignorar la senal.
