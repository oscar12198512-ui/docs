# FASE 4 — FUNDAMENTOS DEL SISTEMA DE CRÉDITO Y EVALUACIÓN INICIAL

## 1. Resumen Ejecutivo
En la Fase 4 se diseñaron e implementaron los fundamentos del módulo de crédito de la plataforma fintech:
* **Ciclo de vida y máquina de estados** de solicitudes de crédito con trazabilidad inmutable.
* **Aislamiento estricto de clientes**: Un cliente únicamente puede consultar y operar sobre sus propias solicitudes.
* **Control de acceso basado en roles (RBAC) en backend**: El personal de riesgo y administración puede examinar expedientes y emitir dictámenes fundamentados; los clientes no pueden dictaminar ni ver solicitudes ajenas.
* **Motor interno de evaluación inicial de riesgo (`DEMO_INTERNAL_RISK_ENGINE`)**: Cálculo transparente y auditable de la relación Deuda/Ingreso (DTI), verificación de formato de cédula, análisis de puntualidad previa demostrativa, detección de señales de alerta financiera y checklist de recaudos pendientes, sin inventar ni simular consultas a burós externos.
* **Migración SQL reversible para PostgreSQL 14+**: Estructura relacional normalizada para solicitudes, historial de estados y evaluaciones iniciales.
* **Integración visual profesional**: Pantallas interactivas con validaciones en tiempo real en la web del cliente y en el panel administrativo.

---

## 2. Arquitectura de Estados y Trazabilidad

### Estados de una Solicitud de Crédito (`CreditApplicationStatus`)
1. `DRAFT` (Borrador): El cliente prepara su solicitud; puede editarla o eliminarla.
2. `SUBMITTED` (Enviada): Solicitud formalmente consignada; activa el motor de pre-evaluación interna de riesgo.
3. `UNDER_REVIEW` (En Revisión): Asignada a un analista de riesgo para verificación de recaudos y solvencia.
4. `REQUIRES_INFO` (Requiere Información): El analista de riesgo solicita aclaratorias o recaudos adicionales con una nota obligatoria. El cliente puede subsanar y reenviar.
5. `APPROVED` (Aprobada): Dictamen favorable emitido con límite y nivel crediticio asignados.
6. `REJECTED` (Rechazada): Dictamen desestimatorio con motivo explícito, objetivo y fundamentado.
7. `CANCELLED` (Cancelada): El solicitante o administración cancela una solicitud no resuelta.

### Registro Inmutable de Estados (`CreditStatusHistoryEntry`)
Cada transición de estado registra en una bitácora inmutable:
* Identificador único del evento.
* `applicationId`: Solicitud afectada.
* `previousStatus` y `newStatus`.
* `changedAt`: Marca de tiempo ISO-8601.
* `actorUserId`, `actorRole`, `actorName`: Identidad y rol del responsable de la decisión.
* `reason`: Justificación técnica o de negocio.
* `notes`: Observaciones adicionales o recaudos requeridos.

---

## 3. Motor de Evaluación Inicial de Riesgo (`evaluation.service.ts`)
* **Identificador de Motor:** `DEMO_INTERNAL_RISK_ENGINE` (transparente, no simula agencias de información crediticia externas).
* **Ratio Deuda / Ingreso (DTI):**
  $$\text{DTI} = \frac{\text{Obligaciones Mensuales Declaradas} + \text{Cuota Mensual Estimada}}{\text{Ingreso Mensual Declarado}} \times 100$$
  * Umbral prudencial de referencia: $\le 40\%$.
* **Historial Interno Demostrativo:**
  * Usuarios nuevos: puntaje base neutro de 650 puntos y nota explicativa de ausencia de operaciones previas.
  * Usuarios con cumplimiento: bonificación progresiva de scoring hasta 950 puntos.
  * Usuarios con morosidad: reducción de puntaje y bandera de alerta de riesgo.
* **Señales de Alerta Detectadas:**
  * `DTI_EXCEDE_UMBRAL_PRUDENCIAL_40`
  * `MONTO_SOLICITADO_SUPERA_CAPACIDAD_MENSUAL`
  * `OBLIGACIONES_DECLARADAS_CERO_SUJETO_VERIFICACION`
  * `FORMATO_CEDULA_REQUIERE_SUBSANACION`
  * `PRIMER_FINANCIAMIENTO_EN_PLATAFORMA`

---

## 4. Endpoints API Implementados

| Método | Ruta | Autorización | Propósito |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/credit/applications` | `CLIENT`, `ADMIN` | Crear solicitud (Borrador o Envío Inmediato con pre-evaluación). |
| `GET` | `/api/v1/credit/applications` | Autenticado | Listar solicitudes (Clientes ven solo las suyas; Riesgo/Admin ven todas). |
| `GET` | `/api/v1/credit/applications/my` | `CLIENT`, `ADMIN` | Listar solicitudes del cliente en sesión. |
| `GET` | `/api/v1/credit/applications/:id` | Autenticado | Detalle completo de solicitud con historial y evaluación (403 si cliente ajeno). |
| `POST` | `/api/v1/credit/applications/:id/submit` | Titular / `ADMIN` | Enviar formalmente borrador o subsanar requerimiento de información. |
| `POST` | `/api/v1/credit/applications/:id/cancel` | Titular / `ADMIN` | Cancelar solicitud antes de dictamen definitivo. |
| `POST` | `/api/v1/credit/applications/:id/review` | `RISK_ANALYST`, `ADMIN` | Emitir dictamen (`UNDER_REVIEW`, `REQUIRES_INFO`, `APPROVE`, `REJECT`). |

---

## 5. Seguridad y Protección de Datos
* **Aislamiento Multi-tenant por Usuario:** Si un cliente solicita una ID ajena, recibe `403 Forbidden` con mensaje `"Aislamiento de clientes: No tienes autorización para consultar solicitudes de otras personas."`
* **Redacción en Logs Técnicos:** El servidor Fastify redacta automáticamente `applicantCedula`, `monthlyIncomeInCents`, `monthlyObligationsInCents`, tokens y cookies.
* **Validación de Capacidad:** No se otorgan aprobaciones sin evaluar capacidad de pago real.
