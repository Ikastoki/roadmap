# Assumptions

Las asunciones (assumptions) en JUnit 5 permiten que un test se ejecute solo si se cumplen ciertas condiciones. A diferencia de las aserciones, que verifican resultados y fallan el test si no se cumplen, las asunciones abortan silenciosamente el test cuando la condicion no se satisface, sin marcarlo como fallo. Esta distincion es fundamental para entender cuando usar cada mecanismo.

## Diferencia entre assertion y assumption

Una assertion expresa una condicion que debe ser verdadera para que el sistema sea correcto. Si falla, el test falla, indicando un defecto en el codigo.

Una assumption expresa una condicion que debe ser verdadera para que el test tenga sentido ejecutarse. Si falla, el test se aborta (ignorado), indicando que el entorno o las condiciones no son las adecuadas, no que el codigo tenga un defecto.

```java
@Test
void testSoloEnLinux() {
    assumeTrue(System.getProperty("os.name").toLowerCase().contains("linux"));
    // El resto del test solo se ejecuta en Linux
}
```

En el reporte de ejecucion, un test abortado por una assumption aparece como ignorado (skipped), no como fallido. Esto evita falsos positivos en el pipeline de CI/CD cuando las condiciones ambientales no son las esperadas.

## assumeTrue

Verifica que una condicion es verdadera. Si es falsa, el test se aborta. Acepta tanto un valor booleano directo como un Supplier<Boolean> para evaluacion perezosa, y un mensaje opcional de motivo.

```java
@Test
void testRequeridoModuloEspecifico() {
    assumeTrue(ModuloManager.estaDisponible("modulo_avanzado"),
        "Modulo avanzado no disponible, test abortado");
    servicio.moduloAvanzado().ejecutar();
}
```

## assumeFalse

La contraparte de assumeTrue: aborta el test si la condicion es verdadera. Es util para saltar tests en entornos especificos donde ciertas funcionalidades no aplican.

```java
@Test
void testNoAplicaEnEntornoDeDesarrollo() {
    assumeFalse("development".equals(System.getenv("ENV")),
        "Test no aplica en entorno de desarrollo");
    servicio.productivo.verificar();
}
```

> Las asunciones se evalúan en el orden en que aparecen en el test. Una vez que una assumption falla, el test se aborta inmediatamente y no se ejecutan las lineas siguientes ni las aserciones.

## Cuando usar assumptions

El caso de uso principal de las assumptions es adaptar los tests al entorno de ejecucion sin modificar la logica del test ni introducir condicionales que reduzcan la legibilidad. Algunos escenarios tipicos incluyen:

- **Sistema operativo:** tests que dependen de funcionalidades especificas de Windows, Linux o macOS.
- **Variables de entorno:** tests que requieren ciertas variables configuradas (API keys, rutas, perfiles de despliegue).
- **Recursos disponibles:** tests que verifican funcionalidades opcionales que pueden no estar presentes (modulos licenciados, hardware especializado, servicios externos).
- **Zona horaria o locale:** tests de internacionalizacion que deben ejecutarse en configuraciones regionales especificas.
- **Version de Java:** tests que aprovechan APIs introducidas en versiones recientes y deben omitirse en versiones anteriores.

```java
@Test
void testFuncionalidadJava17() {
    assumeTrue(Runtime.version().feature() >= 17,
        "Requiere Java 17+");
    // Codigo que usa records, pattern matching, etc.
}

@Test
void testInternacionalizacionEspanol() {
    assumeTrue(Locale.getDefault().equals(new Locale("es", "ES")));
    servicio.generarFactura().verificarFormatoEspanol();
}
```

Las assumptions tambien son utiles en combinacion con tags y perfiles de ejecucion. Mientras que las tags permiten incluir o excluir tests en tiempo de compilacion (mediante la configuracion del build), las assumptions ofrecen un control dinamico en tiempo de ejecucion basado en condiciones que solo pueden evaluarse en el momento de la ejecucion.
