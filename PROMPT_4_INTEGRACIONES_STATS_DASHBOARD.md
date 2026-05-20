# PROMPT 4 — Importaciones + Finanzas + Estadísticas + Dashboard (EJECUTAR ÚLTIMO)

> Los módulos core (Productos, Inventario, Recetas, POS, Compras) están implementados. Ahora completa la app con las integraciones de canales externos, finanzas, estadísticas y el dashboard unificado.

---

Eres un arquitecto de software senior. Implementa los módulos restantes de RestoApp en este orden: ImportOlaClick → ImportRappi → Finance → Statistics → Distribution → CashRegister → Customers → Settings → Dashboard.

---

## MÓDULO 1 — ImportOlaClick.jsx (src/pages/ImportOlaClick.jsx)

**Flujo completo:**

```
1. Zona de drop/upload de archivo .xlsx (SheetJS)
2. Al subir: leer hoja "Productos" y hoja "Pagos" con SheetJS
3. Mostrar preview de datos parseados antes de confirmar importación
4. Paso de mapeo: para cada producto del xlsx sin olaclick_name mapeado,
   mostrar select "¿Este producto es cuál en RestoApp?"
5. Al confirmar: guardar en Supabase
```

**Parseo exacto con SheetJS:**
```js
import * as XLSX from 'xlsx';

const parseOlaClick = (file) => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const wb = XLSX.read(e.target.result, { type: 'array' });

      // Hoja Productos
      const productosSheet = wb.Sheets['Productos'];
      const productos = XLSX.utils.sheet_to_json(productosSheet);
      // Columnas esperadas:
      // "Nombre de la variante de producto", "Cantidad vendida",
      // "Total vendido", "Ganancia por unidad", "Categoría"

      // Hoja Pagos
      const pagosSheet = wb.Sheets['Pagos'];
      const pagos = pagosSheet ? XLSX.utils.sheet_to_json(pagosSheet) : [];
      // Columnas esperadas: "Método de pago", "Monto total recibido"

      // Hoja Pedidos (para fechas)
      const pedidosSheet = wb.Sheets['Pedidos'];
      const pedidos = pedidosSheet ? XLSX.utils.sheet_to_json(pedidosSheet) : [];
      // Para extraer: period_start, period_end, total_orders

      resolve({ productos, pagos, pedidos });
    };
    reader.readAsArrayBuffer(file);
  });
};
```

**Al confirmar importación — secuencia exacta:**
```js
// 1. Calcular totales
const total_sales = productos.reduce((s, r) => s + (r['Total vendido'] || 0), 0);
const total_orders = pedidos.length;
const period_start = pedidos.length ? Math.min(...pedidos.map(p => new Date(p['Fecha']))) : null;
const period_end   = pedidos.length ? Math.max(...pedidos.map(p => new Date(p['Fecha']))) : null;

// 2. INSERT en olaclick_imports
const { data: importRecord } = await supabase.from('olaclick_imports').insert({
  restaurant_id: restaurantId,
  file_name: file.name,
  period_start: period_start ? new Date(period_start).toISOString().split('T')[0] : null,
  period_end:   period_end   ? new Date(period_end).toISOString().split('T')[0]   : null,
  total_sales,
  total_orders,
  raw_products_json: productos,
  raw_payments_json: pagos
}).select().single();

// 3. Crear venta consolidada source='olaclick'
const { data: sale } = await supabase.from('sales').insert({
  restaurant_id: restaurantId,
  source: 'olaclick',
  total: total_sales,
  is_credit: false,
  sale_date: period_end ? new Date(period_end).toISOString().split('T')[0] : new Date().toISOString().split('T')[0]
}).select().single();

// 4. Para cada producto mapeado → INSERT sale_item
for (const row of productos) {
  const localProduct = products.find(p =>
    p.olaclick_name === row['Nombre de la variante de producto']
  );
  if (localProduct) {
    await supabase.from('sale_items').insert({
      sale_id: sale.id,
      product_id: localProduct.id,
      product_name: localProduct.name,
      quantity: row['Cantidad vendida'] || 0,
      unit_price: row['Ganancia por unidad'] || 0,
      unit_cost: localProduct.cost,
      subtotal: row['Total vendido'] || 0
    });
  }
}

// 5. Mostrar resumen: total vendido, órdenes, productos mapeados vs no mapeados
```

**UI adicional:**
- Historial de importaciones anteriores (tabla con fecha, archivo, total, órdenes)
- Botón "Ver detalle" para ver los productos de una importación anterior

---

## MÓDULO 2 — ImportRappi.jsx (src/pages/ImportRappi.jsx)

