# Old Date (Calendar)

Antes de Java 8, el manejo de fechas y tiempos en Java se realizaba mediante `java.util.Date` y `java.util.Calendar`, un diseno con deficiencias arquitectonicas que la API `java.time` vino a corregir. Aunque hoy estan practicamente obsoletas, es frecuente encontrarlas en codigo legacy y en bibliotecas que aun no han migrado.

## java.util.Date

`Date` representa un instante especifico en el tiempo con precision de milisegundos. A pesar de su nombre, no modela una fecha calendario, sino un momento absoluto (epoch: 1 de enero de 1970, 00:00:00 GMT).

```java
Date ahora = new Date(); // momento actual
Date fecha = new Date(1680000000000L); // desde milisegundos epoch
```

Problemas de `Date`:

- Mutabilidad: los metodos `set*()` modifican el objeto, lo que lo hace inseguro en concurrencia.
- Indices de mes basados en 0 (enero = 0, diciembre = 11).
- Anio offset desde 1900: `getYear()` devuelve 123 para 2023.
- Mayorita de metodos `get*()`, `set*()` deprecados desde Java 1.1.
- No distingue entre fecha, hora o zona horaria.

```java
Date date = new Date();
int anio = date.getYear() + 1900;  // obsoleto
int mes = date.getMonth();          // 0-indexado, obsoleto
int dia = date.getDate();           // obsoleto
```

## java.util.Calendar

`Calendar` fue introducido en Java 1.1 como reemplazo de `Date`, pero su diseno sigue siendo problematico.

### Obtencion y uso basico

```java
Calendar cal = Calendar.getInstance(); // fecha/hora actual, zona local
Calendar calUTC = Calendar.getInstance(TimeZone.getTimeZone("UTC"));
```

`Calendar.getInstance()` devuelve una subclase concreta -- tipicamente `GregorianCalendar` -- configurada con la zona horaria y locale por defecto.

### Metodos get y set

```java
Calendar cal = Calendar.getInstance();
int anio = cal.get(Calendar.YEAR);
int mes = cal.get(Calendar.MONTH); // 0-indexado (0 = enero)
int dia = cal.get(Calendar.DAY_OF_MONTH);
int hora = cal.get(Calendar.HOUR_OF_DAY);
int min = cal.get(Calendar.MINUTE);
int seg = cal.get(Calendar.SECOND);
int ms = cal.get(Calendar.MILLISECOND);
int dow = cal.get(Calendar.DAY_OF_WEEK); // 1 = domingo (Calendar.SUNDAY)
```

`Calendar.MONTH` sigue usando indices basados en 0, lo que genera errores frecuentes.

```java
cal.set(2023, Calendar.DECEMBER, 25); // 25-diciembre-2023
cal.set(Calendar.YEAR, 2024);
cal.set(Calendar.MONTH, Calendar.JANUARY);
cal.set(Calendar.DAY_OF_MONTH, 1);
```

### Operaciones con fechas

```java
cal.add(Calendar.DAY_OF_MONTH, 5);    // adelantar 5 dias
cal.add(Calendar.MONTH, -1);          // retroceder 1 mes
cal.add(Calendar.YEAR, 2);            // adelantar 2 anios
```

`add()` propaga los cambios correctamente: aniadir un mes al 31 de enero produce 28/29 de febrero.

```java
cal.set(Calendar.DAY_OF_MONTH, 5);    // fijar dia sin propagar
```

`roll()` funciona como `add()` pero no propaga al campo superior:

```java
cal.roll(Calendar.MONTH, 1); // cambia el mes pero no afecta al anio
```

### Conversion a Date

```java
Date fecha = cal.getTime();           // Calendar -> Date
cal.setTime(new Date());              // Date -> Calendar
long millis = cal.getTimeInMillis();  // Calendar -> long
cal.setTimeInMillis(System.currentTimeMillis());
```

## java.text.SimpleDateFormat

`SimpleDateFormat` permite formatear y analizar fechas como cadenas.

```java
SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy");
String formateado = sdf.format(new Date());
Date parseado = sdf.parse("25/12/2023");
```

Patrones comunes:

| Patron | Significado | Ejemplo |
|--------|-------------|---------|
| yyyy   | Anio (4 digitos) | 2023 |
| MM     | Mes (2 digitos) | 12 |
| dd     | Dia (2 digitos) | 25 |
| HH     | Hora 0-23 | 14 |
| mm     | Minutos | 30 |
| ss     | Segundos | 45 |
| z      | Zona horaria | CET |
| Z      | Offset RFC 822 | +0100 |

> `SimpleDateFormat` no es thread-safe. Usar una instancia compartida entre hilos produce resultados impredecibles. La solucion clasica era sincronizar o usar `ThreadLocal`.

## Problemas de mutabilidad y thread-safety

Tanto `Date` como `Calendar` son mutables, y `SimpleDateFormat` mantiene estado interno durante el formateo. Esto tiene consecuencias graves:

```java
// Problema: Calendar mutable
Calendar cal = Calendar.getInstance();
new Thread(() -> { cal.set(2023, 0, 1); }).start();
new Thread(() -> { cal.set(2024, 11, 31); }).start();
// Ambos hilos modifican el mismo objeto sin sincronizacion
```

```java
// Problema: SimpleDateFormat no thread-safe
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// Usar la misma instancia desde multiples hilos produce datos corruptos
```

La solucion tradicional en codigo legacy:

```java
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

> La API legacy no solo es propensa a errores, sino que carece de conceptos fundamentales como duracion, periodo, fecha sin hora, o zona horaria con cambios de horario de verano. La API `java.time` resuelve todos estos problemas.
