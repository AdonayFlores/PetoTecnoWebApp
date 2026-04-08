# Peto Tecno — Arquitectura del Sistema

> Documento técnico-funcional del ERP/POS de Peto Tecno.
> Audiencia: desarrolladores, administradores del sistema y el dueño del negocio.

---

## 1. Roles y Permisos

El sistema reconoce dos roles definidos en el campo `rol` del modelo `User`.

### 🔑 Administrador (`admin`)

Es el dueño o encargado principal de la tienda. Tiene acceso completo al sistema.

| Módulo | Permisos |
|---|---|
| Dashboard | Ve todos los KPIs: ingresos totales, abonos, tendencias de 7 días |
| Inventario | Crear, editar y eliminar productos |
| POS | Registrar ventas |
| Petoahorro | Ver y gestionar todas las cuentas |
| Caja | Abrir/cerrar caja, ver historial completo de arqueos, aprobar turnos |
| Transacciones | Ver ventas de todos los cajeros |

### 🧾 Cajero (`cajero`)

Operador de caja. Solo tiene acceso a lo necesario para atender al cliente.

| Módulo | Permisos |
|---|---|
| Dashboard | Ve el estado de la caja (abierta/cerrada) y sus propias ventas |
| Inventario | Solo lectura — puede consultar stock y precios |
| POS | Registrar ventas |
| Petoahorro | Registrar abonos y nuevas cuentas |
| Caja | Abrir y cerrar su propio turno |
| Transacciones | Solo sus propias ventas del turno activo |

### Implementación de permisos

**Nivel de página:** El archivo `src/middleware.ts` protege las rutas de navegación. Si un cajero intenta acceder a `/caja/historial`, el middleware lo redirige al inicio.

**Nivel de API:** El helper `requireAuth()` en `src/lib/api-auth.ts` protege cada endpoint. Un cajero con token válido que llame a `GET /api/caja/historial` directamente recibirá un `403 Forbidden`.

```
Flujo de autenticación:
1. Usuario ingresa email + contraseña en /login
2. El servidor valida contra bcrypt y emite un JWT (8 horas de validez)
3. El JWT se almacena en una cookie HttpOnly llamada auth_token
4. Cada request incluye automáticamente la cookie
5. El middleware y los helpers validan el JWT en cada solicitud
```

---

## 2. Módulo de Caja (Arqueo de Turno)

### Concepto de Turno

Un **turno** (`TurnoCaja`) representa el período de operación de caja entre una apertura y un cierre. Solo puede existir **un turno abierto a la vez** en todo el sistema.

> **Regla crítica:** No se puede registrar ninguna venta si no hay un turno abierto. El POS verifica el estado de la caja antes de permitir cobros.

### Ciclo de vida de un turno

```
[APERTURA]
    ↓
  ABIERTA ──────────────────────────────────────────────────┐
    ↓  (Durante el turno)                                   │
  Se acumulan automáticamente:                              │
  • efectivoVentas: suma de ventas en Efectivo              │
  • efectivoAbonos: suma de abonos de Petoahorro            │
  • totalGastos: suma de gastos registrados manualmente     │
    ↓                                                       │
[CIERRE — el cajero cuenta el efectivo físico]              │
    ↓                                                       │
  El servidor calcula:                                      │
  diferencia = efectivoContado − (                          │
    fondoInicial + efectivoVentas + efectivoAbonos          │
    − totalGastos                                           │
  )                                                         │
    ↓                                                       │
  |diferencia| < $0.01 → CERRADA_CUADRA ✅                  │
  diferencia < 0       → FALTANTE ⚠️                         │
  diferencia > 0       → SOBRANTE ℹ️                         │
    ↓                                                       │
[REVISIÓN DEL ADMIN — aprueba el arqueo en /caja/historial] │
    ↓                                                       │
  aprobado = true, revisadoPor = nombre del admin ──────────┘
```

### Principio de "Arqueo Ciego"

El cajero **no puede ver el total esperado** antes de ingresar el efectivo contado. Esta restricción intencional evita que el cajero ajuste su conteo para forzar un cuadre. El sistema muestra únicamente el mensaje "Cuente el efectivo físico en gaveta".

### Seguridad del cierre

