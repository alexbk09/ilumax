# 📋 Plan Maestro: Expansión Multi-Rubro de ilumaX

> **Objetivo**: Convertir ilumaX en un sistema multi-rubro (comercio / restaurante / servicios) SIN dañar lo que ya funciona. Primero los cambios **transversales**, luego cada **módulo** por separado, y al final el **catálogo público** dinámico.

---

## 🧭 Fase 0: Reglas del juego (no romper nada)

1. **No direct calls**: todo pasa por `/services/`. Los componentes NUNCA llaman a `supabase.from()` directo.
2. **Componentización**: UI atómica (cards, modals, tabs) independientes de Supabase.
3. **Permisos por rol**: cada módulo nuevo se registra en `role_permissions` (can_read/write/delete). El menú admin se filtra según el rol.
4. **Migraciones idempotentes**: nueva migración por feature (`0028_*.sql`, `0029_*.sql`, ... `add column if not exists`).
5. **Testing económico**: calculator mjs + tests para precios, validaciones y flujo de auth.
6. **Configuraciones de módulo en `.env`**: 3 variables SI/NO que deciden qué secciones se activan (comercio, restaurante, servicios).
7. **Compatibilidad inversa**: los productos existentes `pricing_type='unit'` siguen funcionando idénticos.
8. **1 migración = 1 feature = 1 PR lógico** (trabajar sin doble esfuerzo).

---

## ♻️ FASE A — CAMBIOS TRANSVERSALES (base para todo)

> "Primero lo transversal para dejar listo lo que conlleva todo, después cada módulo."

### A1. Configuración de módulos activos (variables SI/NO en `.env`)

**Variable propuesta (3 flags)** en `.env.local` y `.env.development.local`:

```env
# ── Módulos del negocio (SI / NO) ──────────────────────────
# Controla qué secciones se muestran en el admin y en el catálogo público.
NEXT_PUBLIC_MODULE_COMERCIO='SI'      # Tienda normal: productos, carrito, WhatsApp
NEXT_PUBLIC_MODULE_RESTAURANTE='NO'   # Menú, combos, extras, pedidos de mesa
NEXT_PUBLIC_MODULE_SERVICIOS='NO'     # Citas, técnica, duración, cupos
```

**Archivo central** `lib/constants.ts`:
```ts
/** Módulos activos del sistema (leídos de NEXT_PUBLIC_MODULE_*). */
export const ACTIVE_MODULES = {
  comercio: process.env.NEXT_PUBLIC_MODULE_COMERCIO !== 'NO',
  restaurante: process.env.NEXT_PUBLIC_MODULE_RESTAURANTE === 'SI',
  servicios: process.env.NEXT_PUBLIC_MODULE_SERVICIOS === 'SI',
} as const
```

- **AdminShell**: el menú lateral oculta/se muestra cada grupo según `ACTIVE_MODULES`.
- **Catálogo público**: se ocultan los filtros, botones o pestañas de los módulos desactivados.
- **Server components**: la página admin a su vez valida en `lib/auth.ts` (requireModule).

---

### A2. Migración transversal `0028_multi_module_base.sql`

