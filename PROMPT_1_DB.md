# PROMPT 1 — Base de datos completa (EJECUTAR PRIMERO)

> Copia y pega esto en Claude Code como primera instrucción del proyecto.

---

Eres un arquitecto de software senior. Crea el esquema completo de Supabase para **RestoApp**, una app de gestión de restaurante multi-canal (POS directo + OlaClick + Rappi).

## PASO 1 — Habilitar extensiones

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

## PASO 2 — Crear todas las tablas en este orden exacto

```sql
-- 1. Restaurantes (raíz del tenant)
CREATE TABLE restaurants (
  id          uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  name        text NOT NULL,
  owner_id    uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  timezone    text NOT NULL DEFAULT 'America/Bogota',
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- 2. Productos
CREATE TABLE products (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name          text NOT NULL,
  category      text,
  price         numeric NOT NULL DEFAULT 0,
  cost          numeric NOT NULL DEFAULT 0,
  active        boolean NOT NULL DEFAULT true,
  olaclick_name text,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- 3. Ingredientes
CREATE TABLE ingredients (
  id               uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id    uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name             text NOT NULL,
  unit             text NOT NULL,
  stock_current    numeric NOT NULL DEFAULT 0,
  stock_minimum    numeric NOT NULL DEFAULT 0,
  cost_per_unit    numeric NOT NULL DEFAULT 0,
  cost_previous    numeric NOT NULL DEFAULT 0,
  supplier         text,
  created_at       timestamptz NOT NULL DEFAULT now()
);

-- 4. Recetas (incluye sub-recetas)
CREATE TABLE recipes (
  id             uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id  uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  product_id     uuid REFERENCES products(id) ON DELETE SET NULL,
  is_sub_recipe  boolean NOT NULL DEFAULT false,
  name           text NOT NULL,
  yield_quantity numeric NOT NULL DEFAULT 1,
  yield_unit     text,
  total_cost     numeric NOT NULL DEFAULT 0
);

-- 5. Ingredientes de receta (ingrediente real O sub-receta, nunca los dos)
CREATE TABLE recipe_ingredients (
  id             uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  recipe_id      uuid NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
  ingredient_id  uuid REFERENCES ingredients(id) ON DELETE SET NULL,
  sub_recipe_id  uuid REFERENCES recipes(id) ON DELETE SET NULL,
  quantity       numeric NOT NULL,
  unit           text NOT NULL,
  CONSTRAINT chk_one_source CHECK (
    (ingredient_id IS NOT NULL AND sub_recipe_id IS NULL) OR
    (ingredient_id IS NULL AND sub_recipe_id IS NOT NULL)
  )
);

-- 6. Clientes
CREATE TABLE customers (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name          text NOT NULL,
  phone         text,
  address       text,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- 7. Medios de pago
CREATE TABLE payment_methods (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  name          text NOT NULL,
  active        boolean NOT NULL DEFAULT true
);

-- 8. Ventas (POS + OlaClick + Rappi)
CREATE TABLE sales (
  id                uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id     uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  source            text NOT NULL CHECK (source IN ('pos','olaclick','rappi')),
  customer_id       uuid REFERENCES customers(id) ON DELETE SET NULL,
  payment_method_id uuid REFERENCES payment_methods(id) ON DELETE SET NULL,
  total             numeric NOT NULL,
  discount          numeric NOT NULL DEFAULT 0,
  is_credit         boolean NOT NULL DEFAULT false,
  credit_paid_at    timestamptz,
  sale_date         date NOT NULL DEFAULT CURRENT_DATE,
  notes             text,
  created_at        timestamptz NOT NULL DEFAULT now()
);

-- 9. Items de venta
CREATE TABLE sale_items (
  id           uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  sale_id      uuid NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
  product_id   uuid REFERENCES products(id) ON DELETE SET NULL,
  product_name text NOT NULL,
  quantity     numeric NOT NULL,
  unit_price   numeric NOT NULL,
  unit_cost    numeric NOT NULL DEFAULT 0,
  subtotal     numeric NOT NULL
);

-- 10. Compras / Facturas
CREATE TABLE purchases (
  id             uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id  uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  supplier       text,
  invoice_number text,
  total_amount   numeric NOT NULL,
  purchase_date  date NOT NULL DEFAULT CURRENT_DATE,
  fund           text,
  created_at     timestamptz NOT NULL DEFAULT now()
);

-- 11. Items de compra
CREATE TABLE purchase_items (
  id                 uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  purchase_id        uuid NOT NULL REFERENCES purchases(id) ON DELETE CASCADE,
  ingredient_id      uuid NOT NULL REFERENCES ingredients(id) ON DELETE RESTRICT,
  quantity           numeric NOT NULL,
  unit               text NOT NULL,
  total_cost         numeric NOT NULL,
  cost_per_unit      numeric NOT NULL,
  prev_cost_per_unit numeric NOT NULL DEFAULT 0
);

-- 12. Gastos
CREATE TABLE expenses (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  type          text NOT NULL CHECK (type IN ('simple','purchase','rappi_fee')),
  description   text NOT NULL,
  amount        numeric NOT NULL,
  fund          text,
  expense_date  date NOT NULL DEFAULT CURRENT_DATE,
  purchase_id   uuid REFERENCES purchases(id) ON DELETE SET NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- 13. Importaciones OlaClick
CREATE TABLE olaclick_imports (
  id                uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id     uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  file_name         text,
  period_start      date,
  period_end        date,
  total_sales       numeric,
  total_orders      integer,
  raw_products_json jsonb,
  raw_payments_json jsonb,
  imported_at       timestamptz NOT NULL DEFAULT now()
);

-- 14. Importaciones Rappi
CREATE TABLE rappi_imports (
  id               uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id    uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  period_start     date,
  period_end       date,
  venta_bruta      numeric NOT NULL DEFAULT 0,
  comision_20pct   numeric NOT NULL DEFAULT 0,
  tarifa_plataforma numeric NOT NULL DEFAULT 0,
  iva_retefuente   numeric NOT NULL DEFAULT 0,
  valor_neto       numeric NOT NULL DEFAULT 0,
  total_orders     integer NOT NULL DEFAULT 0,
  best_day_of_week text,
  credit_sale_id   uuid REFERENCES sales(id) ON DELETE SET NULL,
  paid_at          timestamptz,
  imported_at      timestamptz NOT NULL DEFAULT now()
);

-- 15. Distribución de ingresos por fondos
CREATE TABLE income_distribution (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  fund_name     text NOT NULL,
  percentage    numeric NOT NULL CHECK (percentage >= 0 AND percentage <= 100),
  color         text,
  sort_order    integer NOT NULL DEFAULT 0
);

-- 16. Arqueos de caja
CREATE TABLE cash_registers (
  id             uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id  uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  register_date  date NOT NULL DEFAULT CURRENT_DATE,
  opening_cash   numeric NOT NULL DEFAULT 0,
  closing_cash   numeric,
  sales_total    numeric,
  expenses_total numeric,
  difference     numeric,
  notes          text,
  created_at     timestamptz NOT NULL DEFAULT now()
);

-- 17. Movimientos de inventario (historial completo)
CREATE TABLE inventory_movements (
  id            uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  restaurant_id uuid NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
  ingredient_id uuid NOT NULL REFERENCES ingredients(id) ON DELETE CASCADE,
  type          text NOT NULL CHECK (type IN ('entrada','salida','ajuste','produccion')),
  quantity      numeric NOT NULL,
  notes         text,
  created_at    timestamptz NOT NULL DEFAULT now()
);
```

