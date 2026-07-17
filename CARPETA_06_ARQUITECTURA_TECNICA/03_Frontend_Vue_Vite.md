# Frontend — Vue 3 + Vite (SPA)

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 03 — Frontend Vue + Vite

---

## 1. Stack del Frontend

| Componente | Tecnología | Propósito |
|-----------|-----------|-----------|
| Framework | Vue 3 (Composition API) | Base del SPA |
| Build Tool | Vite 6+ | Dev server rápido, builds optimizados |
| Routing | Vue Router 4 | Navegación SPA (hash o history mode) |
| Estado Global | Pinia | Store para sesión, UI, caché |
| UI Components | **PrimeVue 4** | Tablas, formularios, modales, menús, notificaciones |
| HTTP Client | Axios + **TanStack Query** | Peticiones HTTP, caching automático, refetch |
| Estilos | **Tailwind CSS** + PrimeVue temas | Utility-first, responsive |
| Gráficos | Chart.js + vue-chartjs | Dashboard, reportes |
| Exportación | xlsx (SheetJS) | Exportar tablas a Excel |
| Fechas | date-fns | Formateo, manipulación de fechas |
| Iconos | PrimeIcons + Lucide Vue | Iconos del sistema |
| Testing | Vitest + Vue Test Utils | Pruebas unitarias de componentes |

---

## 2. Estructura de Archivos

```
adventur-frontend/
│
├── index.html
├── package.json
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
│
├── public/
│   ├── favicon.ico
│   └── logo-adventur.svg
│
└── src/
    │
    ├── main.js                     ← Punto de entrada (crea app, monta router + pinia)
    ├── App.vue                     ← Componente raíz
    │
    ├── api/                        ← Capa de comunicación con backend
    │   ├── clienteHttp.js          ← Axios instance (baseURL, interceptors, token)
    │   ├── authApi.js              ← login(), logout(), obtenerPerfil()
    │   ├── cotizacionesApi.js      ← listar(), crear(), obtener(), enviar(), etc.
    │   ├── reservasApi.js          ← listar(), crear(), obtener(), anular(), pagar()
    │   ├── clientesApi.js          ← listar(), crear(), buscar()
    │   ├── leadsApi.js             ← CRUD leads
    │   ├── reportesApi.js          ← ventas(), conversion(), rendimiento()
    │   ├── dashboardApi.js         ← resumen()
    │   ├── adminApi.js             ← usuarios(), configuracion()
    │   └── catalogosApi.js         ← motivosPerdida(), canales(), plantillas(), rutas()
    │
    ├── stores/                     ← Pinia
    │   ├── authStore.js            ← usuario actual, token, rol, permisos
    │   ├── uiStore.js              ← sidebar abierto/cerrado, tema, notificaciones
    │   └── filtrosStore.js         ← Filtros persistentes entre vistas
    │
    ├── composables/                ← Funciones reutilizables (Composition API)
    │   ├── useAuth.js              ← Login/logout, verificar permisos
    │   ├── usePaginacion.js        ← Manejo de tabla paginada
    │   ├── useConfirmacion.js      ← Modal de confirmación genérico
    │   ├── useNotificacion.js      ← Toast/notificaciones
    │   ├── useExportar.js          ← Exportar tabla a Excel
    │   ├── useFormulario.js        ← Estados loading, error, éxito para forms
    │   └── usePermisos.js          ← Verificar si el rol puede X acción
    │
    ├── componentes/                ← Componentes reutilizables
    │   ├── layout/
    │   │   ├── AppLayout.vue       ← Layout principal (sidebar + navbar + content)
    │   │   ├── Sidebar.vue         ← Menú lateral (según rol)
    │   │   ├── AppNavbar.vue       ← Barra superior (usuario, notificaciones)
    │   │   └── AppFooter.vue       ← Pie de página
    │   │
    │   ├── ui/                     ← Componentes genéricos
    │   │   ├── TablaGenerica.vue   ← Tabla paginada, ordenable, exportable
    │   │   ├── ModalConfirmacion.vue
    │   │   ├── SelectorFecha.vue
    │   │   ├── InputBusqueda.vue   ← Búsqueda con debounce
    │   │   ├── BadgeEstado.vue     ← Badge de color según estado
    │   │   ├── AvatarUsuario.vue
    │   │   ├── TarjetaEstadistica.vue
    │   │   └── CargandoSpinner.vue
    │   │
    │   └── modulos/               ← Componentes específicos del negocio
    │       ├── cotizaciones/
    │       │   ├── FiltrosCotizacion.vue
    │       │   ├── TablaCotizaciones.vue
    │       │   ├── FormularioCotizacion.vue
    │       │   ├── DetalleCotizacion.vue
    │       │   ├── ItemsCotizacion.vue
    │       │   ├── HistorialEstados.vue
    │       │   ├── AccionesCotizacion.vue    ← Botones: enviar, aprobar, etc.
    │       │   └── ModalConvertirReserva.vue
    │       │
    │       ├── reservas/
    │       │   ├── FiltrosReserva.vue
    │       │   ├── TablaReservas.vue
    │       │   ├── Flujo5Pasos.vue           ← Contenedor del flujo
    │       │   ├── Paso1SelecciónCliente.vue
    │       │   ├── Paso2DatosViaje.vue
    │       │   ├── Paso3RegistroPasajeros.vue
    │       │   ├── Paso4RegistroPagos.vue
    │       │   ├── Paso5Confirmacion.vue
    │       │   ├── DetalleReserva.vue
    │       │   ├── ListaPasajeros.vue
    │       │   └── ListaPagos.vue
    │       │
    │       ├── clientes/
    │       │   ├── FormularioCliente.vue
    │       │   └── TablaClientes.vue
    │       │
    │       ├── dashboard/
    │       │   ├── TarjetaResumen.vue
    │       │   ├── ProximasSalidas.vue
    │       │   ├── GraficoVentas.vue
    │       │   ├── AlertasPanel.vue
    │       │   └── CotizacionesRecientes.vue
    │       │
    │       └── admin/
    │           ├── TablaUsuarios.vue
    │           ├── FormularioUsuario.vue
    │           ├── EditorPoliticas.vue
    │           └── CatalogosEditor.vue
    │
    ├── vistas/                    ← Páginas (una por ruta)
    │   ├── login/
    │   │   └── LoginView.vue
    │   ├── dashboard/
    │   │   └── DashboardView.vue
    │   ├── cotizaciones/
    │   │   ├── CotizacionesListaView.vue
    │   │   ├── CotizacionDetalleView.vue
    │   │   └── CotizacionNuevaView.vue
    │   ├── reservas/
    │   │   ├── ReservasListaView.vue
    │   │   ├── ReservaDetalleView.vue
    │   │   └── ReservaNuevaView.vue
    │   ├── clientes/
    │   │   ├── ClientesListaView.vue
    │   │   └── ClienteDetalleView.vue
    │   ├── leads/
    │   │   ├── LeadsListaView.vue
    │   │   └── LeadDetalleView.vue
    │   ├── reportes/
    │   │   └── ReportesView.vue
    │   └── admin/
    │       ├── UsuariosView.vue
    │       ├── PoliticasView.vue
    │       └── CatalogosView.vue
    │
    ├── router/
    │   ├── index.js               ← Configuración de Vue Router
    │   └── guardias.js            ← Guardias: auth, rol
    │
    └── utilerias/
        ├── formatos.js            ← formatear moneda, fecha, documento
        ├── constantes.js          ← Estados, roles, colores
        └── validaciones.js        ← Validaciones del lado cliente
```

