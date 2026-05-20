# PROMPT 3 — Módulos core (EJECUTAR DESPUÉS DEL LAYOUT)

> La app tiene Auth, SessionContext, Layout y rutas configuradas. Ahora implementa los módulos core en orden estricto: Productos → Inventario → Recetas → POS → Compras.

---

Eres un arquitecto de software senior. Implementa los 5 módulos core de RestoApp. Usa `useRestaurant()` para obtener `restaurantId`. Toda operación de DB usa el cliente Supabase de `src/lib/supabase.js`. No uses Redux ni Zustand — estado local con `useState`/`useEffect`.

---

## MÓDULO 1 — Products.jsx (src/pages/Products.jsx)

**Funcionalidades:**
- Tabla con todos los productos activos del restaurante: nombre, categoría, precio, costo, margen (%), estado activo/inactivo
- Margen = `((precio - costo) / precio * 100).toFixed(1)` — en verde si > 60%, amarillo 40-60%, rojo < 40%
- Botón "Nuevo producto" → modal con formulario
- Editar producto (click en fila o botón editar)
- Toggle activo/inactivo sin modal (switch inline)
- Campo `olaclick_name` visible en edición (label: "Nombre en OlaClick")
- Filtro por categoría (select + "Todas")
- Búsqueda por nombre (input)

**Formulario de producto:**
```
nombre (text, required)
categoría (text — sugerencias: Postres, Platos, Bebidas, Adicionales)
precio (number, required, min 0)
costo (number, min 0 — label: "Costo estimado (se actualiza con receta)")
olaclick_name (text — label: "Nombre exacto en OlaClick/Rappi xlsx")
activo (checkbox)
```

**Queries Supabase:**
```js
// Listar
supabase.from('products').select('*').eq('restaurant_id', restaurantId).order('name')

// Crear
supabase.from('products').insert({ ...data, restaurant_id: restaurantId })

// Actualizar
supabase.from('products').update(data).eq('id', id)
```

---

## MÓDULO 2 — Inventory.jsx (src/pages/Inventory.jsx)

**Funcionalidades:**
- Tabla de ingredientes: nombre, unidad, stock actual, stock mínimo, costo/unidad, proveedor
- Indicador visual de stock: verde (> mínimo), amarillo (= mínimo), rojo (< mínimo) — badge de color
- Botón "Nuevo ingrediente" → modal con formulario
- Editar ingrediente
- Ajuste manual de stock: input numérico inline con botón "Guardar ajuste" → actualiza `stock_current` y genera movimiento tipo `ajuste`
- Tab "Movimientos" → tabla de `inventory_movements` del ingrediente seleccionado (type, quantity, notes, created_at)
- Búsqueda por nombre

**Formulario de ingrediente:**
```
nombre (text, required)
unidad (select: g, kg, ml, L, unidad, porción)
stock actual (number)
stock mínimo (number)
costo por unidad (number — label: "Costo por unidad (se actualiza en compras)")
proveedor (text)
```

**Queries Supabase:**
```js
// Listar con alerta
supabase.from('ingredients').select('*').eq('restaurant_id', restaurantId).order('name')

// Ajustar stock
supabase.from('ingredients').update({ stock_current: nuevoValor }).eq('id', id)
// El trigger de DB insertará el movimiento automáticamente

// Ver movimientos
supabase.from('inventory_movements')
  .select('*').eq('ingredient_id', ingredientId).order('created_at', { ascending: false })
```

---

## MÓDULO 3 — Recipes.jsx (src/pages/Recipes.jsx)

**Funcionalidades:**
- Lista de recetas del restaurante: nombre, tipo (receta/sub-receta), producto vinculado, rendimiento, costo total calculado
- Crear/editar receta con ingredientes
- Agregar ingredientes a la receta (buscar por nombre) O agregar sub-receta
- Al guardar: calcular `total_cost` = Σ (quantity * ingredient.cost_per_unit) + Σ (sub_recipe_cost * quantity / sub_recipe.yield_quantity)
- Después de calcular costo de receta → actualizar `products.cost` si la receta tiene `product_id`
- Pestaña "Sub-recetas" para ver/crear las que tienen `is_sub_recipe = true`

**Formulario de receta:**
```
nombre (text, required)
producto vinculado (select de products — opcional)
es sub-receta (checkbox)
rendimiento cantidad (number)
rendimiento unidad (text)
--- líneas de ingredientes ---
  [ingrediente o sub-receta] [cantidad] [unidad] [costo calculado] [eliminar]
  botón "Agregar ingrediente" / "Agregar sub-receta"
--- total calculado (se actualiza en tiempo real) ---
```

**Cálculo de costo en tiempo real:**
```js
const calcRecipeCost = (lines, ingredients, subRecipes) => {
  return lines.reduce((total, line) => {
    if (line.ingredient_id) {
      const ing = ingredients.find(i => i.id === line.ingredient_id);
      return total + (line.quantity * (ing?.cost_per_unit || 0));
    }
    if (line.sub_recipe_id) {
      const sub = subRecipes.find(r => r.id === line.sub_recipe_id);
      const costPerUnit = sub ? sub.total_cost / sub.yield_quantity : 0;
      return total + (line.quantity * costPerUnit);
    }
    return total;
  }, 0);
};
```

**Queries Supabase:**
```js
// Recetas con sus líneas
supabase.from('recipes').select(`
  *, 
  recipe_ingredients(*, ingredients(*), sub_recipe:recipes(*))
`).eq('restaurant_id', restaurantId)

// Guardar receta (transacción manual)
// 1. Upsert recipe con total_cost calculado
// 2. Delete existing recipe_ingredients del recipe_id
// 3. Insert nuevas recipe_ingredients
// 4. Update products.cost si hay product_id
```