```sql
-- ============================================================
-- Migración 0028: Base multi-rubro
-- ============================================================

-- 1) Tipo de negocio del producto: product | service | combo
alter table public.products
  add column if not exists product_type text not null default 'product'
    check (product_type in ('product', 'service', 'combo'));

-- 2) Productos de servicio (duración, técnicos, agenda)
alter table public.products
  add column if not exists duration_minutes integer,
  add column if not exists sessions_count integer;

-- 3) Impuesto por producto (restaurante puede tener IVA distinto)
alter table public.products
  add column if not exists tax_rate numeric(5,2) not null default 0;

-- 4) Opciones personalizables (restaurante: tipo de masa, punto de carne)
create table if not exists public.product_options (
  id uuid primary key default gen_random_uuid(),
  product_id uuid not null references public.products(id) on delete cascade,
  name text not null,            -- ej: "Punto de carne", "Tipo de masa"
  required boolean not null default false,
  created_at timestamptz not null default now()
);

-- 5) Opciones (valores) de cada opción (ej: "Término medio", "Delgada")
create table if not exists public.product_option_values (
  id uuid primary key default gen_random_uuid(),
  option_id uuid not null references public.product_options(id) on delete cascade,
  label text not null,
  price_delta numeric(10,2) not null default 0,   -- precio adicional
  sort_order integer not null default 0
);

-- 6) Extras/adicionales con precio (ej: queso extra, doble carne)
create table if not exists public.product_extras (
  id uuid primary key default gen_random_uuid(),
  product_id uuid not null references public.products(id) on delete cascade,
  name text not null,
  price_usd numeric(10,2) not null default 0,
  created_at timestamptz not null default now()
);

-- 7) Componentes de comidas (combos: producto padre = combo, hijos = ítems)
create table if not exists public.product_components (
  id uuid primary key default gen_random_uuid(),
  combo_id uuid not null references public.products(id) on delete cascade,
  component_id uuid not null references public.products(id) on delete cascade,
  quantity integer not null default 1,
  created_at timestamptz not null default now(),
  unique (combo_id, component_id)
);

-- 8) Horarios de disponibilidad (menú por franja: desayunos, cenas)
create table if not exists public.product_schedule (
  id uuid primary key default gen_random_uuid(),
  product_id uuid not null references public.products(id) on delete cascade,
  day_of_week integer not null check (day_of_week between 0 and 6), -- 0=domingo, 6=sábado
  start_time time not null,
  end_time time not null,
  created_at timestamptz not null default now()
);

-- 9) Tipo de pedido en solicitudes (restaurante)
alter table public.solicitudes
  add column if not exists order_type text not null default 'online'
    check (order_type in ('online', 'dine_in', 'takeaway', 'delivery')),
  add column if not exists table_number text;

-- 10) Categorías jerárquicas (padre/hijo para menú y servicios)
alter table public.categories
  add column if not exists parent_id uuid references public.categories(id) on delete set null,
  add column if not exists icon text;

-- 11) Módulos nuevos en la tabla modules para permisos finos por rol
insert into public.modules (name, slug, description)
values
  ('Restaurante', 'restaurante', 'Gestión de menú, combos, extras y pedidos de restaurante'),
  ('Servicios',   'servicios',   'Gestión de citas, servicios, técnicos y agenda')
on conflict (slug) do update
  set name = excluded.name, description = excluded.description;

-- 12) Permisos admin para los módulos nuevos (solo rol admin)
insert into public.role_permissions (role_id, module_id, can_read, can_write, can_delete)
select r.id, m.id, true, true, true
from public.roles r
cross join public.modules m
where r.name = 'admin'
  and m.slug in ('restaurante', 'servicios')
on conflict (role_id, module_id)
do update set can_read = true, can_write = true, can_delete = true;
```

---

### A3. PostgREST `/services/*` — capa de datos

**Nuevos servicios:**

| Archivo | Función |
|---|---|
| `services/product-options.ts` | CRUD opciones, valores de opción y extras |
| `services/product-schedule.ts` | CRUD horarios/agenda por producto |
| `services/combos.ts` | CRUD componentes de combo |
| `services/bookings.ts` | Agenda de citas (servicios) |

**Modificación** `services/catalog.ts`:
- `mapProduct` agrega `product_type`, `duration_minutes`, `sessions_count`, `tax_rate`.
- `PRODUCT_SELECT` incluye las nuevas columnas.
- Cargar `product_options`, `product_extras`, `product_components` y `product_schedule` en el select (relaciones).

**Modificación** `services/requests.ts`:
- Tipo de pedido (`order_type`, `table_number`) en `createRequest`.
- Items con `product_options`/`product_extras` (JSONB snapshot en `solicitud_items`).