**Parseo con SheetJS:**
```js
const parseRappi = (file) => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const wb = XLSX.read(e.target.result, { type: 'array' });
      const sheet = wb.Sheets['Detalle'] || wb.Sheets[wb.SheetNames[0]];
      const rows = XLSX.utils.sheet_to_json(sheet);
      // Filtrar solo órdenes no canceladas
      const activas = rows.filter(r =>
        r['Estado de la orden'] !== 'CANCELLED' &&
        r['Estado de la orden'] !== 'CANCELADO'
      );
      resolve(activas);
    };
    reader.readAsArrayBuffer(file);
  });
};
```

**Cálculo de totales Rappi:**
```js
const calcRappiTotals = (rows) => {
  const venta_bruta      = rows.reduce((s, r) => s + (Number(r['Venta Bruta']) || 0), 0);
  const comision_20pct   = rows.reduce((s, r) => s + (Number(r['Descuento de Producto asumido por el aliado'] || r['Descuento asumido por aliado']) || 0), 0);
  const tarifa_plataforma = rows.reduce((s, r) => s + (Number(r['Uso y alquiler de plataforma Rappi'] || r['Uso y alquiler plataforma Rappi']) || 0), 0);
  const iva_retefuente   = rows.reduce((s, r) => s + (Number(r['IVA'] || 0) + Number(r['Retefuente'] || 0)), 0);
  const valor_neto       = venta_bruta + comision_20pct + tarifa_plataforma + iva_retefuente;
  // comision y tarifa son negativos en el xlsx

  // Día de semana más vendido
  const days = ['Domingo','Lunes','Martes','Miércoles','Jueves','Viernes','Sábado'];
  const byDay = {};
  rows.forEach(r => {
    const d = days[new Date(r['Fecha de creación orden'] || r['Fecha']).getDay()];
    byDay[d] = (byDay[d] || 0) + (Number(r['Venta Bruta']) || 0);
  });
  const best_day_of_week = Object.entries(byDay).sort((a,b) => b[1]-a[1])[0]?.[0] || null;

  return { venta_bruta, comision_20pct, tarifa_plataforma, iva_retefuente, valor_neto,
           total_orders: rows.length, best_day_of_week };
};
```

**Al confirmar importación Rappi — secuencia exacta:**
```js
const { venta_bruta, comision_20pct, tarifa_plataforma, iva_retefuente,
        valor_neto, total_orders, best_day_of_week } = calcRappiTotals(rows);

// 1. Crear venta a CRÉDITO (el dinero no entra hasta confirmar pago)
const { data: sale } = await supabase.from('sales').insert({
  restaurant_id: restaurantId,
  source: 'rappi',
  total: venta_bruta,
  is_credit: true,
  credit_paid_at: null,
  sale_date: new Date().toISOString().split('T')[0]
}).select().single();

// 2. INSERT rappi_imports
const { data: rappiImport } = await supabase.from('rappi_imports').insert({
  restaurant_id: restaurantId,
  period_start: periodStart,
  period_end: periodEnd,
  venta_bruta, comision_20pct, tarifa_plataforma, iva_retefuente, valor_neto,
  total_orders, best_day_of_week,
  credit_sale_id: sale.id
}).select().single();

// 3. Crear gastos automáticos (comisiones son negativas → guardar en positivo)
await supabase.from('expenses').insert([
  {
    restaurant_id: restaurantId,
    type: 'rappi_fee',
    description: 'Comisión Rappi 20%',
    amount: Math.abs(comision_20pct),
    fund: 'Rappi',
    expense_date: new Date().toISOString().split('T')[0]
  },
  {
    restaurant_id: restaurantId,
    type: 'rappi_fee',
    description: 'Tarifa plataforma Rappi',
    amount: Math.abs(tarifa_plataforma),
    fund: 'Rappi',
    expense_date: new Date().toISOString().split('T')[0]
  }
]);

// 4. Mostrar resumen:
// Bruto: $X | Comisión: -$Y | Tarifa: -$Z | Neto real: $W
// "Crédito pendiente — se confirma cuando Rappi pague"
// Botón "Marcar como pagado" en la lista de importaciones
```

**Panel Rappi — lista de importaciones:**
- Por cada importación: período, bruto, neto, órdenes, estado (Pendiente / Pagado)
- Botón "Marcar como pagado" → `UPDATE sales SET credit_paid_at = NOW()` + `UPDATE rappi_imports SET paid_at = NOW()`
- Badge verde "Pagado" / amarillo "Pendiente"

---

## MÓDULO 3 — Finance.jsx (src/pages/Finance.jsx)

