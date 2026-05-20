# PROMPT 2 — Auth + Layout completo (EJECUTAR DESPUÉS DEL DB)

> El proyecto ya tiene Supabase con las 17 tablas y RLS. Ahora crea la app React desde cero.

---

Eres un arquitecto de software senior. Crea la estructura completa de **RestoApp** usando React 18 + Vite + Tailwind CSS + React Router v6 + Supabase Auth.

## Requisitos de autenticación

- Supabase Email/Password Auth (sin magic link, sin OAuth por ahora)
- Al registrarse, el usuario pasa `restaurant_name` en `options.data` para que el trigger de la DB lo use
- Guardar sesión en contexto global con `SessionContext`
- Rutas protegidas: si no hay sesión activa → redirigir a `/login`
- Al hacer logout → limpiar sesión y redirigir a `/login`

## Estructura de archivos a crear

```
restoapp/
├── index.html
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
├── package.json
└── src/
    ├── main.jsx
    ├── App.jsx                          ← Router raíz + SessionProvider
    ├── lib/
    │   └── supabase.js                  ← createClient con URL y anon key de .env
    ├── contexts/
    │   └── SessionContext.jsx           ← user, restaurantId, loading, logout
    ├── hooks/
    │   └── useRestaurant.js             ← shortcut: useContext(SessionContext)
    ├── pages/
    │   ├── Login.jsx                    ← Login + Registro en tabs
    │   ├── Dashboard.jsx
    │   ├── POS.jsx
    │   ├── Products.jsx
    │   ├── Inventory.jsx
    │   ├── Recipes.jsx
    │   ├── Purchases.jsx
    │   ├── Finance.jsx
    │   ├── Statistics.jsx
    │   ├── Distribution.jsx
    │   ├── ImportOlaClick.jsx
    │   ├── ImportRappi.jsx
    │   ├── CashRegister.jsx
    │   ├── Customers.jsx
    │   └── Settings.jsx
    └── components/
        ├── Layout.jsx                   ← Sidebar + topbar + Outlet
        ├── ProtectedRoute.jsx           ← Guard de sesión
        └── ui/
            ├── Button.jsx
            ├── Modal.jsx
            ├── Card.jsx
            ├── Badge.jsx
            └── Alert.jsx
```

## SessionContext.jsx — lógica exacta

```jsx
// Debe:
// 1. Llamar supabase.auth.getSession() al montar para restaurar sesión
// 2. Suscribirse a onAuthStateChange para detectar login/logout
// 3. Cuando hay sesión, hacer SELECT id FROM restaurants WHERE owner_id = user.id LIMIT 1
// 4. Exponer: { user, restaurantId, loading, logout }
// logout = () => supabase.auth.signOut()
```

## App.jsx — estructura de rutas

```jsx
// Rutas:
// /login                → <Login /> (pública)
// /                     → <ProtectedRoute> <Layout> <Outlet> → redirect a /dashboard
// /dashboard            → <Dashboard />
// /pos                  → <POS />
// /products             → <Products />
// /inventory            → <Inventory />
// /recipes              → <Recipes />
// /purchases            → <Purchases />
// /finance              → <Finance />
// /statistics           → <Statistics />
// /distribution         → <Distribution />
// /import-olaclick      → <ImportOlaClick />
// /import-rappi         → <ImportRappi />
// /cash-register        → <CashRegister />
// /customers            → <Customers />
// /settings             → <Settings />
```

## Login.jsx — dos tabs: Ingresar y Registrarse

```
Tab "Ingresar": email + password → supabase.auth.signInWithPassword()
Tab "Registrarse": nombre del restaurante + email + password → supabase.auth.signUp({
  email, password,
  options: { data: { restaurant_name: nombreRestaurante } }
})
Mostrar errores de Supabase inline. Loading state en botón.
```

## Layout.jsx — sidebar oscuro con badge de stock bajo

```
Sidebar (#1a1a2e, texto blanco):
  - Logo / nombre del restaurante (de restaurantId context)
  - Navegación:
    🏠 Dashboard           → /dashboard
    🛒 Punto de Venta      → /pos
    📦 Inventario          → /inventory  [badge rojo si hay ingredientes bajo mínimo]
    🍽️ Productos           → /products
    👨‍🍳 Recetas             → /recipes
    🧾 Compras             → /purchases
    💰 Finanzas            → /finance
    📊 Estadísticas        → /statistics
    🚀 Importar OlaClick   → /import-olaclick
    🏍️ Reporte Rappi       → /import-rappi
    🗃️ Arqueo de Caja      → /cash-register
    📈 Distribución        → /distribution
    👥 Clientes            → /customers
    ⚙️ Configuración       → /settings
  - Botón Cerrar sesión al fondo

Badge de stock bajo:
  Query: SELECT COUNT(*) FROM ingredients
         WHERE restaurant_id = $restaurantId
           AND stock_current <= stock_minimum
  Si count > 0 → mostrar círculo rojo con el número en el sidebar item de Inventario
  Refrescar cada 60 segundos con setInterval

Topbar: nombre de la página activa + nombre del usuario
Área principal: bg-gray-50, overflow-y-auto, <Outlet />
```

## Color scheme exacto

```
Sidebar bg:        #1a1a2e
Sidebar text:      #e8e8f0
Sidebar active:    #7c6dfa (violeta)
Sidebar hover:     rgba(124,109,250,0.12)
App bg:            #f8f9ff
Accent violeta:    #7c6dfa
Verde positivo:    #10b981
Rojo negativo:     #ef4444
Amarillo alerta:   #f59e0b
```

## Componentes UI mínimos

```
Button: variantes primary (violeta), secondary (gris), danger (rojo), ghost. Props: loading (spinner), disabled.
Modal: overlay oscuro, contenido centrado, prop onClose. Tamaños sm/md/lg.
Card: fondo blanco, borde sutil, sombra xs, padding 24px. Slot header y body.
Badge: variantes green/red/yellow/gray. Tamaño xs.
Alert: variantes info/success/warning/error con ícono y mensaje. Dismissable.
```

## Páginas placeholder (todas menos Login)

Cada página debe tener por ahora:
```jsx
export default function NombrePagina() {
  const { restaurantId } = useRestaurant();
  return (
    <div>
      <h1 className="text-2xl font-bold text-gray-900 mb-6">Nombre de la página</h1>
      <p className="text-gray-500">Módulo en construcción — restaurant_id: {restaurantId}</p>
    </div>
  );
}
```

## package.json — dependencias exactas

```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.0",
    "@supabase/supabase-js": "^2.45.0",
    "recharts": "^2.12.7",
    "xlsx": "^0.18.5",
    "jspdf": "^2.5.1",
    "jspdf-autotable": "^3.8.2",
    "lucide-react": "^0.414.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.40",
    "tailwindcss": "^3.4.9",
    "vite": "^5.4.0"
  }
}
```

## .env.local

```
VITE_SUPABASE_URL=TU_URL_AQUI
VITE_SUPABASE_ANON_KEY=TU_ANON_KEY_AQUI
```

Sin pausar. Sin preguntar. Crear todos los archivos completos y funcionales. Correr `npm install` al final.
