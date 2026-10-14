# Sistema de 7 Niveles de Crédito Progresivos y Financiamiento Vehicular

**Estado:** Requisito Funcional y Arquitectura Registrada (Previo a Fase 4)  
**Módulos Responsables:** `@megaproyecto/config`, `@megaproyecto/schemas`  
**Entorno:** Fintech Platform  

---

## 1. Visión General del Sistema Progresivo

La plataforma contempla un modelo de crédito en **siete niveles progresivos** diseñado para acompañar el crecimiento y comportamiento de pago de los usuarios. 

Para evitar la dispersión de reglas y garantizar el principio de **fuente única de verdad (Single Source of Truth)**, los criterios, categorías, límites máximos de plazos y banderas de verificación están centralizados en `@megaproyecto/config` (`CREDIT_TIERS_CONFIG`) y tipados en `@megaproyecto/schemas` (`CreditTierLevel`, `VehicleVerificationRequirements`).

---

## 2. Definición Centralizada de los 7 Niveles

| Nivel | Denominación | Categoría Financiable | Plazo Máximo Deseado | Tipo de Bien |
| :---: | :--- | :--- | :---: | :--- |
| **1** | Inicial / Microcrédito | Bienes de consumo de primera necesidad | 4 cuotas (quincenales) | Consumo masivo / Farmacia / Alimentos |
| **2** | Consolidación | Calzado, vestimenta y retail general | 6 cuotas (quincenales) | Comercio afiliado |
| **3** | Electrodomésticos y Hogar | Bienes duraderos y línea blanca | 8 cuotas (quincenales) | Hogar y oficina |
| **4** | Tecnología y Equipos | Telefonía celular, computación y herramientas | 12 cuotas | Tecnología y productividad |
| **5** | Línea Premium / Comercial | Equipamiento comercial y compras mayores | 16 cuotas | Emprendimiento y gama alta |
| **6** | **Movilidad Ligera** | **Motocicletas** | **Hasta 24 cuotas** | Vehículo motorizado de 2 ruedas |
| **7** | **Movilidad Integral** | **Automóviles** | **Hasta 48 cuotas** | Vehículo automotor particular/comercial |

---

## 3. Requisitos Especiales: Niveles 6 y 7 (Vehículos)

### 3.1. Naturaleza de los Plazos Máximos
> [!IMPORTANT]
> Los plazos máximos indicados (**hasta 24 cuotas para motocicletas** y **hasta 48 cuotas para automóviles**) constituyen **límites máximos deseados del producto**, y bajo ninguna circunstancia representan una aprobación automática.

La aprobación final de cualquier solicitud para los Niveles 6 y 7 está condicionada a:
1. **Políticas de riesgo crediticio** y score de comportamiento del cliente.
2. **Capacidad de pago demostrada**, evaluando la relación cuota mensual vs. ingreso neto disponible (Debt-to-Income / DTI).
3. **Condiciones del contrato** de financiamiento y porcentaje de cuota inicial obligatoria aportada por el solicitante.
4. **Disponibilidad del vehículo** en la red de concesionarios y ensambladoras aliadas.
5. **Requisitos legales, contractuales y de garantías aplicables en Venezuela**.

---

## 4. Verificaciones Diferenciadas para Motocicletas y Automóviles

Para los Niveles 6 y 7, la plataforma contempla una arquitectura de validación documental y de riesgo diferenciada:

```
                  ┌────────────────────────────────────────┐
                  │    SOLICITUD FINANCIAMIENTO VEHÍCULO    │
                  └───────────────────┬────────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
              ▼                                               ▼
   ┌──────────────────────┐                       ┌──────────────────────┐
   │ CONTROL 1: EVALUACIÓN│                       │ CONTROL 2: REQUISITOS│
   │  CREDITICIA Y RIESGO │                       │  LEGALES Y VEHÍCULO  │
   ├──────────────────────┤                       ├──────────────────────┤
   │ - Capacidad de Pago  │                       │ - Edad legal (18+)   │
   │ - Relación Cuota/Ingr│                       │ - Licencia INTT      │
   │ - Historial en Niv.1-5                       │   (2da Motos / 3ra)  │
   │ - Score de Riesgo    │                       │ - Doc. y Seriales    │
   │ - Cuota Inicial Pago │                       │ - RCV + Seguro Casco │
   │ - Reserva de Dominio │                       │ - Cert. de Origen    │
   └──────────┬───────────┘                       └──────────┬───────────┘
              │                                              │
              └───────────────────────┬──────────────────────┘
                                      │
                                      ▼
                      ┌───────────────────────────────┐
                      │ DECISIÓN FINAL INDEPENDIENTE  │
                      │  (Ambos Controles Aprobados)  │
                      └───────────────────────────────┘
```