**4 secciones en tabs:**

### Tab 1 — Ingresos
```
Tabla de ventas (sales) con filtro de mes:
- POS: ventas directas (is_credit=false)
- OlaClick: ventas source='olaclick'
- Rappi PAGADO: source='rappi' AND credit_paid_at IS NOT NULL
- Total = suma de los anteriores
Excluir ventas Rappi con credit_paid_at IS NULL (son crédito pendiente)
```

### Tab 2 — Gastos
```
Tabla de expenses con filtro de mes y fondo
Tipos: simple / purchase (con link a factura) / rappi_fee
Total por fondo (Materia Prima, Nómina, etc.)
Botón "Nuevo gasto simple" → modal rápido
```

### Tab 3 — Créditos Rappi pendientes
```
SELECT * FROM rappi_imports WHERE restaurant_id=$id AND paid_at IS NULL
Por cada uno mostrar: período, venta bruta, neto real, total_orders
Botón "Marcar como pagado" → actualiza sales + rappi_imports
Suma total pendiente de cobro
```

### Tab 4 — Saldo por fondo
```
Basado en income_distribution:
Por cada fondo: % configurado × ingresos totales del mes = asignado
Mostrar cuánto se ha gastado de ese fondo (join con expenses.fund)
Saldo disponible = asignado - gastado
Barra de progreso visual
```

---

## MÓDULO 4 — Statistics.jsx (src/pages/Statistics.jsx)

**Filtros globales:** rango de fechas (inicio / fin) — por defecto: mes actual

**Sección 1 — Resumen multi-canal**
```
KPI cards:
- Total ventas (POS + OlaClick + Rappi pagado)
- Total gastos
- Utilidad neta = ventas - gastos
- Ticket promedio = total / cantidad de ventas
- Número de órdenes total
```

**Sección 2 — Ventas por canal (Recharts BarChart)**
```js
// Query
const { data } = await supabase.from('sales')
  .select('source, total')
  .eq('restaurant_id', restaurantId)
  .gte('sale_date', startDate)
  .lte('sale_date', endDate)
  .or('is_credit.eq.false,credit_paid_at.not.is.null');

// Agrupar por source → data para BarChart
// [{ source: 'pos', total: X }, { source: 'olaclick', total: Y }, { source: 'rappi', total: Z }]
```

**Sección 3 — Top 10 productos más vendidos (Recharts HorizontalBarChart)**
```js
// Query
const { data } = await supabase.from('sale_items')
  .select('product_name, quantity, subtotal, sales!inner(sale_date, restaurant_id)')
  .eq('sales.restaurant_id', restaurantId)
  .gte('sales.sale_date', startDate)
  .lte('sales.sale_date', endDate);

// Agrupar por product_name → sumar quantity y subtotal
// Ordenar por quantity DESC → top 10
```

**Sección 4 — Ventas por día de semana (Recharts BarChart)**
```js
// Misma query de sales, agrupar por dayOfWeek(sale_date)
// ['Lunes','Martes',...,'Domingo'] → total por día
// Resaltar el día con más ventas
```

**Sección 5 — Evolución 30 días (Recharts LineChart)**
```js
// Sales agrupadas por sale_date → línea de evolución
// Múltiples series: POS / OlaClick / Rappi en colores distintos
```

---

## MÓDULO 5 — Distribution.jsx (src/pages/Distribution.jsx)

**Funcionalidades:**
- Lista de fondos con su porcentaje y color
- Suma siempre debe ser 100% — validar al guardar
- Editar porcentaje y nombre inline
- Agregar / eliminar fondos
- Gráfica de dona (Recharts PieChart) que se actualiza en tiempo real
- Botón "Guardar distribución"
- Panel de aplicación: dado el total de ventas del mes → mostrar cuánto corresponde a cada fondo

**Queries:**
```js
// Leer
supabase.from('income_distribution').select('*').eq('restaurant_id', restaurantId).order('sort_order')

// Guardar (replace total)
// 1. DELETE existing
await supabase.from('income_distribution').delete().eq('restaurant_id', restaurantId);
// 2. INSERT nuevos
await supabase.from('income_distribution').insert(fondos.map((f, i) => ({
  ...f, restaurant_id: restaurantId, sort_order: i
})));
```

---

## MÓDULO 6 — CashRegister.jsx (src/pages/CashRegister.jsx)