## PASO 3 — Trigger: crear restaurante automáticamente al registrarse

```sql
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.restaurants (name, owner_id)
  VALUES (
    COALESCE(NEW.raw_user_meta_data->>'restaurant_name', 'Mi Restaurante'),
    NEW.id
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

## PASO 4 — Trigger: registrar movimiento de inventario automáticamente

```sql
CREATE OR REPLACE FUNCTION log_inventory_movement()
RETURNS trigger AS $$
DECLARE
  v_restaurant_id uuid;
  v_type text;
  v_qty numeric;
BEGIN
  SELECT restaurant_id INTO v_restaurant_id FROM ingredients WHERE id = NEW.id;

  IF OLD.stock_current IS DISTINCT FROM NEW.stock_current THEN
    v_qty := NEW.stock_current - OLD.stock_current;
    IF v_qty > 0 THEN
      v_type := 'entrada';
    ELSIF v_qty < 0 THEN
      v_type := 'salida';
      v_qty := ABS(v_qty);
    ELSE
      RETURN NEW;
    END IF;

    INSERT INTO inventory_movements (restaurant_id, ingredient_id, type, quantity, notes)
    VALUES (v_restaurant_id, NEW.id, v_type, v_qty, 'Automático');
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_inventory_movement
  AFTER UPDATE OF stock_current ON ingredients
  FOR EACH ROW EXECUTE FUNCTION log_inventory_movement();
