# Arquitectura de Autenticación, Usuarios, Roles y Seguridad (Fase 3)

## 1. Arquitectura Técnica de Autenticación

La plataforma **Fintech Platform** implementa un modelo de autenticación y autorización desacoplado, modular y altamente seguro, que opera bajo el principio de **mínimo privilegio** y **defensa en profundidad**.

```
   [ Cliente Web ]        [ Portal Comercios ]        [ Panel Admin ]
   (Port 3000)            (Port 3001)                 (Port 3002)
        │                       │                          │
        └───────────────────────┼──────────────────────────┘
                                │ HTTP / JSON (Credentials: include)
                                ▼
                   ┌───────────────────────────┐
                   │   Fastify API (Port 4000) │
                   ├───────────────────────────┤
                   │  - CORS Restrictivo       │
                   │  - Cookie HttpOnly Parser │
                   │  - Logger Redaction       │
                   └─────────────┬─────────────┘
                                 │
                   ┌─────────────┴─────────────┐
                   │    Auth Middleware        │
                   ├───────────────────────────┤
                   │  - authenticate()         │
                   │  - requireRole() [RBAC]   │
                   │  - requireMerchantScope() │
                   └─────────────┬─────────────┘
                                 │
                   ┌─────────────┴─────────────┐
                   │       Auth Service        │
                   ├───────────────────────────┤
                   │  - crypto.scrypt (OWASP)  │
                   │  - timingSafeEqual        │
                   │  - Anti-brute force lock  │
                   │  - Anti-mass assignment   │
                   └─────────────┬─────────────┘
                                 │
                   ┌─────────────┴─────────────┐
                   │    Repository Pattern     │
                   ├───────────────────────────┤
                   │ InMemory (Dev/Tests)      │
                   │ PostgreSQL (Database)     │
                   └───────────────────────────┘
```

### Componentes de Seguridad Criptográfica
1. **Algoritmo de Hash**: `crypto.scrypt` con sal aleatoria criptográfica de 16 bytes y clave derivada de 64 bytes por usuario.
2. **Mitigación de Ataques de Tiempo (Timing Attacks)**: Se utiliza `crypto.timingSafeEqual` para comparar hashes y firmas HMAC, impidiendo inferencias estadísticas sobre el contenido de contraseñas o tokens.
3. **Gestión de Sesiones**: Identificadores de sesión de 32 bytes de alta entropía firmados criptográficamente mediante `crypto.createHmac('sha256', secret)`.
4. **Almacenamiento de Sesión en Navegador**: Cookies `HttpOnly`, `SameSite=Lax`, con directiva `Secure` forzada en producción (`NODE_ENV === 'production'`), imposibilitando el robo de tokens por Cross-Site Scripting (XSS). No se almacenan tokens ni contraseñas en `localStorage`.
5. **Prevención de Enumeración de Cuentas**:
   - En el login: Respuestas uniformes de credenciales inválidas ante correos inexistentes o contraseñas incorrectas, con tiempo de procesamiento computacional parejo.
   - En recuperación de contraseña: Se responde siempre con el mismo mensaje genérico preventivo sin confirmar la existencia del correo en la base de datos.
6. **Protección contra Fuerza Bruta**: Bloqueo automático de cuenta por 15 minutos al acumular 5 intentos fallidos consecutivos (`lockedUntil`).
7. **Redacción de Logs (Log Sanitization)**: Fastify logger configurado para enmascarar automáticamente campos sensibles (`password`, `passwordHash`, `salt`, `token`, `newPassword`, `confirmPassword`, `authorization`, `cookie`).

---

## 2. Modelo de Roles y Permisos (RBAC)

Se definen estrictamente los 7 roles requeridos por la plataforma:

| Rol | Alcance de Autorización | Acceso a Datos de Comercio | Capacidad Administrativa |
| :--- | :--- | :--- | :--- |
| `CLIENT` | Cliente final comprador: consultar créditos propios, compras, cuotas y pagos. | No | No |
| `MERCHANT_OWNER` | Propietario de tienda afiliada: gestionar sucursales, personal, catálogo y liquidaciones de su comercio. | Estrictamente su `merchantId` | No |
| `MERCHANT_STAFF` | Personal de caja de tienda: autorizaciones de compra en mostrador y registro de transacciones. | Estrictamente su `merchantId` | No |
| `SUPPORT` | Mesa de ayuda: atención y resolución de incidencias de usuarios y comercios. | No | No |
| `OPERATIONS` | Personal operativo: supervisión de pedidos internacionales, tracking aduanal y conciliación. | No | No |
| `RISK_ANALYST` | Analista crediticio: evaluación de scoring, revisión de expedientes KYC y límites crediticios. | No | No |
| `ADMIN` | Superadministrador de gobernanza: configuración global, auditoría inmutable y gestión de usuarios. | Global (Multi-tenant audit) | Total |

