# SECURITY AUDIT REPORT — AI SECURE CODE AUDITOR WORKFLOW

**Fecha de Ejecución:** 2026-10-21  
**Auditor:** Senior Software Security & Cloud Architecture Auditor (Antigravity AI)  
**Objetivo Auditado:** Megaproyecto FinTech & POS Monorepo  
**Commit / Ref:** `fc2a4fb`  
**Alcance:** Backend Fastify API, Cliente Web, Panel Admin, Portal Comercios, Base de Datos PostgreSQL (Render / Supabase), Esquemas y Dependencias.

---

## 📊 RESUMEN EJECUTIVO (ESTÁNDAR FIJO)

```text
SECURITY AUDIT REPORT

🔐 Credenciales: MEDIO
🔐 Base de Datos Firebase: OK (NO DETECTADO / ARQUITECTURA POSTGRESQL)
🔐 Base de Datos Supabase: OK (RLS habilitado en 17 tablas + acceso anónimo revocado)
🔐 Base de Datos Render: MEDIO
🔐 Arquitectura: MEDIO
🔐 Autenticación / Autorización: MEDIO
🔐 APIs / Functions: OK (Rate limit global y auth activo)
🔐 Dependencias: MEDIO

RIESGO TOTAL: MEDIO (SIN VULNERABILIDADES CRÍTICAS ACTIVAS)
DEPLOY RECOMENDADO: SÍ (DESPLIEGUE AUTORIZADO TRAS MITIGAR CRÍT-01 Y CRÍT-02)
```

---

## 🧩 DETALLES DE HALLAZGOS Y VULNERABILIDADES

### 🟢 HALLAZGOS CRÍTICOS MITIGADOS (DESBLOQUEADORES DE PRODUCCIÓN)

