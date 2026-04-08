# Peto Tecno — Sistema POS & ERP

> Sistema de Punto de Venta, Inventario y Apartados (`Petoahorro`) diseñado para tiendas de tecnología en El Salvador.
> Construido con Next.js 16, Prisma 7, PostgreSQL y Tailwind CSS 4.

---

## 📋 Tecnologías

| Capa | Tecnología | Versión |
|---|---|---|
| Framework principal | Next.js (App Router) | 16.2.2 |
| Lenguaje | TypeScript | ^5 |
| ORM | Prisma + Adapter PG | ^7.7.0 |
| Base de datos | PostgreSQL | ≥14 |
| Estilos | Tailwind CSS | ^4 |
| Íconos | Lucide React | ^1.7.0 |
| Gráficos | Recharts | ^3.8.1 |
| Autenticación | JWT via `jose` | ^6.2.2 |
| Hashing contraseñas | `bcrypt` | ^6.0.0 |
| Escaneo QR/Barcode | `html5-qrcode` | ^2.3.8 |

---

## ⚙️ Instalación desde cero

### Prerrequisitos

- **Node.js** ≥ 20
- **PostgreSQL** ≥ 14 corriendo localmente (o en la red)
- **Git**

### 1. Clonar el repositorio

```bash
git clone https://github.com/AdonayFlores/PetoTecnoWebApp.git
cd PetoTecnoWebApp
```

### 2. Instalar dependencias

```bash
npm install
```

### 3. Configurar variables de entorno

Crea el archivo `.env` en la raíz del proyecto (nunca lo subas a Git):

```env
# Cadena de conexión a PostgreSQL
DATABASE_URL="postgresql://TU_USUARIO:TU_CONTRASEÑA@localhost:5432/petotecno?schema=public"

# Clave secreta para firmar los JWT — usa una cadena aleatoria larga
JWT_SECRET="reemplaza-esto-con-una-clave-muy-larga-y-aleatoria"
```

> ⚠️ **Importante:** Si `JWT_SECRET` no está definido, la aplicación lanzará un error al arrancar. Nunca uses claves predecibles en producción.

### 4. Crear la base de datos

Conéctate a PostgreSQL y crea la base de datos (si no existe):

```sql
CREATE DATABASE petotecno;
```

### 5. Sincronizar el esquema de Prisma

```bash
# Aplica el schema.prisma a la base de datos (crea tablas e índices)
npx prisma db push
```

### 6. Poblar datos iniciales (Seed)

```bash
# Carga usuarios de prueba (Admin: admin@petotecno.com / Cajero: cajero@petotecno.com)
# Las contraseñas están en la ruta GET /api/seed
curl http://localhost:3000/api/seed
```

> O visita `http://localhost:3000/api/seed` desde el navegador tras iniciar el servidor.

### 7. Iniciar el servidor de desarrollo

```bash
npm run dev
```

La aplicación estará disponible en `https://localhost:3000`.

> **Nota:** el script `dev` usa `--experimental-https`, por lo que el navegador puede mostrar una advertencia de seguridad en desarrollo. Acéptala para continuar.

---

## 📁 Estructura de carpetas

```
PetoTecnoAppWeb/
│
├── prisma/
│   └── schema.prisma          # Modelos de BD: User, Product, Sale, Petoahorro, TurnoCaja
│
├── src/
│   ├── app/
│   │   ├── page.tsx           # 🏠 Dashboard principal
│   │   ├── layout.tsx         # Layout raíz con Sidebar
│   │   ├── login/             # Página de autenticación
│   │   ├── pos/               # 🛒 Módulo Punto de Venta
│   │   ├── inventario/        # 📦 Gestión de inventario
│   │   ├── petoahorro/        # 💰 Módulo de apartados
│   │   ├── caja/              # 🏦 Corte de caja y arqueo
│   │   ├── transacciones/     # 📊 Historial de ventas
│   │   └── api/               # API Routes (backend)
│   │       ├── auth/          # login, logout, me
│   │       ├── products/      # CRUD de productos
│   │       ├── sales/         # Crear y consultar ventas
│   │       ├── petoahorro/    # CRUD de cuentas y abonos
│   │       └── caja/          # Apertura, cierre, historial
│   │
│   ├── components/
│   │   └── layout/
│   │       └── Sidebar.tsx    # Navegación principal + Bottom Nav móvil
│   │
│   └── lib/
│       ├── auth.ts            # signJwt / verifyJwt
│       ├── api-auth.ts        # requireAuth() — protección de API Routes
│       ├── prisma.ts          # Singleton del cliente Prisma
│       └── timezone.ts        # Utilidades de zona horaria GMT-6 El Salvador
│
├── .env                       # Variables de entorno (NO subir a Git)
├── .gitignore
├── next.config.ts
└── package.json
```

---

## 🔑 Credenciales de prueba (Seed)

| Rol | Email | Contraseña |
|---|---|---|
| Administrador | `admin@petotecno.com` | `admin123` |
| Cajero | `cajero@petotecno.com` | `cajero123` |

---

## 🛠️ Comandos útiles

```bash
# Iniciar desarrollo
npm run dev

# Aplicar cambios al schema de Prisma (sin migración formal)
npx prisma db push

# Ver/editar la BD en interfaz web de Prisma
npx prisma studio

# Compilar para producción
npm run build

# Iniciar en modo producción
npm start

# Linter
npm run lint
```

---

## 🚢 Despliegue en producción

1. Asegúrate de que `JWT_SECRET` esté configurado en las variables de entorno del servidor.
2. Usa `npm run build && npm start` o despliega en **Vercel** / **Railway**.
3. Cambia `DATABASE_URL` para apuntar a la base de datos de producción (Neon, Supabase, Railway, etc.).
4. Ejecuta `npx prisma db push` una vez en el servidor de producción para sincronizar índices.