---

## 3. Flujo de Autenticación (Frontend)

```
1. Usuario visita /login
2. Ingresa email + password
3. authStore.login() llama a POST /api/auth/login
4. Backend valida y devuelve token + usuario
5. authStore guarda token en localStorage/Pinia
6. Axios interceptor agrega header Authorization a todas las requests
7. Router redirige a /dashboard
8. Si token expira (401), authStore.logout() y redirige a /login
```

### Interceptor Axios

```javascript
// api/clienteHttp.js
import axios from 'axios'

const clienteHttp = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api',
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
})

// Interceptor: agrega token automáticamente
clienteHttp.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Interceptor: manejo global de errores
clienteHttp.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default clienteHttp
```

---

## 4. Enrutamiento (Vue Router)

```javascript
// router/index.js
const rutas = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/vistas/login/LoginView.vue'),
    meta: { requiereAuth: false },
  },
  {
    path: '/',
    component: () => import('@/componentes/layout/AppLayout.vue'),
    meta: { requiereAuth: true },
    children: [
      {
        path: '',
        redirect: '/dashboard',
      },
      {
        path: 'dashboard',
        name: 'Dashboard',
        component: () => import('@/vistas/dashboard/DashboardView.vue'),
        meta: { titulo: 'Dashboard', icono: 'pi pi-home' },
      },
      {
        path: 'cotizaciones',
        name: 'CotizacionesLista',
        component: () => import('@/vistas/cotizaciones/CotizacionesListaView.vue'),
        meta: { titulo: 'Cotizaciones', icono: 'pi pi-file', roles: ['Admin', 'Vendedor', 'Supervisor', 'Gerencia'] },
      },
      {
        path: 'cotizaciones/nueva',
        name: 'CotizacionNueva',
        component: () => import('@/vistas/cotizaciones/CotizacionNuevaView.vue'),
        meta: { titulo: 'Nueva Cotización', roles: ['Admin', 'Vendedor', 'Supervisor', 'Gerencia'] },
      },
      {
        path: 'cotizaciones/:id',
        name: 'CotizacionDetalle',
        component: () => import('@/vistas/cotizaciones/CotizacionDetalleView.vue'),
        meta: { titulo: 'Detalle Cotización' },
      },
      {
        path: 'reservas',
        name: 'ReservasLista',
        component: () => import('@/vistas/reservas/ReservasListaView.vue'),
        meta: { titulo: 'Reservas', icono: 'pi pi-check-circle' },
      },
      {
        path: 'reservas/nueva',
        name: 'ReservaNueva',
        component: () => import('@/vistas/reservas/ReservaNuevaView.vue'),
        meta: { titulo: 'Nueva Reserva' },
      },
      {
        path: 'reservas/:id',
        name: 'ReservaDetalle',
        component: () => import('@/vistas/reservas/ReservaDetalleView.vue'),
        meta: { titulo: 'Detalle Reserva' },
      },
      {
        path: 'clientes',
        name: 'ClientesLista',
        component: () => import('@/vistas/clientes/ClientesListaView.vue'),
        meta: { titulo: 'Clientes', icono: 'pi pi-users' },
      },
      {
        path: 'reportes',
        name: 'Reportes',
        component: () => import('@/vistas/reportes/ReportesView.vue'),
        meta: { titulo: 'Reportes', icono: 'pi pi-chart-bar' },
      },
      {
        path: 'admin/usuarios',
        name: 'AdminUsuarios',
        component: () => import('@/vistas/admin/UsuariosView.vue'),
        meta: { titulo: 'Usuarios', roles: ['Admin'] },
      },
      {
        path: 'admin/politicas',
        name: 'AdminPoliticasDocumento',
        component: () => import('@/vistas/admin/PoliticasDocumentoView.vue'),
        meta: { titulo: 'Políticas del PDF', permiso: 'politicas' },
      },
    ],
  },
]
```

