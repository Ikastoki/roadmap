# Privacidad y licencias

La decision de hacer un repositorio publico o privado, y la eleccion de una licencia, son dos de las decisiones mas importantes que se toman al crear un proyecto. La privacidad determina quien puede ver y contribuir al codigo; la licencia determina que pueden hacer otros con el codigo una vez que lo obtienen. Tomar estas decisiones sin investigar puede generar problemas legales, de seguridad o de adopcion del proyecto.

## Repositorios publicos vs. privados

**Repositorios publicos:**

- Cualquier persona puede ver el codigo, el historial y los archivos.
- Facilitan la colaboracion abierta y la comunidad.
- Son ideales para codigo abierto, portfolios y proyectos educativos.
- GitHub, GitLab y Bitbucket ofrecen repositorios publicos gratuitos.

**Repositorios privados:**

- Solo los usuarios autorizados pueden acceder.
- Son obligatorios para codigo empresarial, propietario o sensible.
- Pueden tener restricciones adicionales (IP whitelisting, SSO, audit logs).
- En GitHub, los repositorios privados son gratuitos pero con limitaciones en funciones gratuitas.

**Recomendacion:** si el codigo no contiene informacion sensible ni propiedad intelectual que deba protegerse, un repositorio publico genera mas beneficios (visibilidad, colaboracion, portafolio).

## Licencias de software

Una licencia define los terminos bajo los cuales otros pueden usar, modificar y distribuir tu codigo. Sin licencia, el codigo es automaticamente propiedad exclusiva del autor bajo la legislacion de derechos de autor, incluso si esta en un repositorio publico.

**Licencias populares:**

| Licencia | Permite uso comercial | Permite modificar | Permite distribuir | Obliga a开源 |
|---|---|---|---|---|
| MIT | Si | Si | Si | No |
| Apache 2.0 | Si | Si | Si | No |
| GPL v3 | Si | Si | Si | Si |
| LGPL v3 | Si | Si | Si | Solo partes modificadas |
| BSD 3-Clause | Si | Si | Si | No |

**MIT:** la licencia mas permisiva. Cualquiera puede usar, modificar y distribuir el codigo sin obligacion de开源. Es la licencia usada por la mayoria de proyectos populares.

**GPL v3:** licencia copyleft. Cualquiera puede usar, modificar y distribuir, pero si distribuyes una version modificada, debes开源 el codigo completo bajo la misma licencia.

**Apache 2.0:** similar a MIT pero incluye proteccion contra patentes. Ideal para proyectos empresariales que quieren ser abiertos sin riesgo de demandas por patentes.

```markdown
<!-- Incluir en el archivo LICENSE del repositorio -->
MIT License

Copyright (c) 2024 Nombre del autor

Permission is hereby granted, free of charge, to any person obtaining a copy
...
```

## Como elegir una licencia

- **Quieres que cualquiera use tu codigo sin restricciones:** MIT o Apache 2.0.
- **Quieres que las mejoras se compartan:** GPL v3.
- **Es un proyecto interno de la empresa:** no necesitas licencia publica (o usa una licencia propietaria).
- **No sabes:** MIT es la opcion segura por defecto.

## Errores comunes

- **No incluir licencia:** el codigo es propietario por defecto, incluso en repositorios publicos.
- **Mezclar licencias incompatibles:** usar codigo GPL en un proyecto MIT puede crear conflictos legales.
- **Olvidar licencias de dependencias:** el uso de librerias con licencias restrictivas (GPL) puede obligar a开源 tu proyecto.

> La licencia es el contrato legal de tu proyecto con el mundo. Sin ella, tu codigo esta protegido pero inutilizable por otros. Con la licencia correcta, tu proyecto puede ser adoptado, mejorado y compartido por una comunidad global. Tomate el tiempo de elegir bien desde el primer dia.