El campo `estado` del turno y la `diferencia` **siempre son calculados por el servidor**, nunca por el cliente. Esto impide que alguien manipule la petición HTTP para declarar un cuadre falso.

---

## 3. Módulo Petoahorro (Sistema de Apartados)

### Concepto

Petoahorro es el sistema de ventas a cuotas de Peto Tecno. Un cliente puede apartar un producto pagando una prima inicial y luego realizando abonos semanales hasta liquidar el total.

### Flujo completo

```
1. CREACIÓN DE LA CUENTA
   ┌─────────────────────────────────────┐
   │ Cajero registra:                    │
   │  • Nombre y DUI del cliente         │
   │  • Producto elegido                 │
   │  • Prima inicial (puede ser $0)     │
   │  • Número de cuotas                 │
   └─────────────────┬───────────────────┘
                     ↓
   • product.reservado += quantity   (el producto queda apartado)
   • petoahorro.status = "ACTIVE"
   • Se calcula: cuotaSugerida = (totalAmount - prima) / numCuotas
   • Si prima >= totalAmount → status = "LIQUIDATED" directamente

2. ABONOS (puede repetirse N veces)
   ┌─────────────────────────────────────┐
   │ Cajero registra un abono:           │
   │  • Monto del abono                  │
   └─────────────────┬───────────────────┘
                     ↓
   • paidAmount = MIN(paidAmount + abono, totalAmount)
     (no puede sobrepasarse del total)
   • Si paidAmount >= totalAmount → status = "LIQUIDATED"
   • Se registra en PaymentHistory

3. ENTREGA DEL PRODUCTO
   ┌─────────────────────────────────────┐
   │ Solo si status = "LIQUIDATED"       │
   │ Cajero ejecuta la entrega           │
   └─────────────────┬───────────────────┘
                     ↓
   • petoahorro.status → "DELIVERED"      ← updateMany condicional
   • product.stock    -= quantity          (sale del inventario)
   • product.reservado -= quantity         (se libera la reserva)
   • Se genera una venta (Sale) con paymentMethod = "Petoahorro (Cuotas)"
   • Se emite el DTE correspondiente
```

### Estados del Petoahorro

| Estado | Descripción |
|---|---|
| `ACTIVE` | Cuenta abierta con pagos pendientes |
| `LIQUIDATED` | Pagado en su totalidad, pendiente de entrega física |
| `DELIVERED` | Producto entregado al cliente y DTE emitido |

### Reglas de negocio importantes

1. **No se puede abonar más del saldo pendiente.** El sistema usa `Math.min()` para limitar el paidAmount.
2. **No se puede eliminar un producto con unidades en Petoahorro.** El sistema verifica `product.reservado > 0` antes de eliminar.
3. **Solo se entrega si está LIQUIDADO.** El endpoint `deliver` usa `updateMany` con `where: { status: "LIQUIDATED" }` para prevenir doble entrega por doble clic.
4. **El stock físico no disminuye al crear el apartado.** Solo disminuye al momento de la entrega. Ver columna `stock` vs `reservado` en `Product`.

### Diferencia entre `stock` y `reservado` en `Product`

```
stock     = unidades físicas totales en bodega
reservado = unidades comprometidas en Petoahorros activos
libre     = stock - reservado   ← lo que realmente se puede vender
```

El POS siempre verifica `libre` antes de permitir una venta, no `stock` directamente.

---

## 4. Módulo POS — Reglas de Facturación

### Ley Salvadoreña: Umbral de $25

Toda venta mayor a **$25.00** requiere obligatoriamente:
- Nombre completo del cliente
- Número de DUI (`XXXXXXXX-X`)

Esta validación ocurre tanto en el frontend (UX) como en el backend (`/api/sales`).

### Cálculo fiscal (13% IVA)

```
totalAPagar  = precio_db × cantidad   (precio siempre desde BD, nunca del cliente)
subtotalNeto = totalAPagar / 1.13     (base imponible)
iva          = totalAPagar - subtotalNeto
```

### Seguridad de precios

Los precios **nunca** provienen del cuerpo del request. El endpoint `POST /api/sales` hace un `findMany` de los productos por ID a la base de datos y usa esos precios para calcular el total. Esto previene manipulación de precios desde el navegador o herramientas externas.