---

### A4. APIs admin nuevas

| API | Métodos |
|---|---|
| `app/api/admin/product-options/route.ts` | POST/PATCH/DELETE opciones y extras |
| `app/api/admin/combos/route.ts` | POST/PATCH/DELETE combos |
| `app/api/admin/schedule/route.ts` | POST/PATCH/DELETE horarios |
| `app/api/admin/bookings/route.ts` | POST/PATCH/DELETE citas |

---

### A5. Permisos por rol (ya funcionando, se reutiliza)

- `role_permissions` ya filtra por módulo.
- Agregar módulos nuevos (`restaurante`, `servicios`) a la migración.
- En `AdminShell` el menú se filtra según los permisos del usuario autenticado (obtener permisos en `requireAdmin`/servidor y pasarlos como `initialPermissions` prop).

---

## 🏪 FASE B — MÓDULO COMERCIO (tienda normal — lo que ya tenemos)

> Ya está 90% completo. Solo se refuerza con lo transversal.

**Plus que se pueden agregar sin romper:**

| Plus | Descripción | Esfuerzo |
|---|---|---|
| **Ratings/reviews** | Tabla `product_reviews` (puntaje 1-5 + comentario) para fidelizar | Media |
| **Etiquetas (`tags`)** | `products.tags text[]` para filtrar "Nuevo", "Promo", "Premium" en el catálogo | Baja |
| **Stock múltiple (sucursales)** | `warehouses` + `warehouse_stock` para multi-ubicación | Alta |
| **Carrito persistente por usuario** | Tabla `carts` para recuperar carrito en otro dispositivo | Media |
| **Descuentos por cupón** | `coupons` (código + % + validez) aplicados en el checkout | Media |
| **Pago en línea / link de pago** | Enlace de pago generado al cerrar solicitud | Media |

**Vista admin**: ya existe (`/admin/productos`). Solo se añade `product_type='product'` como filtro por defecto.

**Vista pública**: card + carrito + WhatsApp (sin cambios) — el flujo actual.

---

## 🍽️ FASE C — MÓDULO RESTAURANTE

### C1. Requisitos de software

1. `product_type='product'` + `unit_label='ración'` para menú.
2. Opciones y extras (`product_options`, `product_option_values`, `product_extras`).
3. Combos (`product_components`).
4. Horarios (`product_schedule`).
5. `order_type` + `table_number` en solicitudes.
6. Impresión de cocina (kitchen ticket).
7. Estaciones de cocina (agrupar ítems por área).
8. Estado del pedido: `confirmed = en cocina`, `shipped = listo`.

### C2. Plus recomendados para reforzar

| Plus | Descripción | Impacto |
|---|---|---|
| **Kitchen ticket** | Vista imprimible `order-ticket` que agrupa por estación (parrilla/barra/postres) al pasar a `confirmed` | Alto |
| **Horarios del menú** | `product_schedule` oculta platos fuera de franja (ej: desayuno hasta 11am) | Alto |
| **Combos** | `product_components` arma combos con descuento automático (2x1, menú del día) | Alto |
| **Opciones de alergias/temperatura** | `product_options` con `required` (punto de carne, sin cebolla) | Alto |
| **Rastreo de comanda** | Timeline del pedido (recibido → cocina → listo → entregado) usando `solicitud_status_history` | Medio |
| **Bebidas/servicio** | Paquete de servicios (reserva de mesa) combinando `product_type='service'` + `bookings` | Medio |
| **IVA diferenciado** | `tax_rate` por producto (comida 16%, bebidas 16%, postres 16%) | Medio |

### C3. Vistas propuestas

| Vista | Ruta | Descripción |
|---|---|---|
| **Menú del administrador** | `/admin/restaurante/menu` | Tabla con platos, opciones, extras, combos, horarios (pestañas) |
| **Combos** | `/admin/restaurante/combos` | Armar combos con búsqueda de componentes |
| **Pedidos de cocina** | `/admin/restaurante/pedidos` | Tabla de solicitudes con filtro por estación y impresión de ticket |
| **Horarios** | `/admin/restaurante/horarios` | Editor semanal de franjas por producto |

