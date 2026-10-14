# Decisiones iniciales de arquitectura

- **Estructura del repositorio**: Monorepo gobernado con `npm workspaces` nativo (`apps/*`, `packages/*`, `services/*`).
- **Arquitectura del backend**: Backend modular monolítico en `services/api` utilizando **Fastify** con TypeScript estructurado mediante plugins por módulo de negocio. No se introducen microservicios prematuros.
- **Interfaces web**: Tres aplicaciones web desacopladas en `apps/` desarrolladas con **Vite + React + TypeScript** (`cliente-web`, `portal-comercios`, `panel-admin`) con diseño responsivo, accesible y componentes compartidos en `packages/ui`.
- **Manejo de dinero y datos contables**: Se establece el tipo estricto `Money` en `@megaproyecto/schemas`, exigiendo importes enteros en unidades menores (centavos) para erradicar errores de coma flotante.
- **Base de datos transaccional**: PostgreSQL como motor objetivo para transacciones, compras y planes de amortización; en Fase 1 no se exige base de datos activa para el healthcheck.
- **Entorno de simulación obligatorio**: Modo de simulación explícito y activo (`isSimulationMode: true`) con avisos visibles en todas las interfaces y respuestas de la API. Ninguna operación ejecuta préstamos ni pagos reales.
- **Seguridad y auditoría**: Principio de mínimo privilegio, segregación estricta de datos por comercio (multitenancy isolation), cero secretos en Git y registro auditable de eventos administrativos y financieros.

