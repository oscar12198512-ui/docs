# Decisiones iniciales de arquitectura

- Repositorio: un monorepo para interfaces, API, paquetes compartidos y documentación.
- MVP: backend modular; no microservicios prematuros.
- Datos: PostgreSQL como opción inicial para transacciones de compras, cuotas y pagos; revisar requisitos antes de producción.
- Frontends: tres interfaces web separadas con componentes compartidos.
- Crédito y pagos: modo de simulación hasta revisión legal, acuerdos con proveedores y pruebas independientes.
- Seguridad: mínimo privilegio, secretos fuera del repositorio, auditoría, idempotencia y protección de datos.
- Despliegue: definir después de confirmar el stack y las necesidades; Render es una opción, no una configuración ya ejecutada.