### C4. Vista pública

Portada con **pestañas por categoría** (Desayuno/Platos/Postres/Bebidas). En cada plato: card con botón **"Personalizar"** que abre modal con opciones/extras. Al agregar se muestra el total con extras. Botón **"Pedir a mesa"** (si `order_type=dine_in`) o **"Llevar/Entrega"**.

---

## 🛠️ FASE D — MÓDULO SERVICIOS

### D1. Requisitos de software

1. `product_type='service'` — no usa stock, ni gramos, ni envío.
2. `duration_minutes` — duración del servicio.
3. `sessions_count` — sesiones/paquetes.
4. `bookings` — agenda de citas.
5. `staff` (técnicos) — quién ejecuta.
6. `service_checklist` JSONB — checklist de tareas.

### D2. Plus recomendados para reforzar

| Plus | Descripción | Impacto |
|---|---|---|
| **Agenda visual** | Calendario semanal/diario con citas por técnico y estado (pendiente/completada/cancelada) | Alto |
| **Cupos por fecha** | `stock` = cupos disponibles por fecha para un servicio; `low_stock_threshold` para avisar "quedan 2 cupos" | Alto |
| **Recordatorios** | Integración con email/WhatsApp al crear cita (ya hay SMTP) | Medio |
| **Checklist de servicio** | `service_checklist` JSONB por producto para inspección/visita técnica (pasos a seguir) | Medio |
| **Paquetes de sesiones** | `sessions_count` + tracking de sesiones usadas/restantes del cliente | Medio |
| **Certificados/notas** | Archivo PDF/nota de servicio por cita completada | Bajo |

### D3. Vistas propuestas

| Vista | Ruta | Descripción |
|---|---|---|
| **Catálogo de servicios** | `/admin/servicios/catalogo` | Productos con `product_type='service'` (duración, cupos, técnicos) |
| **Agenda** | `/admin/servicios/agenda` | Calendario con citas por día/técnico |
| **Técnicos** | `/admin/servicios/tecnicos` | CRUD de personal que ejecuta servicios |
| **Mis citas (cliente)** | `/perfil/citas` | Próximas y pasadas citas del cliente |

### D4. Vista pública

Catálogo con **filtro por tipo de servicio** y cards que muestran **duración + duración + cupos**. Botón **"Agendar cita"** abre modal con fecha/hora/técnico (en vez de "Agregar al carrito"). El cliente ve "Mis citas" en su perfil.

---

## 🧩 FASE E — CATÁLOGO PÚBLICO (dinámico por módulos activos)

1. **`lib/constants.ts`** `ACTIVE_MODULES` controla qué se muestra:
   - Solo comercio → la vista actual (cards + carrito + WhatsApp).
   - Comercio + restaurante → portada con pestañas de categorías y personalización de platos.
   - Comercio + servicios → filtro "Productos / Servicios" y botón de agendar.
   - Todos → catálogo unificado con pestañas: **🏪 Tienda | 🍽️ Restaurante | 🛠️ Servicios**.
2. **`CatalogView`** recibe `modules` en props (server → client) y renderiza filtros/tabs según activos.
3. **`Storefront`** puede mostrar secciones dinámicas (ej: si `restaurante="SI"`, agregar hero de menú).
4. **Footer/Header** sin cambios (ya son dinámicos).

---

## 🗓️ ORDEN DE EJECUCIÓN (para no trabajar doble)