### Guardias (Navigation Guards)

```javascript
// router/guardias.js

// Antes de cada ruta:
// 1. ¿Requiere auth? → verificar token en store
// 2. ¿Requiere rol específico? → verificar rol del usuario
// 3. Si no cumple → redirigir a login o dashboard

router.beforeEach((to, from, next) => {
  const authStore = useAuthStore()

  if (to.meta.requiereAuth && !authStore.estaAutenticado) {
    return next({ name: 'Login' })
  }

  if (to.meta.roles && !to.meta.roles.includes(authStore.rol)) {
    return next({ name: 'Dashboard' })
  }

  next()
})
```

---

## 5. Manejo de Estado con Pinia

### authStore.js

```javascript
import { defineStore } from 'pinia'
import { authApi } from '@/api/authApi'

export const useAuthStore = defineStore('auth', {
  state: () => ({
    usuario: null,
    token: localStorage.getItem('token'),
    rol: null,
  }),

  getters: {
    estaAutenticado: (state) => !!state.token,
    esAdmin: (state) => state.rol === 'Administrador',
    esVendedor: (state) => state.rol === 'Vendedor',
    nivelAcceso: (state) => {
      const niveles = { Administrador: 100, Gerencia: 80, Supervisor: 60, Contabilidad: 50, Operaciones: 40, Vendedor: 20 }
      return niveles[state.rol] || 0
    },
  },

  actions: {
    async login(email, password) {
      const respuesta = await authApi.login(email, password)
      this.token = respuesta.datos.token
      this.usuario = respuesta.datos.usuario
      this.rol = respuesta.datos.usuario.rol
      localStorage.setItem('token', this.token)
    },

    async logout() {
      await authApi.logout()
      this.token = null
      this.usuario = null
      this.rol = null
      localStorage.removeItem('token')
    },

    async obtenerPerfil() {
      const respuesta = await authApi.me()
      this.usuario = respuesta.datos
      this.rol = respuesta.datos.rol
    },
  },
})
```

---

## 6. TanStack Query (Caching de API)

```javascript
// En un componente
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query'
import { cotizacionesApi } from '@/api/cotizacionesApi'

// OBTENER lista (con caché automática)
const { datos, estaCargando, error } = useQuery({
  queryKey: ['cotizaciones', filtros],
  queryFn: () => cotizacionesApi.listar(filtros),
  keepPreviousData: true,  // mantiene datos anteriores mientras carga nueva página
  staleTime: 30000,       // 30 segundos antes de refetch
})

// MUTACIÓN (crear, actualizar, eliminar)
const queryClient = useQueryClient()

const mutation = useMutation({
  mutationFn: (nuevaCotizacion) => cotizacionesApi.crear(nuevaCotizacion),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['cotizaciones'] })
    // muestra notificación, redirige
  },
  onError: (error) => {
    // muestra error
  },
})
```