**Flujo:**
- Ver el arqueo del día actual (si existe) o crear uno nuevo
- Al abrir turno: ingresar efectivo inicial (`opening_cash`)
- Durante el día: ventas y gastos se registran en sus módulos
- Al cerrar turno: el sistema calcula automáticamente:
  ```
  sales_total    = SELECT SUM(total) FROM sales WHERE sale_date=hoy AND source IN ('pos','olaclick') AND restaurant_id=$id
  expenses_total = SELECT SUM(amount) FROM expenses WHERE expense_date=hoy AND restaurant_id=$id
  closing_cash   = (campo manual — lo que hay físicamente en caja)
  difference     = closing_cash - (opening_cash + sales_total_efectivo - expenses_total)
  ```
- Notas de cierre (text)
- Botón "Generar PDF del arqueo" → jsPDF con autotable

**PDF del arqueo (jsPDF):**
```
Encabezado: nombre restaurante + fecha
Sección ventas: tabla con source / total
Sección gastos: tabla con descripción / monto
Totales
Efectivo: apertura / cierre / diferencia
Notas
```

---

## MÓDULO 7 — Customers.jsx (src/pages/Customers.jsx)

- Lista de clientes con búsqueda por nombre o teléfono
- Crear / editar cliente: nombre, teléfono, dirección
- Perfil de cliente: ver historial de ventas (`sales JOIN sale_items WHERE customer_id = X`)
- Total comprado, última compra, cantidad de pedidos

---

## MÓDULO 8 — Settings.jsx (src/pages/Settings.jsx)

**Secciones:**

### Medios de pago
```
Lista de payment_methods del restaurante
Toggle activo/inactivo
Agregar nuevo medio de pago
```

### Datos del restaurante
```
Editar nombre del restaurante
Timezone (select — solo America/Bogota por ahora)
```

### Categorías de productos
```
Lista de categorías únicas de products
Opción para renombrar (update masivo)
```

---

## MÓDULO 9 — Dashboard.jsx (src/pages/Dashboard.jsx) — EL ÚLTIMO

**Este módulo agrega todo lo anterior. Implementar al final.**

**Layout: grid de 2 columnas en desktop, 1 en móvil**

### Fila 1 — KPIs del día (4 cards)
```js
// Ventas hoy (POS + OlaClick)
SELECT SUM(total) FROM sales WHERE sale_date = CURRENT_DATE AND restaurant_id = $id AND source != 'rappi'

// Gastos hoy
SELECT SUM(amount) FROM expenses WHERE expense_date = CURRENT_DATE AND restaurant_id = $id

// Utilidad hoy
ventas_hoy - gastos_hoy

// Órdenes hoy
SELECT COUNT(*) FROM sales WHERE sale_date = CURRENT_DATE AND restaurant_id = $id
```

### Fila 2 — KPIs del mes (4 cards)
```
Ventas mes | Gastos mes | Utilidad mes | Ticket promedio mes
```

### Fila 3 — Gráficas (2 columnas)
```
Izquierda: LineChart — Ventas últimos 30 días (una línea por canal)
Derecha:   BarChart  — Top 5 productos del mes (quantity)
```

### Fila 4 — Estado operacional (2 columnas)
```
Izquierda: "Alertas de stock bajo"
  SELECT * FROM ingredients WHERE stock_current <= stock_minimum AND restaurant_id=$id
  Lista: nombre, stock actual, stock mínimo, unidad — badge rojo

Derecha: "Créditos Rappi pendientes"
  SELECT * FROM rappi_imports WHERE paid_at IS NULL AND restaurant_id=$id
  Por cada uno: período, valor neto pendiente
  Total pendiente de cobro
  Botón "Marcar como pagado" directo desde el dashboard
```

### Fila 5 — Resumen de canales del mes
```
Tabla: Canal | Ventas | % del total | Órdenes | Ticket promedio
Canales: POS / OlaClick / Rappi (pagado)
```

**Todas las queries del dashboard deben ejecutarse en paralelo con `Promise.all` para no bloquear la carga.**

---

## NOTA FINAL — Recharts en todos los módulos con gráficas

Configuración base reutilizable:
```jsx
// Paleta de colores consistente
const COLORS = {
  pos:      '#7c6dfa',   // violeta
  olaclick: '#10b981',   // verde
  rappi:    '#f59e0b',   // amarillo
  expenses: '#ef4444',   // rojo
  neutral:  '#6b7280'    // gris
};

// Formato de pesos colombianos
const formatCOP = (v) => `$${Number(v).toLocaleString('es-CO')}`;

// Todos los charts: responsive wrapper
<ResponsiveContainer width="100%" height={300}>
  ...
</ResponsiveContainer>
```

Sin pausar. Sin preguntar. Implementar todos los módulos completos y funcionales. Verificar que las queries de Supabase incluyan siempre el filtro `restaurant_id` para respetar el RLS multi-tenant.