| # | Paso | Entregable | Depende |
|---|---|---|---|
| 1 | Migración 0028 base multi-rubro | SQL idempotente | — |
| 2 | `lib/constants.ts` ACTIVE_MODULES + `.env` 3 flags | Config | — |
| 3 | Servicios `product-options`, `combos`, `schedule`, `bookings` | Capa de datos | 1 |
| 4 | APIs admin nuevas | Endpoints | 3 |
| 5 | `mapProduct` + `PRODUCT_SELECT` ampliados | Servicio catálogo | 1 |
| 6 | Tipos de producto/servicio en `services/requests.ts` | Solicitudes | 1,5 |
| 7 | Permisos por rol para nuevos módulos (`AdminShell` filtra por permisos) | Seguridad | 1 |
| 8 | Menú admin dinámico por `ACTIVE_MODULES` + permisos | UI Admin | 2,7 |
| 9 | **FASE C** — Módulo Restaurante (vistas + API + catálogo público) | Restaurante | 1-8 |
| 10 | **FASE D** — Módulo Servicios (vistas + API + perfil citas) | Servicios | 1-8 |
| 11 | **FASE E** — Catálogo público unificado con tabs | Público | 2,8,9,10 |
| 12 | Pruebas de regresión (76 tests + nuevos) + ajustes de UI | QA | 1-11 |

---

## 🧪 Testing económico (FASE F)

- **`tests/product-options.test.mjs`**: cálculo de precio de un plato con opciones/extras (ej: base $10 + extra queso $2 + término medio $0 = $12).
- **`tests/combos.test.mjs`**: precio del combo = suma de componentes × cantidad − descuento.
- **`tests/bookings.test.mjs`**: validación de fechas, duración, cupos por día y solapamiento de técnicos.
- **`tests/quote-calculator.test.mjs`**: sigue funcionando (sin cambios), añadir caso `tax_rate`.

---

## 📌 Resumen de arquitectura final

```
app/
├── admin/
│   ├── productos/          → MÓDULO COMERCIO (product_type='product')
│   ├── restaurante/
│   │   ├── menu/           → Platos + opciones + extras
│   │   ├── combos/         → Combos
│   │   ├── pedidos/        → Solicitudes de restaurante + kitchen tickets
│   │   └── horarios/       → Franjas horarias
│   ├── servicios/
│   │   ├── catalogo/       → Servicios (duración, cupos, técnicos)
│   │   ├── agenda/         → Calendario de citas
│   │   └── tecnicos/       → Personal
│   └── ... (resto igual)
components/
├── admin-shell.tsx         → Menú dinámico por ACTIVE_MODULES + permisos
├── menu-manager.tsx        → Nuevo (restaurante)
├── combo-manager.tsx       → Nuevo (combos)
├── booking-manager.tsx     → Nuevo (servicios)
├── booking-calendar.tsx    → Nuevo (agenda)
services/
├── catalog.ts              → Ampliado (mapProduct + relaciones)
├── product-options.ts      → Nuevo
├── combos.ts               → Nuevo
├── schedule.ts             → Nuevo
├── bookings.ts             → Nuevo
supabase/migrations/
├── 0027_product_pricing_units.sql → Ya hecho
├── 0028_multi_module_base.sql      → Este plan
```

---

## ✅ Conclusión

Con esta estrategia:

1. **No rehacemos nada**: la base actual (catálogo + solicitudes + cotizaciones + facturas + permisos) se reutiliza al 100%.
2. **Los 3 módulos se activan con 3 variables SI/NO** en `.env` — el sistema se vende como comercio, restaurante, servicios o combo completo.
3. **El menú admin se adapta**: cada rol ve solo sus módulos permitidos, sin sobrecarga.
4. **El catálogo público se adapta**: tienda / menú / citas según lo activado.
5. **Cada módulo trae su plus**: kitchen ticket, horarios, combos, agenda visual, cupos, checklist, IVA por producto.
6. **Todo paso a paso** con migraciones idempotentes y tests económicos, sin romper los 76 tests existentes.

**Empieza con la FASE A (migración 0028 + ACTIVE_MODULES + permisos)** y luego módulo por módulo.