---

## MÓDULO 4 — POS.jsx (src/pages/POS.jsx)

**Funcionalidades:**
- Panel izquierdo: catálogo de productos activos en cards (nombre, precio, categoría)
  - Filtro por categoría (tabs)
  - Búsqueda por nombre
  - Click en producto → agregar al carrito
- Panel derecho: carrito
  - Lista de items: nombre, cantidad (+ / -), precio unitario, subtotal, eliminar
  - Campo descuento (número o %)
  - Selector de cliente (buscar en `customers` o "Sin cliente")
  - Selector de medio de pago (lista de `payment_methods` activos)
  - Toggle "Venta a crédito" (para Rappi manual — oculto por defecto)
  - Total a cobrar
  - Botón "Registrar venta"

**Al registrar venta — secuencia exacta:**
```js
// 1. INSERT en sales
const { data: sale } = await supabase.from('sales').insert({
  restaurant_id: restaurantId,
  source: 'pos',
  customer_id: selectedCustomer?.id || null,
  payment_method_id: selectedMethod.id,
  total: totalConDescuento,
  discount: descuento,
  is_credit: false,
  sale_date: new Date().toISOString().split('T')[0]
}).select().single();

// 2. INSERT en sale_items
await supabase.from('sale_items').insert(
  cartItems.map(item => ({
    sale_id: sale.id,
    product_id: item.product.id,
    product_name: item.product.name,       // snapshot
    quantity: item.quantity,
    unit_price: item.product.price,        // snapshot
    unit_cost: item.product.cost,          // snapshot
    subtotal: item.quantity * item.product.price
  }))
);

// 3. Para cada item, descontar ingredientes según la receta
for (const item of cartItems) {
  const { data: recipeLines } = await supabase
    .from('recipe_ingredients')
    .select('*, recipes!inner(product_id)')
    .eq('recipes.product_id', item.product.id);

  for (const line of recipeLines || []) {
    if (line.ingredient_id) {
      await supabase.rpc('decrement_stock', {
        p_ingredient_id: line.ingredient_id,
        p_quantity: line.quantity * item.quantity
      });
    }
  }
}

// 4. Verificar stock bajo y mostrar Alert si corresponde
```

**Función RPC en Supabase (crear junto con este módulo):**
```sql
CREATE OR REPLACE FUNCTION decrement_stock(p_ingredient_id uuid, p_quantity numeric)
RETURNS void AS $$
  UPDATE ingredients
  SET stock_current = GREATEST(0, stock_current - p_quantity)
  WHERE id = p_ingredient_id;
$$ LANGUAGE sql SECURITY DEFINER;
```

**Post-venta:**
- Mostrar resumen: "Venta registrada — Total: $X — Cambio: $Y" (si efectivo)
- Vaciar carrito
- Opción "Imprimir recibo" (jsPDF — simple: lista de items + total + fecha)

---

## MÓDULO 5 — Purchases.jsx (src/pages/Purchases.jsx)

**Dos modos de registro:**

### Modo A — Gasto simple
```
descripción (text)
monto (number)
fondo (select: Materia Prima, Nómina, Servicios, Varios)
fecha (date)
→ INSERT en expenses (type='simple')
```

### Modo B — Factura con ingredientes
```
proveedor (text)
número de factura (text)
fecha (date)
fondo (select)
--- líneas de ingredientes ---
  [ingrediente] [cantidad] [unidad] [costo total línea] [costo/unidad calculado]
  botón "Agregar línea"
--- total factura (suma automática) ---
```

**Al guardar Factura — secuencia exacta:**
```js
// 1. INSERT purchases
const purchase = await supabase.from('purchases').insert({...}).select().single();

// 2. INSERT purchase_items con prev_cost_per_unit
for (const line of lines) {
  const { data: ing } = await supabase
    .from('ingredients').select('cost_per_unit').eq('id', line.ingredient_id).single();

  const newCostPerUnit = line.total_cost / line.quantity;
  const prevCost = ing.cost_per_unit;

  await supabase.from('purchase_items').insert({
    purchase_id: purchase.id,
    ingredient_id: line.ingredient_id,
    quantity: line.quantity,
    unit: line.unit,
    total_cost: line.total_cost,
    cost_per_unit: newCostPerUnit,
    prev_cost_per_unit: prevCost
  });

  // 3. Actualizar ingrediente: stock += qty, nuevo costo
  await supabase.from('ingredients').update({
    stock_current: ing.stock_current + line.quantity,  // el trigger registra movimiento
    cost_previous: prevCost,
    cost_per_unit: newCostPerUnit
  }).eq('id', line.ingredient_id);

  // 4. Alerta de variación si cambió más de 5%
  const variacion = Math.abs(newCostPerUnit - prevCost) / prevCost;
  if (prevCost > 0 && variacion > 0.05) {
    // Mostrar Alert: "El costo de [nombre] cambió +X%"
  }
}

// 5. Recalcular costo de recetas que usan los ingredientes comprados
// (llamar a la misma lógica de Recipes.jsx)

// 6. INSERT en expenses (type='purchase', purchase_id=purchase.id)
await supabase.from('expenses').insert({
  restaurant_id: restaurantId,
  type: 'purchase',
  description: `Factura ${purchase.invoice_number || ''} — ${purchase.supplier}`,
  amount: purchase.total_amount,
  fund: purchase.fund,
  expense_date: purchase.purchase_date,
  purchase_id: purchase.id
});
```

**Lista de compras:**
- Tabla con historial: fecha, proveedor, factura, total, fondo
- Filtro por fecha (mes actual por defecto)
- Click en fila → ver detalle con lista de ingredientes

Sin pausar. Sin preguntar. Implementar los 5 módulos completos y funcionales.
