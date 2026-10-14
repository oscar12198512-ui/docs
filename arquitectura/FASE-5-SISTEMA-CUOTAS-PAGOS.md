# FASE 5 — SISTEMA DE CUOTAS, PAGOS E HISTORIAL FINANCIERO

## 1. Resumen Ejecutivo y Alcance
En la **Fase 5** se construyó el módulo financiero central de la plataforma fintech para gestionar:
1. **Planes de pago** vinculados exclusivamente a solicitudes de crédito aprobadas.
2. **Calendarios de amortización de cuotas** generados con aritmética exacta de números enteros (centavos), garantizando **cero error de redondeo acumulado** mediante ajuste en la última cuota.
3. **Registro de pagos en modo demostración**, con control estricto de **idempotencia** por referencia declarada para impedir pagos duplicados.
4. **Integridad de saldos y transaccionalidad**:
   - Un pago en espera de confirmación (`PENDING_CONFIRMATION`) **no descuenta** el saldo ni marca la cuota como pagada.
   - La confirmación por parte del comercio receptor o analista autorizado imputa el pago y actualiza el saldo.
   - La reversión de un pago confirmado restaura el saldo del plan y el estado de la cuota correspondiente **sin eliminar** el registro del pago ni su historial de auditoría.
5. **Aislamiento multi-tenant**: Clientes únicamente acceden a sus planes y comprobantes (403 ante acceso cruzado), comercios solo a operaciones de sus tiendas, y personal autorizado a la consola global.
6. **Alertas de vencimiento**: Monitoreo de cuotas vencidas y próximas a vencer sin cobros automáticos ni penalizaciones no autorizadas.
7. **Compatibilidad total** con la autenticación (Fase 3), solicitudes de crédito (Fase 4) y diseño de interfaz (Fase 2).

---

## 2. Modelos de Dominio y Esquemas (`packages/schemas/src/payments.ts`)

| Modelo / DTO | Descripción |
| :--- | :--- |
| `PaymentPlan` | Representa el crédito concedido, monto principal, moneda, tasa de interés, método de cálculo (`FIXED_EQUAL_INSTALLMENTS`), frecuencia (`QUINCENAL` / `MENSUAL`), saldo pendiente y estado (`ACTIVE`, `COMPLETED`, `DEFAULTED`, `CANCELLED`). |
| `PaymentScheduleInstallment` | Cuota individual con nro, fecha de vencimiento, importe exigible, importe pagado, saldo pendiente, estado (`PENDING`, `PARTIALLY_PAID`, `PAID`, `OVERDUE`, `CANCELLED`) y referencias asociadas. |
| `PaymentRecord` | Registro auditable del abono: referencia única interna (`PAG-VE-XXXXX`), referencia declarada/voucher del cliente, método de pago, banco emisor, estado (`PENDING_CONFIRMATION`, `CONFIRMED`, `REVERSED`, `REJECTED`, `FAILED`), actor y notas. |
| `PaymentAuditEvent` | Trazabilidad inmutable de eventos (`CREATED`, `CONFIRMED`, `REVERSED`, `REJECTED`) con ID del usuario, rol, fecha/hora y justificación. |
| `CreatePaymentPlanDTO` | Parámetros para originar el plan desde una solicitud aprobada. |
| `RegisterPaymentDTO` | Parámetros para declarar un abono con voucher de pago. |
| `ConfirmPaymentDTO` | Dictamen de conciliación por parte del comercio o staff (`CONFIRM` o `REJECT`). |
| `ReversePaymentDTO` | Reversión financiera con motivo formal auditado. |

---

## 3. Base de Datos y Migración (`database/migrations/004_payment_plans_and_installments.sql`)

La migración crea las siguientes tablas con restricciones de integridad referencial:
- `payment_plans`: Tabla principal con llaves foráneas hacia `users` (cliente) y `credit_applications`.
- `plan_installments`: Cuotas con restricción única compuesta `(plan_id, installment_number)`.
- `payment_records`: Movimientos financieros con restricción condicional única para idempotencia:
  ```sql
  CREATE UNIQUE INDEX idx_payment_idempotency 
  ON payment_records(plan_id, declared_reference) 
  WHERE status != 'REJECTED' AND status != 'FAILED';
  ```
- `payment_audit_events`: Bitácora inmutable de eventos financieros.

---

## 4. Algoritmo de Generación de Cuotas con Cero Desfase (`schedule-generator.service.ts`)

Para evitar cualquier error de redondeo acumulado en monedas con fracciones (USD / VES en centavos):
1. Se calcula la cuota base entera: $\text{base} = \lfloor \text{montoTotal} / N \rfloor$.
2. Las primeras $N - 1$ cuotas reciben el valor $\text{base}$.
3. La última cuota ($N$) recibe la diferencia exacta: $\text{total} - \sum_{i=1}^{N-1} \text{cuota}_i$.
4. **Garantía matemática**: La suma de todas las cuotas es idéntica al monto total financiado en el 100% de los casos (incluso en divisiones impares como \$100.00 entre 3 cuotas: \$33.33, \$33.33, \$33.34 = exactamente \$100.00).

---

## 5. Reglas de Negocio Implementadas

