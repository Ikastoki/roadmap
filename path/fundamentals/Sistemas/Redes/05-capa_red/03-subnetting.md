# Subnetting

El subnetting es el proceso de dividir una red en subredes mas pequenas utilizando mascaras de subred mas largas. Permite optimizar la asignacion de direcciones IP, mejorar el rendimiento de la red reduciendo los dominios de broadcast y mejorar la seguridad segmentando el trafico.

El subnetting es una de las habilidades mas fundamentales y practicas en la administracion de redes. Sin subnetting, una red corporativa con 500 dispositivos tendria que estar en un unico dominio de broadcast, causando trafico innecesario y problemas de rendimiento.

## Por que subdividir redes

Las razones principales para subdividir una red son:

- **Reducir el dominio de broadcast:** cada subred tiene su propio dominio de broadcast. En una red grande con 1000 dispositivos, cada broadcast afecta a todos los dispositivos. Con subnetting, cada subred puede tener 100-200 dispositivos, reduciendo el trafico broadcast en un 80-90%.
- **Seguridad:** diferentes subredes pueden tener politicas de firewall diferentes. El trafico entre subredes pasa por un router donde se pueden aplicar filtros.
- **Eficiencia en la asignacion de direcciones:** subnetting permite asignar rangos de direcciones segun la necesidad real de cada departamento o zona.
- **Rendimiento:** routers entre subredes pueden aplicar QoS y gestionar el trafico de forma mas granular.

## Ejemplos practicos de subnetting

### Ejemplo 1: /24 en subredes /26

Red original: 10.0.0.0/24 (254 hosts posibles)

Mascara /26 significa 26 bits de red y 6 bits de host:

```
2^6 - 2 = 62 hosts por subred
2^2 = 4 subredes disponibles

Subred 1: 10.0.0.0/26     → broadcast: 10.0.0.63     → hosts: 10.0.0.1 - 10.0.0.62
Subred 2: 10.0.0.64/26    → broadcast: 10.0.0.127    → hosts: 10.0.0.65 - 10.0.0.126
Subred 3: 10.0.0.128/26   → broadcast: 10.0.0.191    → hosts: 10.0.0.129 - 10.0.0.190
Subred 4: 10.0.0.192/26   → broadcast: 10.0.0.255    → hosts: 10.0.0.193 - 10.0.0.254
```

### Ejemplo 2: /24 en subredes /28

```
2^4 - 2 = 14 hosts por subred
2^4 = 16 subredes disponibles

Subred 1:  10.0.0.0/28     → hosts: .1  - .14
Subred 2:  10.0.0.16/28    → hosts: .17 - .30
Subred 3:  10.0.0.32/28    → hosts: .33 - .46
...
Subred 16: 10.0.0.240/28   → hosts: .241 - .254
```

### Ejemplo 3: red punto a punto /30

```
2^2 - 2 = 2 hosts por subred (ideal para enlaces punto a punto)

Subred 1: 192.168.1.0/30   → hosts: .1 y .2
Subred 2: 192.168.1.4/30   → hosts: .5 y .6
Subred 3: 192.168.1.8/30   → hosts: .9 y .10
```

Los enlaces punto a punto entre routers tipicamente utilizan /30 o /31 para no desperdiciar direcciones.

## Formula rapida

Para calcular el numero de hosts en una subred con mascara /n:

```
Hosts = 2^(32-n) - 2

/24 → 2^8 - 2 = 254 hosts
/25 → 2^7 - 2 = 126 hosts
/26 → 2^6 - 2 = 62 hosts
/27 → 2^5 - 2 = 30 hosts
/28 → 2^4 - 2 = 14 hosts
/29 → 2^3 - 2 = 6 hosts
/30 → 2^2 - 2 = 2 hosts (punto a punto)
```

Para calcular cuantas subredes se obtienen al prestar m bits de una mascara original /n:

```
Subredes = 2^m
```

> El subnetting es una habilidad que se practica con ejercicios y se perfecciona con la experiencia. Herramientas como calculadoras de subnet (subnetcalculator.com) son utiles para verificar resultados, pero un administrador de redes competente debe poder hacer los calculos mentalmente para resolver problemas rapidamente en campo.