---

## 5. Decisiones Técnicas de Arquitectura

### 5.1 Huso Horario GMT-6 (El Salvador)

El servidor corre en UTC. PostgreSQL almacena todas las fechas en UTC. El Salvador opera en UTC-6, lo que significa que su medianoche local equivale a **06:00 AM UTC** del mismo día.

**Problema que resuelve:** sin este ajuste, el reporte del "día de hoy" incluiría transacciones del día anterior (desde las 6 PM salvadoreña en adelante) y excluiría las de las últimas horas del día actual.

**Implementación:** todas las comparaciones de fecha pasan por `src/lib/timezone.ts`:

```typescript
// Medianoche del 8 de Abril en El Salvador = 06:00 UTC
getSvDayRange(2026, 4, 8)
// → { gte: 2026-04-08T06:00:00Z, lt: 2026-04-09T06:00:00Z }
```

### 5.2 Autenticación con JWT + Cookie HttpOnly

Se usa `jose` (implementación Web Crypto API) en vez de la popular `jsonwebtoken` porque Next.js Edge Runtime no soporta módulos de Node.js nativos.

Las sesiones duran **8 horas** y se almacenan en cookies `HttpOnly`, lo que las hace inaccesibles desde JavaScript del navegador (protección XSS).

### 5.3 Singleton de Prisma en Desarrollo

```typescript
// src/lib/prisma.ts
export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

Sin este patrón, el Hot Module Replacement de Next.js generaría una nueva conexión a la BD en cada guardado de archivo, eventualmente agotando el pool de conexiones de PostgreSQL.

### 5.4 Transacciones Prisma para Atomicidad Financiera

Toda operación que modifica múltiples tablas (crear venta, registrar abono, entregar producto) ocurre dentro de un `prisma.$transaction()`. Si cualquier paso falla, todos los cambios se revierten automáticamente, garantizando la consistencia financiera.

```
Ejemplo — Crear una venta:
  BEGIN TRANSACTION
    1. findUnique de cada producto → verificar stock
    2. sale.create → registrar la venta y sus items
    3. product.update × N → decrementar stock
  COMMIT (o ROLLBACK si hubo error)
```

### 5.5 Índices de Base de Datos

Se han declarado `@@index` en los modelos con mayor volumen de consultas para garantizar rendimiento con 20,000+ registros:

| Modelo | Índices |
|---|---|
| `Sale` | `createdAt`, `cashierId`, `(cashierId, createdAt)`, `status` |
| `Petoahorro` | `status`, `cashierId`, `clientDui` |
| `PaymentHistory` | `createdAt`, `petoahorroId`, `cashierId` |
| `TurnoCaja` | `estado`, `fechaApertura` |
| `Product` | `category`, `stock` |

---

## 6. Flujo de datos — Resumen visual

```
CLIENTE (Navegador)
        │
        │  HTTPS + Cookie JWT
        ▼
┌─────────────────────────────────────────┐
│          NEXT.JS APP ROUTER             │
│                                         │
│  middleware.ts  ←  Protege páginas      │
│  api-auth.ts    ←  Protege API Routes   │
│                                         │
│  /app/pos/page.tsx   (React Client)     │
│  /app/inventario/    (React Client)     │
│  /app/caja/          (React Client)     │
│                                         │
│  /api/sales/         (Route Handler)    │
│  /api/petoahorro/    (Route Handler)    │
│  /api/caja/          (Route Handler)    │
└────────────────┬────────────────────────┘
                 │  Prisma Client (TCP)
                 ▼
┌─────────────────────────────────────────┐
│         POSTGRESQL DATABASE             │
│                                         │
│  User  ←──── Sale ←──── SaleItem        │
│    │                        │           │
│    └──── Petoahorro ────────┘           │
│               │                         │
│          PaymentHistory                 │
│                                         │
│  TurnoCaja ←─── GastoCaja              │
│                                         │
│  Product ───── SaleItem                 │
│     └───────── Petoahorro               │
└─────────────────────────────────────────┘
```

---

*Documento generado el 8 de Abril de 2026. Mantener actualizado con cada cambio arquitectónico relevante.*