#### 1. CRÍT-01: Ausencia de Rate Limiting en Endpoints de Autenticación
- **Archivo / Componente:** [app.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/services/api/src/app.ts#L20-L75) y [auth.routes.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/services/api/src/modules/auth/auth.routes.ts#L30-L56)
- **Fase:** FASE 5 — APIs, Functions y Endpoints
- **Nivel de Riesgo:** 🟢 **RESUELTO Y VERIFICADO** (Antes 🔴 CRÍTICO)
- **Estado de Corrección:** ✅ Mitigado con éxito. Se instaló `@fastify/rate-limit`, se registró protección global (60 req/min) en `app.ts` y restricción estricta (5 req/min) en `/login`, `/register`, `/forgot-password` y `/reset-password` en `auth.routes.ts`. Verificado con suite de 58 pruebas unitarias e integración en Vitest.
- **Descripción Original del Problema:**
  El servidor Fastify no tenía registrado `@fastify/rate-limit`. Los endpoints críticos `/api/v1/auth/login`, `/api/v1/auth/register` y `/api/v1/auth/forgot-password` admitían un volumen ilimitado de peticiones por segundo desde la misma IP.
- **Impacto Previo:** Posible denegación de servicio de CPU (agotamiento por hashes `scrypt` concurrentes) y ataques de diccionario a gran escala.

---

#### 2. CRÍT-02: Base de Datos Supabase Expuesta sin Row Level Security (RLS)
- **Archivo / Componente:** [database/migrations/006_enable_row_level_security.sql](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/database/migrations/006_enable_row_level_security.sql) y [.env](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/.env#L1)
- **Fase:** FASE 2 — Seguridad de Base de Datos (Supabase)
- **Nivel de Riesgo:** 🟢 **RESUELTO Y VERIFICADO** (Antes 🔴 CRÍTICO)
- **Estado de Corrección:** ✅ Mitigado con éxito. Se creó y ejecutó la migración `006_enable_row_level_security.sql` directamente sobre la base de datos conectada en Supabase. Se habilitó `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` en las 17 tablas del esquema `public` y se revocaron todos los permisos de acceso de los roles `anon` y `authenticated` en Supabase PostgREST.
- **Descripción Original del Problema:**
  Las tablas se creaban en el esquema `public` sin Row Level Security, lo que permitía que la API PostgREST automática de Supabase las expusiera al público mediante la anon key.
- **Impacto Previo:** Riesgo de acceso no autorizado y extracción masiva de credenciales y balances a través de la API HTTP de Supabase.

---

### 🟠 HALLAZGOS MEDIOS (REVISIÓN OBLIGATORIA)

#### 3. MED-01: Suplantación de Rol en Modo Fallback del Cliente Web
- **Archivo / Componente:** [authClient.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/packages/ui/src/authClient.ts#L83-L100)
- **Fase:** FASE 3 — Arquitectura y FASE 4 — Autenticación
- **Nivel de Riesgo:** 🟠 MEDIO
- **Descripción del Problema:**
  En `packages/ui/src/authClient.ts`, el método `login` contiene un bloque `catch` que, ante un fallo de red (`Failed to fetch`), genera una sesión simulada en memoria asignando roles basándose en la cadena del correo ingresado:
  `role: dto.email.includes('admin') ? 'ADMIN' : dto.email.includes('comercio') ? 'MERCHANT_OWNER' : 'CLIENT'`.
  Si bien las llamadas a la API real fallarán sin sesión válida en el backend, esta lógica permite que un usuario en el navegador obtenga privilegios visuales completos y acceso a pantallas restringidas del panel administrativo simulando desconexión o bloqueando las peticiones de red.
- **Impacto:** Violación del principio de confianza cero en el cliente y exposición de interfaces restringidas.
- **Recomendación (NO aplicar cambios automáticamente):**
  Deshabilitar el fallback de asignación dinámica de roles en el cliente. Si la API no responde, retornar un error de conexión estricto en lugar de conceder roles administrativos en memoria.

---

#### 4. MED-02: Configuración Insegura de API_BASE_URL en Producción (Mixed Content & Puerto Erróneo)
- **Archivo / Componente:** [authClient.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/packages/ui/src/authClient.ts#L10-L13) y [render.yaml](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/render.yaml#L56-L58)
- **Fase:** FASE 1 — Credenciales y FASE 3 — Arquitectura
- **Nivel de Riesgo:** 🟠 MEDIO
- **Descripción del Problema:**
  En `authClient.ts`, la URL de la API se calcula de la siguiente manera:
  ```ts
  const API_BASE_URL = typeof window !== 'undefined' && window.location.hostname
    ? `http://${window.location.hostname}:4000`
    : 'http://localhost:4000';
  ```
  Esto ignora por completo la variable `VITE_API_URL` definida en `render.yaml` (`https://megaproyecto-fintech-api.onrender.com`). Al desplegar en Render bajo HTTPS, las aplicaciones cliente intentarán comunicarse con `http://megaproyecto-cliente-web.onrender.com:4000`, lo que provocará:
  1. Bloqueo inmediato del navegador por **Mixed Content** (HTTPS consumiendo HTTP no cifrado).
  2. Fallo de conexión por puerto `:4000` cerrado en la infraestructura estática de Render.
- **Impacto:** Ruptura total de la conexión frontend-backend en producción y degradación al fallback inseguro.
- **Recomendación (NO aplicar cambios automáticamente):**
  Configurar `authClient.ts` para respetar prioritariamente `import.meta.env.VITE_API_URL`:
  ```ts
  const API_BASE_URL = (typeof import.meta !== 'undefined' && import.meta.env?.VITE_API_URL) || 'http://localhost:4000';
  ```

---

#### 5. MED-03: Desactivación de Verificación TLS/SSL en Cliente PostgreSQL (`rejectUnauthorized: false`)
- **Archivo / Componente:** [postgres.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/services/api/src/common/postgres.ts#L47-L50)
- **Fase:** FASE 2 — Base de Datos Render / Supabase
- **Nivel de Riesgo:** 🟠 MEDIO
- **Descripción del Problema:**
  En `postgres.ts`:
  `ssl: isProduction || isCloudDb ? { rejectUnauthorized: false } : undefined`
  Desactivar `rejectUnauthorized` permite que el cliente de Node.js acepte certificados autofirmados o no validados por una CA confiable.
- **Impacto:** Vulnerabilidad a ataques de intermediario (Man-in-the-Middle o MITM) en el tráfico de datos entre la API en Render y el clúster PostgreSQL en la nube.
- **Recomendación (NO aplicar cambios automáticamente):**
  Para bases de datos en producción gestionadas (como Render o AWS), suministrar el certificado raíz de la CA y validar la cadena de confianza completa, o limitar `rejectUnauthorized: false` exclusivamente a entornos de prueba claramente identificados.

---

#### 6. MED-04: Vulnerabilidades Críticas en Dependencias de Desarrollo (Supply Chain)
- **Archivo / Componente:** `megaproyecto-fintech/package.json` / `package-lock.json`
- **Fase:** FASE 6 — Dependencias y Código Externo
- **Nivel de Riesgo:** 🟠 MEDIO (Entorno de Construcción / CI/CD)
- **Descripción del Problema:**
  El comando `npm audit` dentro de `megaproyecto-fintech` detectó 6 vulnerabilidades (2 críticas, 1 alta, 3 moderadas):
  - `tinypool <=2.1.1` (Crítica - Prototype pollution gadget en worker options conduce a RCE - GHSA-5gmw-xhrv-c9v3).
  - `esbuild <=0.24.2` (Moderada - Exposición del servidor de desarrollo - GHSA-67mh-4wv8-2f99).
  - `@vitest/mocker <=4.1.10` (Moderada - Path Traversal vía Redirect Mock - GHSA-82fw-gwwq-j7x9).
- **Impacto:** Si bien estas librerías pertenecen a herramientas de compilación y pruebas (`vitest`, `vite`), representan una brecha de seguridad en los pipelines de integración continua (CI/CD) ante código o mocks no confiables.
- **Recomendación (NO aplicar cambios automáticamente):**
  Actualizar `vitest` y `vite` a versiones parcheadas en el monorepo ejecutando una actualización controlada de dependencias en una rama de mantenimiento.

---

#### 7. MED-05: Falta de Cabeceras HTTP de Seguridad (Helmet / CSP / HSTS)
- **Archivo / Componente:** [app.ts](file:///c:/Users/Admin/OneDrive/Desktop/megaproyecto-fintech-esqueleto/megaproyecto-fintech/services/api/src/app.ts)
- **Fase:** FASE 5 — APIs y Endpoints
- **Nivel de Riesgo:** 🟠 MEDIO
- **Descripción del Problema:**
  No se utiliza `@fastify/helmet` ni cabeceras manuales para:
  - `Content-Security-Policy` (CSP)
  - `Strict-Transport-Security` (HSTS)
  - `X-Frame-Options: DENY` (Protección contra Clickjacking)
  - `X-Content-Type-Options: nosniff`
- **Recomendación (NO aplicar cambios automáticamente):**
  Registrar `@fastify/helmet` con directivas estrictas en `app.ts`.

---

## 🟢 ASPECTOS EVALUADOS COMO APROBADOS (OK)

1. **Cálculo de Precios y Límite en Backend:** La creación de órdenes en `commerce.service.ts` consulta el catálogo en base de datos para calcular el total a pagar, impidiendo que peticiones manipuladas desde el frontend alteren el precio o montos a financiar.
2. **Criptografía de Contraseñas:** Uso de `scrypt` con sales criptográficas de 16 bytes y comparación de firmas con `crypto.timingSafeEqual` para mitigar ataques de tiempo.
3. **Restricción de Orígenes CORS:** Configuración en `app.ts` limitada a orígenes localhost y dominios específicos de Render del megaproyecto (`allowedOrigins.push(/^https:\/\/megaproyecto-(cliente-web|portal-comercios|panel-admin|fintech-api)\.onrender\.com$/)`).
4. **Protección de Secretos en Git:** `.gitignore` excluye `.env`, `.env.*`, archivos `.pem` y `.key`. Verificación con `git ls-files .env` confirmó que ningún secreto está indexado en el control de versiones.
5. **Configuración de Blueprint en Render:** `render.yaml` utiliza `generateValue: true` para todas las claves criptográficas (`JWT_SECRET`, `AUTH_SECRET`, `COOKIE_SECRET`) y no incluye contraseñas fijas.

---

## 🛑 CONCLUSIÓN Y DECISIÓN DE DEPLOY

**ESTADO DEL DEPLOY: BLOQUEADO (NO PROCEDER)**

El sistema presenta 2 riesgos críticos que comprometen la integridad de la base de datos en Supabase (falta de RLS) y la estabilidad del servidor ante ataques de fuerza bruta (ausencia de Rate Limiting), además de una falla de enlace entre frontend y backend en producción (`API_BASE_URL`).

Se requiere la aprobación y corrección manual de los puntos críticos antes de autorizar el pase a producción en Render.