### 5.1. Solicitud Aprobada Obligatoria
- Antes de crear cualquier plan de pago, `PaymentsService` consulta la solicitud de crédito mediante `CreditService.getApplicationById`.
- Si `application.status !== 'APPROVED'`, el sistema rechaza la operación de inmediato con código HTTP `400` y código de error `'NO_APPROVED_CREDIT'`.
- Si el monto principal financiado supera el límite aprobado (`approvedLimitInCents`), la operación es rechazada con HTTP `400`.

### 5.2. Mecanismo de Idempotencia y Prevención de Pagos Duplicados
- Todo pago requiere un identificador o voucher declarado (`declaredReference`).
- El servicio consulta `findPaymentByDeclaredReference(planId, declaredReference)`.
- Si ya existe un registro activo o confirmado con esa misma referencia para ese plan, el backend responde con código HTTP `409 Conflict` (`PAGO_DUPLICADO`), impidiendo dobles acreditaciones.

### 5.3. Integridad de Saldos (Pendientes vs. Confirmados)
- **Registro de Pago Pendiente**: Entra en estado `PENDING_CONFIRMATION`. **No** altera el `remainingBalanceInCents` del plan ni modifica el estado de la cuota a pagada.
- **Confirmación por Personal o Comercio**: Transiciona a `CONFIRMED`. Descuenta el saldo pendiente del plan y abona a la cuota objetivo (o en cascada a las cuotas vencidas/pendientes más antiguas). Si todas las cuotas quedan saldadas, el plan transiciona a `COMPLETED`.
- **Reversión**: Solo permitida para perfiles con rol de supervisión (`ADMIN`, `RISK_ANALYST`, `ADMIN_SUPERVISOR`). Restaura el saldo del plan y los montos pendientes de las cuotas afectadas, cambia el estado del pago a `REVERSED` y guarda un evento de auditoría con el motivo y el usuario responsable, sin borrar el registro contable.

### 5.4. Aislamiento Multi-Tenant
- **Cliente**: Solo puede consultar y registrar pagos en planes donde `plan.clientId === user.id`. Si intenta consultar o pagar planes de otros clientes, el backend responde con HTTP `403 Forbidden`.
- **Comercio Afiliado**: Solo puede consultar y confirmar pagos de planes asociados a su tienda (`plan.merchantId === user.merchantId`). Intentos de acceso cruzado reciben HTTP `403 Forbidden`.
- **Staff / Admin**: Tiene visibilidad global y capacidad de dictamen y reversión de acuerdo a su rol.

---

## 6. Endpoints de la API REST (`services/api`)

| Método | Endpoint | Roles Permitidos | Descripción |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/payments/plans` | Cliente / Staff | Crea un plan de pagos para una solicitud aprobada. |
| `GET` | `/api/v1/payments/plans` | Autenticado | Lista los planes según el rol (aislamiento estricto). |
| `GET` | `/api/v1/payments/plans/:id` | Propietario / Staff | Detalle del plan y su calendario de cuotas. |
| `POST` | `/api/v1/payments` | Cliente / Comercio / Staff | Registra un pago demostrativo con idempotencia. |
| `GET` | `/api/v1/payments` | Autenticado | Historial de pagos filtrado por pertenencia. |
| `GET` | `/api/v1/payments/:id` | Propietario / Staff | Detalle de un pago y su bitácora de auditoría. |
| `POST` | `/api/v1/payments/:id/confirm` | Comercio / Staff | Confirma o rechaza un pago pendiente (cliente bloqueado con 403). |
| `POST` | `/api/v1/payments/:id/reverse` | Admin / Riesgo | Reversa un pago confirmado y restaura saldos. |
| `GET` | `/api/v1/payments/alerts/due` | Autenticado | Monitoreo de cuotas vencidas y próximas a vencer. |

---

## 7. Pruebas Automatizadas y Resultados

Se implementaron 15 pruebas unitarias y de integración en `services/api/src/modules/payments/payments.test.ts` que cubren:
1. Generación de cuotas con cero error de redondeo (división impar \$100 / 3 cuotas).
2. Cálculo de fechas límite para frecuencias quincenales y mensuales.
3. Rechazo de plan si la solicitud no está aprobada (código 400).
4. Rechazo si el monto solicitado supera el límite aprobado (código 400).
5. Creación exitosa de plan para solicitud aprobada.
6. Rechazo de pago duplicado por misma referencia declarada (código 409).
7. Saldo intacto ante pagos en estado `PENDING_CONFIRMATION`.
8. Bloqueo al cliente cuando intenta confirmar su propio pago (código 403).
9. Confirmación de pago por parte del comercio y reducción del saldo pendiente.
10. Bloqueo de reversión a usuarios sin rol staff (código 403).
11. Reversión por analista de riesgo, restauración de saldos y preservación del historial.
12. Bloqueo de cliente al consultar planes de otro cliente (código 403).
13. Bloqueo de cliente al consultar pagos de otro cliente (código 403).
14. Consulta de planes propios con filtro automático.
15. Monitoreo de alertas de vencimiento.

**Resultado de ejecución global**:
```text
✓ @megaproyecto/cliente-web: 1 passed
✓ @megaproyecto/panel-admin: 1 passed
✓ @megaproyecto/portal-comercios: 1 passed
✓ @megaproyecto/api: 35 passed (payments: 15, auth: 12, credit: 7, health: 1)
Total: 38/38 pruebas aprobadas (100% exitosas)
Compilación TypeScript y Vite: Exit Code 0 en todos los workspaces.
```
