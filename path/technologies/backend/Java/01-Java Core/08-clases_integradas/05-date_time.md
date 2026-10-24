# Date (Time)

El paquete `java.time` introducido en Java 8 (JSR-310) disenado por Stephen Colebourne, creador de Joda-Time, reemplaza por completo a `java.util.Date` y `java.util.Calendar`. Ofrece un modelo de datos inmutable, fluido y semanticamente rico para representar fechas, horas, instantes, duraciones y periodos.

## Ventajas frente a Date/Calendar

- **Inmutablebilidad**: todas las clases son inmutables y thread-safe.
- **Separacion de conceptos**: fecha, hora, fecha-hora, instante, duracion, periodo son tipos distintos.
- **Metodos semanticos**: `plusDays()`, `minusMonths()`, `withYear()` en lugar de `Calendar.add()`.
- **Soporte de zonas horarias**: manejo correcto de DST, offsets y reglas historicas.
- **Formateo thread-safe**: `DateTimeFormatter` es inmutable y thread-safe.
- **Interfaz fluida**: los metodos retornan nuevas instancias permitiendo encadenamiento.

## LocalDate

Representa una fecha sin hora ni zona horaria (anio, mes, dia).

```java
LocalDate hoy = LocalDate.now();
LocalDate navidad = LocalDate.of(2023, 12, 25);
LocalDate desdeEpoch = LocalDate.ofEpochDay(0); // 1970-01-01
LocalDate parseada = LocalDate.parse("2023-12-25");
```

Operaciones:

```java
LocalDate maniana = hoy.plusDays(1);
LocalDate mesAnterior = hoy.minusMonths(1);
LocalDate inicioAnio = hoy.withDayOfYear(1);
LocalDate siguienteMiercoles = hoy.with(TemporalAdjusters.next(DayOfWeek.WEDNESDAY));

int anio = hoy.getYear();
Month mes = hoy.getMonth(); // enum Month
int diaMes = hoy.getDayOfMonth();
DayOfWeek dow = hoy.getDayOfWeek(); // enum DayOfWeek
int diaAnio = hoy.getDayOfYear();
boolean bisiesto = hoy.isLeapYear();
```

## LocalTime

Representa una hora sin fecha ni zona horaria.

```java
LocalTime ahora = LocalTime.now();
LocalTime almuerzo = LocalTime.of(13, 30, 0);
LocalTime parseado = LocalTime.parse("14:30:00");
```

```java
LocalTime masTarde = ahora.plusHours(2);
LocalTime antes = ahora.minusMinutes(30);
boolean esAntes = LocalTime.of(10, 0).isBefore(LocalTime.of(12, 0));
int hora = ahora.getHour();
```

## LocalDateTime

Combina fecha y hora sin zona horaria.

```java
LocalDateTime ahora = LocalDateTime.now();
LocalDateTime reunion = LocalDateTime.of(2023, 12, 25, 10, 30);
LocalDateTime desdeLocal = LocalDate.now().atTime(LocalTime.NOON);
```

```java
LocalDateTime modificada = reunion.plusWeeks(2).minusHours(1);
```

## ZonedDateTime y ZoneId

Representa una fecha-hora con zona horaria completa, manejando correctamente los cambios de horario de verano (DST).

```java
ZoneId madrid = ZoneId.of("Europe/Madrid");
ZonedDateTime ahoraMadrid = ZonedDateTime.now(madrid);
ZonedDateTime reunionZona = reunion.atZone(madrid);
```

```java
ZoneId ny = ZoneId.of("America/New_York");
ZonedDateTime enNY = ahoraMadrid.withZoneSameInstant(ny);
```

`ZoneId.getAvailableZoneIds()` devuelve todas las zonas disponibles. Para offsets fijos:

```java
ZoneOffset offset = ZoneOffset.of("+02:00");
OffsetDateTime odt = OffsetDateTime.now(offset);
```

## Instant

Representa un instante en la linea de tiempo (epoch + nanosegundos). Es la representacion mas pura del tiempo maquina.

```java
Instant ahora = Instant.now();
Instant epoch = Instant.EPOCH; // 1970-01-01T00:00:00Z
```

Conversiones:

```java
LocalDateTime ldt = LocalDateTime.ofInstant(ahora, ZoneId.systemDefault());
ZonedDateTime zdt = ahora.atZone(ZoneId.of("Europe/Madrid"));
Instant desdeLocal = reunion.toInstant(ZoneOffset.UTC);
```

## Duration y Period

`Duration` mide tiempo basado en segundos/nanosegundos (hasta nanosegundos). `Period` mide tiempo basado en fechas (anios, meses, dias).

```java
Duration d = Duration.ofHours(2).plusMinutes(30);
long segundos = d.getSeconds();
Duration entre = Duration.between(inicio, fin);

Period p = Period.of(1, 6, 15); // 1 anio, 6 meses, 15 dias
long totalDias = ChronoUnit.DAYS.between(inicio, fin);
```

```java
LocalDate nacimiento = LocalDate.of(1990, 5, 15);
Period edad = Period.between(nacimiento, LocalDate.now());
int anios = edad.getYears();
int meses = edad.getMonths();
```

## DateTimeFormatter

Formateo y parseo thread-safe, a diferencia de `SimpleDateFormat`.

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String texto = hoy.format(formatter); // "25/12/2023"
LocalDate parseada2 = LocalDate.parse("25/12/2023", formatter);
```

Formateadores predefinidos:

```java
String iso = hoy.format(DateTimeFormatter.ISO_LOCAL_DATE);
String fechaHora = ahora.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
```

Formateo localizado:

```java
DateTimeFormatter larga = DateTimeFormatter.ofLocalizedDate(FormatStyle.LONG)
    .withLocale(new Locale("es", "ES"));
String textoLargo = hoy.format(larga);
```

## Conversiones con Date/Calendar

Compatibilidad con codigo legacy usando metodos de conversion en `java.util.Date`:

```java
// Legacy -> java.time
Date legacy = new Date();
Instant instante = legacy.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instante, ZoneId.systemDefault());

// java.time -> Legacy
Date deVuelta = Date.from(Instant.now());
Date deLocal = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```

```java
// Calendar -> java.time
Calendar cal = Calendar.getInstance();
Instant inst = cal.toInstant();
ZonedDateTime zdt = inst.atZone(cal.getTimeZone().toZoneId());
```

## ChronoUnit

Operaciones entre unidades temporales:

```java
long dias = ChronoUnit.DAYS.between(inicio, fin);
long semanas = ChronoUnit.WEEKS.between(inicio, fin);
long horas = ChronoUnit.HOURS.between(inicio, fin);
```

## TemporalAdjusters

Operaciones comunes sobre fechas:

```java
LocalDate primerDiaMes = hoy.with(TemporalAdjusters.firstDayOfMonth());
LocalDate ultimoDiaMes = hoy.with(TemporalAdjusters.lastDayOfMonth());
LocalDate siguienteLunes = hoy.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
LocalDate anteriorViernes = hoy.with(TemporalAdjusters.previous(DayOfWeek.FRIDAY));
```