### 4.1. Separación Estricta de Controles (Licencia vs. Crédito)
> [!CAUTION]
> **Principio de No Garantía**: La posesión de una licencia de conducir válida es un requisito legal y operativo para la conducción y entrega del vehículo, pero **es un control completamente independiente de la evaluación crediticia**.
> 
> La plataforma debe impedir expresamente que la validación de una licencia válida derive por sí sola en la aprobación del crédito o fuerce una decisión favorable.

### 4.2. Matriz de Verificaciones Diferenciadas (Venezuela)

| Tipo de Control | Nivel 6: Motocicletas (Hasta 24 cuotas) | Nivel 7: Automóviles (Hasta 48 cuotas) |
| :--- | :--- | :--- |
| **Identidad y Mayoría de Edad** | Cédula laminada vigente. Mayor de 18 años según Código Civil. | Cédula laminada vigente. Mayor de 18 años según Código Civil. |
| **Capacidad de Pago** | Comprobante de ingresos demostrables, actividad económica o certificación de ingresos visada. | Declaración de ISLR, estados de cuenta bancarios (últimos 3-6 meses) y certificación de ingresos visada por contador público colegiado. |
| **Licencia de Conducir (INTT)** | Licencia de **2do Grado** para motocicletas (o constancia de tramitación legal ante el INTT si aplica). | Licencia de **3er Grado** para automóviles particulares o de grado superior para transporte comercial. |
| **Documentación del Vehículo** | Certificado de Origen (unidades 0 km), factura comercial del concesionario o título con revisión de seriales de chasis y motor por el INTT/CPBEZ/PNB. | Certificado de Origen, factura con IVA detallado, título de propiedad INTT y revisión técnica de seriales de carrocería y motor. |
| **Seguros de Garantía** | Póliza de Responsabilidad Civil de Vehículos (**RCV** obligatoria según Ley de Transporte Terrestre) y cobertura contra pérdida total / robo. | RCV obligatoria y póliza de seguro de cobertura amplia (**Casco**) con endoso a favor de la entidad acreedora durante la vigencia del financiamiento. |
| **Instrumento de Garantía** | Venta con Reserva de Dominio conforme a la Ley sobre Ventas con Reserva de Dominio o prenda mercantil. | Venta con Reserva de Dominio inscrita ante el Registro Público / Notaría e INTT hasta la liquidación total de la cuota 48. |

*Nota regulatoria:* No se presume obligatoriedad de requisitos extraordinarios sin previa validación contra las resoluciones vigentes de SUDEBAN, el INTT y la legislación comercial de la República Bolivariana de Venezuela.

---

## 5. Decisiones Arquitectónicas para las Fases Posteriores

Para no incurrir en desarrollo prematuro ni alterar el alcance previo a la Fase 4:

### Funcionalidades Registradas y Preparadas:
- Constantes centralizadas en `packages/config` (`CREDIT_TIERS_CONFIG`).
- Interfaces de dominio en `packages/schemas` (`CreditTierLevel`, `VehicleVerificationRequirements`).
- Campo `targetTier` en `CreditApplication` y `currentTier` en `CreditProfile`.

### Funcionalidades Dejadas Estrictamente para Fases Posteriores:
1. **Lógica de Scoring y Algoritmo de Subida de Nivel**: Implementación en backend de los algoritmos que evalúan la puntualidad de pagos en los Niveles 1 al 5 antes de desbloquear los Niveles 6 y 7.
2. **Formularios Especializados de Carga Documental de Vehículos**: Subida de fotos de licencias INTT, certificados de origen y pólizas de seguro en frontend.
3. **Generador de Tablas de Amortización Extendida (24 y 48 cuotas)**: Algoritmo de cuota fija con desglose de amortización a capital e intereses regulados.
4. **Validación de Integración con Concesionarios**: Módulo de inventario vehicular y reserva de chasis/VIN.
5. **Generador de Pagarés y Contratos de Reserva de Dominio**: Plantillas formales de contratos prendarios para protocolización.
