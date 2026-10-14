# Guía de Arquitectura del Monorepo

## 1. Organización de Workspaces

El proyecto utiliza **npm workspaces** administrado desde el `package.json` raíz. Cada workspace tiene su propio ciclo de vida y puede depender de otros paquetes del monorepo utilizando la sintaxis de enlace local (`*`).

### Mapeo de Workspaces:
| Workspace | Paquete / Directorio | Responsabilidad | Puerto Local |
| :--- | :--- | :--- | :--- |
| **API Backend** | `@megaproyecto/api` (`services/api`) | Servidor Fastify modular con rutas agrupadas en plugins (`/health`, `/auth`, etc.) | `4000` |
| **Cliente Web** | `@megaproyecto/cliente-web` (`apps/cliente-web`) | Portal web responsivo para clientes: compras, simulación de crédito y cuotas | `3000` |
| **Portal Comercios** | `@megaproyecto/portal-comercios` (`apps/portal-comercios`) | Gestión de sucursales, conciliación y catálogo de comercios | `3001` |
| **Panel Admin** | `@megaproyecto/panel-admin` (`apps/panel-admin`) | Backoffice de supervisión, KYC, auditoría y revisión manual de riesgo | `3002` |
| **Config** | `@megaproyecto/config` (`packages/config`) | Constantes de entorno, seguridad y configuración común | N/A |
| **Schemas** | `@megaproyecto/schemas` (`packages/schemas`) | Tipos TypeScript, esquemas de validación y utilidades monetarias exactas | N/A |
| **UI** | `@megaproyecto/ui` (`packages/ui`) | Tokens de diseño visual CSS y componentes transversales compartidos | N/A |

---

## 2. Flujo de Trabajo y Comandos

### Instalación de Dependencias
Ejecutar siempre en la raíz del monorepo (`megaproyecto-fintech`):
```bash
npm install
```
*npm resolverá todas las dependencias cruzadas entre paquetes y creará los symlinks en `node_modules` automáticamente.*

### Compilación Unificada
```bash
npm run build
```

### Ejecución de Pruebas Unitarias
```bash
npm test
```

### Ejecución en Modo Desarrollo
Para ejecutar cada servicio o aplicación individualmente:
- **Solo API Backend**: `npm run dev:api`
- **Solo Portal de Clientes**: `npm run dev:cliente`
- **Solo Portal de Comercios**: `npm run dev:comercios`
- **Solo Panel Administrativo**: `npm run dev:admin`

---

## 3. Principios de Seguridad y Contabilidad
1. **Representación de Dinero (`Money`)**:  
   Todo cálculo que involucre saldo, cuotas o pagos debe utilizar el tipo `Money` (`amountInCents: number`, `currency: string`) del paquete `@megaproyecto/schemas`. Nunca utilizar números decimales directos para valores monetarios.
2. **Avisos de Simulación**:  
   Todas las interfaces incluyen el banner accesible de advertencia `PLATFORM_DISCLAIMER`. Ningún usuario debe operar con la expectativa de que se trata de fondos reales.
3. **Segregación de Comercios**:  
   Las consultas y mutaciones de comercios deben filtrar siempre por `merchantId` validado en la sesión autenticada del servidor.