### Aislamiento Multi-Tenant
- Cada usuario perteneciente a una tienda tiene un identificador forzoso `merchantId`.
- El middleware `requireMerchantScope('merchantId')` valida en el servidor que `req.user.merchantId === req.params.merchantId`. Si un empleado o dueño del Comercio A intenta solicitar información del Comercio B, el servidor rechaza la solicitud con **HTTP 403 Forbidden**.

### Prevención de Asignación Masiva (*Anti-Mass Assignment*)
- El endpoint público de registro `POST /api/v1/auth/register` asigna de forma determinista e irrevocable el rol `CLIENT`. Cualquier atributo `role` enviado en el cuerpo de la petición es descartado por el backend.

---

## 3. Procedimiento Seguro de Aprovisionamiento Administrativo

Para no exponer contraseñas quemadas en el repositorio ni habilitar accesos anónimos, la plataforma incluye el script de aprovisionamiento seguro:

```bash
# Modo 1: Provisión con contraseña definida por variable de entorno
ADMIN_EMAIL="admin@tudominio.com" ADMIN_PASSWORD="MiPasswordSegura#2026" npm run provision:admin

# Modo 2: Provisión automática con generación de contraseña de alta entropía de un solo uso
npm run provision:admin
```

El script genera el hash con `scrypt` y sal criptográfica única y muestra la clave generada **una sola vez por pantalla**, sin escribirla en archivos versionados de Git.

---

## 4. Variables de Entorno Requeridas

| Variable | Descripción | Valor por Defecto (Dev) |
| :--- | :--- | :--- |
| `PORT` | Puerto de escucha de la API Fastify | `4000` |
| `HOST` | Interfaz de red de la API | `0.0.0.0` |
| `NODE_ENV` | Entorno de ejecución (`development` / `production` / `test`) | `development` |
| `AUTH_SECRET` | Secreto criptográfico HMAC para firma de sesiones (mín 32 caracteres) | Clave de desarrollo local |
| `DATABASE_URL` | Cadena de conexión para PostgreSQL | `postgresql://USER:PASSWORD@localhost:5432/megaproyecto_dev` |
| `ADMIN_EMAIL` | Correo del administrador inicial | `admin.oficial@fintech.local` |
| `ADMIN_PASSWORD` | Contraseña del administrador inicial (opcional en script) | Autogenerada en ejecución |

---

## 5. Pruebas Automatizadas Ejecutadas (12 Casos de Seguridad)

La suite de pruebas `services/api/src/modules/auth/auth.test.ts` valida:

1. **Registro válido**: Registro exitoso de usuario con rol `CLIENT` e inicialización de perfil KYC.
2. **Rechazo de entradas inválidas**: Validación de campos vacíos, emails mal formados y longitud/complejidad de contraseñas.
3. **Inicio de sesión correcto**: Emisión de token firmado y cabecera `Set-Cookie` con directivas seguras.
4. **Rechazo de credenciales incorrectas**: Retorno de 401 con mensaje genérico preventivo de enumeración de cuentas.
5. **Cierre de sesión**: Revocación de sesión e invalidación de cookie mediante `Max-Age=0`.
6. **Recuperación de contraseña**: Token temporal de un solo uso, actualización de hash y bloqueo de clave anterior.
7. **Acceso a ruta privada sin autenticación**: Retorno inmediato de 401 Unauthorized.
8. **Rechazo de usuario sin permisos administrativos**: Cliente intentando acceder a ruta de Admin recibe 403 Forbidden.
9. **Imposibilidad de auto-asignarse roles privilegiados**: Payload con `role: 'ADMIN'` es forzado a `CLIENT`.
10. **Aislamiento multi-tenant entre comercios**: Comercio A recibe 403 Forbidden al consultar datos de Comercio B.
11. **Rechazo de sesiones expiradas o adulteradas**: Firmas HMAC inválidas y tokens vencidos son rechazados con 401.
12. **Protección de datos sensibles**: Ni la contraseña, ni su hash ni la sal se exponen en DTOs de perfil o respuestas JSON.

---

## 6. Riesgos y Pendientes Previos a Producción

1. **Configuración de Proveedor de Correo Transaccional (SMTP / SendGrid / Resend)**:
   - En desarrollo, el código de recuperación se expone de forma controlada en el entorno local (`devCode`).
   - Para producción, se debe conectar un proveedor SMTP real con plantillas HTML firmadas con DKIM/SPF.
2. **Mecanismo de Doble Factor de Autenticación (2FA / TOTP)**:
   - La estructura de base de datos soporta los campos de verificación; para producción se recomienda agregar autenticación TOTP (Google Authenticator) para los roles `ADMIN` y `RISK_ANALYST`.
3. **Persistencia Activa en PostgreSQL**:
   - Las migraciones `001` y `002` están listas en `database/migrations/`. Se debe aprovisionar el servidor PostgreSQL en la infraestructura objetivo (Docker/Render/AWS RDS) y ejecutar las migraciones.