---

## 7. PrimeVue — Componentes Principales

| Componente PrimeVue | Uso en el sistema |
|--------------------|-------------------|
| `DataTable` | Tablas de cotizaciones, reservas, clientes, usuarios |
| `Paginator` | Paginación de tablas |
| `InputText` | Campos de texto |
| `InputNumber` | Campos numéricos (montos, cantidades) |
| `InputMask` | DNI (99999999), RUC (99999999999), teléfono |
| `Calendar` | Selector de fechas |
| `Dropdown` | Selectores (estados, roles, métodos de pago) |
| `MultiSelect` | Filtros múltiples |
| `Dialog` | Modales (confirmación, detalle rápido) |
| `ConfirmDialog` | Confirmación de acciones (anular, eliminar) |
| `Toast` | Notificaciones |
| `Menubar` / `PanelMenu` | Menú lateral y barra superior |
| `Card` | Tarjetas del dashboard |
| `Chart` | Gráficos (Chart.js envuelto) |
| `Tag` / `Badge` | Badges de estado (colores según estado) |
| `Stepper` | Flujo de 5 pasos para crear reserva |
| `FileUpload` | Subida de comprobantes de pago |
| `ProgressSpinner` | Cargando |
| `InputTextarea` | Notas, observaciones |
| `TabView` | Pestañas en detalle de reserva |
| `Toolbar` | Barra de acciones en tablas |

---

## 8. Manejo de Errores (Frontend)

```
TanStack Query / Axios
  │
  ▼
Error detectado
  │
  ├─ 401 → authStore.logout() → redirigir a /login
  ├─ 403 → Mostrar mensaje "No tienes permiso"
  ├─ 404 → Mostrar "No encontrado"
  ├─ 409 → Mostrar mensaje del backend (violación regla negocio)
  ├─ 422 → Mostrar errores de validación en formulario
  └─ 500 → Mostrar "Error del servidor. Intenta nuevamente"
```

```javascript
// composables/useNotificacion.js
export function useNotificacion() {
  const toast = useToast()

  function mostrarError(error) {
    const mensaje = error.response?.data?.mensaje || 'Error inesperado'
    const codigo = error.response?.status

    toast.add({
      severity: 'error',
      summary: `Error ${codigo}`,
      detail: mensaje,
      life: 5000,
    })
  }

  function mostrarExito(mensaje) {
    toast.add({
      severity: 'success',
      summary: 'Éxito',
      detail: mensaje,
      life: 3000,
    })
  }

  return { mostrarError, mostrarExito }
}
```

---

## 9. Responsive Design

| Dispositivo | Ancho | Comportamiento |
|-------------|-------|---------------|
| Desktop | ≥ 1024px | Layout completo: sidebar + navbar + contenido |
| Tablet | 768px - 1023px | Sidebar colapsable, navbar compacto |
| Celular | < 768px | Sidebar en overlay, tabla horizontal scroll, formularios apilados |

PrimeVue maneja responsive automáticamente en DataTable y formularios. Tailwind CSS para ajustes finos.

---

## 10. Seguridad (Frontend)

| Aspecto | Implementación |
|---------|---------------|
| Token | Almacenado en localStorage. Interceptor en cada request |
| Rotación | Login nuevo invalida token anterior (Sanctum) |
| XSS | Vue 3 escapa todo el HTML por defecto |
| Rutas protegidas | Vue Router guards + verificación de rol |
| Botones/acciones ocultos | `v-if="store.esAdmin"` en componentes |
| Logout | Borra token local + invalida en backend |
| Timeout sesión | 30 min sin actividad → redirigir a login (interceptor detecta 401) |

---

## 11. Build y Despliegue

### Desarrollo

```bash
pnpm run dev    # → http://localhost:5173
```

Proxy automático de `/api` a Laravel:

```javascript
// vite.config.js
export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': {
        target: 'http://adventur-api.test',
        changeOrigin: true,
      },
    },
  },
})
```

### Producción

```bash
pnpm run build    # → dist/ (archivos estáticos)

# En SiteGround: copiar dist/ a public_html/
# En VPS: Nginx apunta a dist/
```

```nginx
# Nginx (VPS)
server {
    listen 80;
    server_name app.adventur.com;
    root /var/www/adventur-frontend/dist;

    location / {
        try_files $uri $uri/ /index.html;  # SPA fallback
    }

    location /api {
        proxy_pass http://localhost:8000;   # Laravel
    }
}
```
