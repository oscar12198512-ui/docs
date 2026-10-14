# Prompt inicial para Antigravity

Actúa como arquitecto de software senior y líder técnico. Trabaja sobre esta carpeta local y construye de forma incremental una plataforma original de servicios financieros y comercio electrónico, con interfaz de clientes, portal de comercios, panel administrativo y backend modular.

## Antes de modificar
1. Inspecciona el contenido real de la carpeta y detecta herramientas instaladas, archivos existentes y posibles proyectos previos. No borres ni sobrescribas código existente sin explicar el impacto y pedir confirmación.
2. Resume el estado actual, las decisiones que faltan y un plan por etapas. Evita regenerar todo el repositorio si ya hay código útil.
3. No inventes detalles de las empresas de referencia. Mantén la investigación de mercado separada de las decisiones confirmadas del producto.

## Arquitectura objetivo
- Monorepo con `apps/cliente-web`, `apps/portal-comercios`, `apps/panel-admin`, `services/api`, `packages`, `database`, `infrastructure`, `docs` y `tests`.
- Preferir TypeScript y un gestor de paquetes con workspaces, siempre que el entorno existente lo permita.
- Interfaces web responsivas y accesibles; backend modular; PostgreSQL para datos transaccionales.
- Mantener una estructura sencilla para un MVP; no crear microservicios independientes sin una justificación concreta.
- Crear `README.md`, documentación de arquitectura, `.env.example`, scripts de desarrollo y pruebas.

## Módulos previstos
1. Autenticación, sesiones y permisos por rol.
2. Perfiles de clientes y estado de verificación de identidad.
3. Solicitudes y límites de crédito con decisiones explicables y revisión manual.
4. Compras, planes de cuotas y calendario de vencimientos.
5. Pagos, asignación de pagos, conciliación y libro de movimientos.
6. Comercios, sucursales, productos, transacciones y liquidaciones.
7. Compras internacionales, cotizaciones, órdenes y logística.
8. Contratos versionados y registro de aceptación.
9. Notificaciones, soporte, auditoría y reportes.

## Seguridad y finanzas
- No activar préstamos, crédito real, captación de fondos ni pagos reales durante el MVP de demostración.
- Usar proveedores autorizados y confirmar con asesoría legal los requisitos aplicables antes de operar.
- Nunca guardar contraseñas en texto plano, secretos en Git ni datos completos de tarjetas.
- Aplicar autorización del lado del servidor, validación de entrada, limitación de solicitudes, protección de sesiones y registros sin secretos.
- Usar DECIMAL o unidades monetarias enteras; no usar cálculos monetarios basados en coma flotante sin control.
- Diseñar operaciones de pago idempotentes, con estados explícitos, transacciones y reconciliación. No marcar pagos como confirmados por una simple redirección del navegador.
- Mantener un libro de movimientos con historial auditable; no permitir borrado silencioso de movimientos financieros.
- Separar datos por comercio y probar que un comercio no puede leer datos de otro.
- Datos ficticios solamente en desarrollo y demostraciones.

## Forma de trabajar para ahorrar tiempo y créditos
- Trabaja una fase a la vez y presenta cambios pequeños, verificables y coherentes.
- Antes de instalar dependencias, explica cuáles hacen falta y por qué.
- No refactorices archivos no relacionados con la tarea actual.
- No inventes credenciales, API keys, proveedores ni contratos.
- Ejecuta las pruebas y el lint disponibles después de cada fase; informa qué se ejecutó y qué falló.
- Si falta información crítica, deja un TODO documentado y pregunta en vez de inventarla.
- No afirmes que desplegaste, conectaste servicios o modificaste cuentas externas si no lo hiciste realmente.

## Primera tarea
Implementa solamente la base del monorepo: inspección del entorno, propuesta de stack, configuración de workspaces, archivos README, configuración de TypeScript/lint/pruebas apropiada, páginas de bienvenida mínimas para las tres interfaces y un backend de salud (`/health`). No implementes aún crédito real, pagos reales ni integraciones financieras. Asegura que los comandos de instalación, desarrollo y pruebas queden documentados. Al terminar, informa los archivos creados, comandos ejecutados, resultados y el siguiente paso sugerido.