```

## PASO 5 — RLS: habilitar y crear políticas de aislamiento por tenant

```sql
-- Habilitar RLS en todas las tablas
ALTER TABLE restaurants         ENABLE ROW LEVEL SECURITY;
ALTER TABLE products            ENABLE ROW LEVEL SECURITY;
ALTER TABLE ingredients         ENABLE ROW LEVEL SECURITY;
ALTER TABLE recipes             ENABLE ROW LEVEL SECURITY;
ALTER TABLE recipe_ingredients  ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers           ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_methods     ENABLE ROW LEVEL SECURITY;
ALTER TABLE sales               ENABLE ROW LEVEL SECURITY;
ALTER TABLE sale_items          ENABLE ROW LEVEL SECURITY;
ALTER TABLE purchases           ENABLE ROW LEVEL SECURITY;
ALTER TABLE purchase_items      ENABLE ROW LEVEL SECURITY;
ALTER TABLE expenses            ENABLE ROW LEVEL SECURITY;
ALTER TABLE olaclick_imports    ENABLE ROW LEVEL SECURITY;
ALTER TABLE rappi_imports       ENABLE ROW LEVEL SECURITY;
ALTER TABLE income_distribution ENABLE ROW LEVEL SECURITY;
ALTER TABLE cash_registers      ENABLE ROW LEVEL SECURITY;
ALTER TABLE inventory_movements ENABLE ROW LEVEL SECURITY;

-- Función auxiliar reutilizable para obtener el restaurant_id del usuario actual
CREATE OR REPLACE FUNCTION my_restaurant_id()
RETURNS uuid AS $$
  SELECT id FROM restaurants WHERE owner_id = auth.uid() LIMIT 1;
$$ LANGUAGE sql STABLE SECURITY DEFINER;

-- Políticas: restaurants (solo el dueño)
CREATE POLICY "owner_only" ON restaurants
  USING (owner_id = auth.uid())
  WITH CHECK (owner_id = auth.uid());

-- Macro para las demás tablas: solo accede si restaurant_id es el del usuario
-- products
CREATE POLICY "tenant" ON products
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- ingredients
CREATE POLICY "tenant" ON ingredients
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- recipes
CREATE POLICY "tenant" ON recipes
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- recipe_ingredients (join a través de recipe)
CREATE POLICY "tenant" ON recipe_ingredients
  USING (recipe_id IN (SELECT id FROM recipes WHERE restaurant_id = my_restaurant_id()));

-- customers
CREATE POLICY "tenant" ON customers
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- payment_methods
CREATE POLICY "tenant" ON payment_methods
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- sales
CREATE POLICY "tenant" ON sales
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- sale_items (join a través de sale)
CREATE POLICY "tenant" ON sale_items
  USING (sale_id IN (SELECT id FROM sales WHERE restaurant_id = my_restaurant_id()));

-- purchases
CREATE POLICY "tenant" ON purchases
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- purchase_items
CREATE POLICY "tenant" ON purchase_items
  USING (purchase_id IN (SELECT id FROM purchases WHERE restaurant_id = my_restaurant_id()));

-- expenses
CREATE POLICY "tenant" ON expenses
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- olaclick_imports
CREATE POLICY "tenant" ON olaclick_imports
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- rappi_imports
CREATE POLICY "tenant" ON rappi_imports
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- income_distribution
CREATE POLICY "tenant" ON income_distribution
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- cash_registers
CREATE POLICY "tenant" ON cash_registers
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());

-- inventory_movements
CREATE POLICY "tenant" ON inventory_movements
  USING (restaurant_id = my_restaurant_id())
  WITH CHECK (restaurant_id = my_restaurant_id());
```

## PASO 6 — Datos iniciales de medios de pago

```sql
-- Se insertan después del primer registro de usuario (lo maneja el trigger).
-- Este script debe ejecutarse UNA VEZ por restaurante creado en desarrollo/seed.
-- En producción, el onboarding de usuario lo hará desde la app.

-- Ejemplo seed para desarrollo local:
DO $$
DECLARE v_rid uuid;
BEGIN
  SELECT id INTO v_rid FROM restaurants LIMIT 1;
  IF v_rid IS NOT NULL THEN
    INSERT INTO payment_methods (restaurant_id, name) VALUES
      (v_rid, 'Efectivo'),
      (v_rid, 'Transferencia'),
      (v_rid, 'Nequi'),
      (v_rid, 'Daviplata'),
      (v_rid, 'Tarjeta débito');
  END IF;
END $$;
```

Sin pausar. Sin preguntar. Ejecutar todo el SQL en el editor de Supabase en el orden dado.
