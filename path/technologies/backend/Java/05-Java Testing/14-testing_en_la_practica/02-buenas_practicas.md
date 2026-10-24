# Buenas practicas de testing

Las buenas practicas de testing no son reglas absolutas, sino principios que han demostrado su eficacia a traves de la experiencia. Aplicarlos consistentemente mejora la calidad, mantenibilidad y utilidad de la suite de tests.

## Un assert por test

Cada test debe verificar un unico comportamiento. Esto no significa literalmente una sola llamada `assertThat`, sino un unico concepto. Si un test verifica que el resultado es positivo y que el usuario recibe un email, son dos responsabilidades.

```java
// Un concepto: el saldo se descuenta correctamente
@Test
void descuentaSaldoEnRetiro() {
    Cuenta cuenta = new Cuenta(100);
    cuenta.retirar(30);
    assertThat(cuenta.getSaldo()).isEqualTo(70);
}

// Otro concepto: la transaccion se registra
@Test
void registraTransaccionEnRetiro() {
    Cuenta cuenta = new Cuenta(100);
    cuenta.retirar(30);
    assertThat(cuenta.getUltimaTransaccion()).isNotNull();
}
```

## Tests independientes

Cada test debe poder ejecutarse de forma aislada, en cualquier orden y sin dependencias de otros tests. El estado compartido entre tests mediante variables estaticas o campos de clase es una fuente comun de fallos intermitentes.

```java
// Mal: dependencia entre tests
static List<String> historial = new ArrayList<>();

@Test void t1() { historial.add("A"); }
@Test void t2() { assertThat(historial).contains("A"); } // depende de t1

// Bien: cada test crea sus propios datos
@Test void t1() {
    List<String> historial = new ArrayList<>();
    historial.add("A");
    assertThat(historial).contains("A");
}
```

## Reproducibles

Un test debe producir el mismo resultado siempre que se ejecute en las mismas condiciones. Los tests que dependen de la hora actual, numeros aleatorios no controlados, o servicios externos son intrinsecamente no reproducibles.

```java
// Mal: dependiente del momento actual
@Test
void descuentoFindeSemana() {
    double descuento = servicio.calcularDescuento();
    // el resultado depende de si hoy es sabado o no
}

// Bien: tiempo controlado
@Test
void descuentoFindeSemana() {
    Reloj reloj = mock(Reloj.class);
    when(reloj.esFinDeSemana()).thenReturn(true);
    servicio = new Servicio(reloj);
    assertThat(servicio.calcularDescuento()).isEqualTo(0.20);
}
```

## Auto-verificables

Un test debe decidir por si mismo si pasa o falla, sin intervencion humana. Esto significa que no debe requerir inspeccion visual de archivos, capturas de pantalla, logs, o bases de datos externas.

```java
// Mal: requiere inspeccion visual
@Test
void generaReporte() {
    reporteService.generar("reporte.pdf");
    // hay que abrir el pdf para comprobarlo
}

// Bien: auto-verificable
@Test
void generaReporte() {
    String contenido = reporteService.generarComoString();
    assertThat(contenido).contains("Cliente: C-001");
    assertThat(contenido).contains("Total: 250.00");
}
```

## Rapidos

La suite de tests debe ejecutarse en segundos, no minutos. Los tests lentos se convierten en tests que no se ejecutan. Si un test necesita acceder a una base de datos, usar un mock. Si necesita un servidor web, usar un servidor embebido ligero o un mock.

## De comportamiento, no de implementacion

Un test debe verificar que comportamiento se produce, no como se implementa internamente. Los tests de implementacion se rompen al refactorizar aunque el comportamiento externo sea correcto.

```java
// Mal: test de implementacion
@Test
void usaHashMapInternamente() {
    Almacen almacen = new Almacen();
    assertThat(almacen.getMapaInterno()).isInstanceOf(HashMap.class);
}

// Bien: test de comportamiento
@Test
void almacenaYRecupera() {
    Almacen almacen = new Almacen();
    almacen.guardar("clave", "valor");
    assertThat(almacen.recuperar("clave")).isEqualTo("valor");
}
```

Estas buenas practicas no son dogmas. En situaciones concretas puede tener sentido relajarlas, pero siempre debe saberse por que se esta haciendo y cual es el coste de la decision.
