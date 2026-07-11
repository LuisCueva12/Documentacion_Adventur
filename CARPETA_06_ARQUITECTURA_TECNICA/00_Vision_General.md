# Arquitectura del Sistema Adventur Travel — Visión General

**Carpeta:** 06 — Arquitectura Técnica  
**Versión:** 1.0 — Documento de Implementación  
**Stack:** API REST (Laravel + PostgreSQL) + Frontend SPA (Vue 3 + Vite)

---

## 1. Principios Arquitectónicos

| # | Principio | Descripción |
|---|-----------|-------------|
| PA-01 | **Separación API / Frontend** | El backend expone una API REST pura. El frontend es una SPA (Single Page Application) que consume la API. Pueden vivir en servidores diferentes, escalar independientemente y cambiar de tecnología sin afectar al otro |
| PA-02 | **Arquitectura Hexagonal (Puertos y Adaptadores)** | El core del negocio (entidades, reglas, casos de uso) NO depende del framework ni de la infraestructura. Laravel es solo un adaptador de entrada (HTTP) y salida (BD, PDF, correo) |
| PA-03 | **Framework reemplazable** | La capa `Core/Dominio` + `Core/Aplicacion` no tiene UNA sola línea de Laravel. Si mañana cambias a Django, Express o Spring Boot, copias esas carpetas y reescribes solo `Infraestructura` e `Interfaces` |
| PA-04 | **API-first** | Todo lo que el sistema hace se expresa como endpoints REST. No hay lógica de negocio en controladores. Los controladores solo reciben la request, llaman a un Caso de Uso y devuelven la respuesta |
| PA-05 | **Cero acoplamiento tecnológico** | Base de datos, motor de PDF, servicio de correo, sistema de archivos — TODO es intercambiable gracias a interfaces (Puertos) en el dominio |
| PA-06 | **SPA sin SEO** | Sistema interno administrativo. No necesita SSR (Server Side Rendering). Vue 3 SPA se compila a archivos estáticos y se sirve con Nginx/Apache |

---

## 2. Diagrama de Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENTE (NAVEGADOR)                           │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    VUE 3 SPA (Single Page Application)            │  │
│  │                                                                   │  │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────────┐  │  │
│  │  │ Vistas  │→ │ Servicios│→ │ Pinia   │  │ PrimeVue (UI)    │  │  │
│  │  │ (Pages) │  │ (API     │  │ (Estado │  │ TanStack Query   │  │  │
│  │  │         │  │  Client) │  │  Global)│  │ (Caching)        │  │  │
│  │  └─────────┘  └──────────┘  └─────────┘  └──────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │  HTTPS / JSON (Bearer Token)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        LARAVEL API (BACKEND)                            │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  INTERFACES (Laravel HTTP)                                       │  │
│  │  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │  │
│  │  │Middleware  │→ │ Controladores│→ │ Form Requests /        │  │  │
│  │  │(Sanctum,   │  │ (Api)        │  │ API Resources          │  │  │
│  │  │ CORS, etc) │  │              │  │                        │  │  │
│  │  └────────────┘  └──────┬───────┘  └────────────────────────┘  │  │
│  └─────────────────────────┼──────────────────────────────────────┘  │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐  │
│  │  APLICACIÓN (Casos de Uso)                                     │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │ UseCase: CrearReserva, AplicarPago, GenerarPDF, etc.     │  │  │
│  │  │ → Orquestan el flujo, invocan repositorios, aplican      │  │  │
│  │  │   reglas de negocio, disparan eventos                     │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐  │
│  │  DOMINIO (Entidades, Value Objects, Reglas)                    │  │
│  │  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │  │
│  │  │ Entidades  │  │ ValueObjects │  │ Reglas de Negocio      │  │  │
│  │  │ (E01-E18)  │  │ (Precio,     │  │ (R1-R4, RF, NF)       │  │  │
│  │  │            │  │  Email, etc) │  │                        │  │  │
│  │  └────────────┘  └──────────────┘  └────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │ Repositorios (Interfaces - Puertos)                       │  │  │
│  │  │ → Define QUÉ hace, no CÓMO lo hace                       │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐  │
│  │  INFRAESTRUCTURA (Adaptadores)                                 │  │
│  │  ┌──────────────┐  ┌──────────┐  ┌────────┐  ┌─────────────┐  │  │
│  │  │ Persistencia │→│ Eloquent │  │ PDF    │  │ Mail        │  │  │
│  │  │ (PostgreSQL) │  │ Modelos  │  │(DomPDF)│  │(Laravel     │  │  │
│  │  │              │  │          │  │        │  │ Mail)       │  │  │
│  │  └──────────────┘  └──────────┘  └────────┘  └─────────────┘  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Organización de Carpetas (Backend)

```
adventur-api/
│
├── app/
│   │
│   ├── Core/                              ← CAPA PORTÁTIL (0 dependencias de Laravel)
│   │   ├── Dominio/
│   │   │   ├── Entidades/                 ← E01: Usuario, E02: Cliente, ... E18: ServicioReserva
│   │   │   ├── ObjetosValor/              ← Precio, Email, EstadoCotizacion, Rol, CodigoCorrelativo...
│   │   │   ├── Repositorios/              ← InterfazRepositorioUsuario, InterfazRepositorioReserva...
│   │   │   ├── Eventos/                   ← EventoReservaConfirmada, EventoCotizacionEnviada...
│   │   │   └── Reglas/                    ← ReglaR1SeparacionCotizacionReserva, ReglaR2Truncamiento...
│   │   │
│   │   └── Aplicacion/
│   │       ├── CasosUso/                  ← CrearCotizacionUseCase, AplicarPagoUseCase...
│   │       ├── DTOs/                      ← Data Transfer Objects (entrada/salida de casos de uso)
│   │       └── Excepciones/               ← ExcepcionNegocio, ExcepcionNoEncontrado...
│   │
│   ├── Infraestructura/                   ← ADAPTADORES (cambian según tecnología)
│   │   ├── Persistencia/
│   │   │   ├── Eloquent/                  ← Repositorios concretos que implementan las interfaces
│   │   │   └── Modelos/                   ← Modelos Eloquent (mapeo ORM)
│   │   ├── Pdf/                           ← ServicioPdf (implementación con DomPDF)
│   │   ├── Correo/                        ← ServicioCorreo (implementación con Laravel Mail)
│   │   ├── Seguridad/                     ← ServicioAutenticacion (Sanctum adapter)
│   │   └── Archivos/                      ← ServicioAlmacenamiento (disco local / S3)
│   │
│   ├── Http/                              ← LARAVEL (Interfaces de entrada)
│   │   ├── Controladores/
│   │   │   └── Api/                       ← Controladores REST
│   │   ├── Peticiones/                    ← Form Requests (validación)
│   │   └── Recursos/                      ← API Resources (transformación de respuestas)
│   │
│   ├── Proveedores/                       ← Service Providers de Laravel (inyección de dependencias)
│   └── Excepciones/                       ← Handler de excepciones
│
├── bootstrap/
├── config/
├── database/
│   ├── migraciones/                       ← Migraciones de BD
│   └── semillas/                          ← Seeders
├── routes/
│   └── api.php                            ← Rutas de la API
├── public/
├── resources/
├── storage/
└── tests/
```

---

## 4. Organización de Carpetas (Frontend)

```
adventur-frontend/
│
├── src/
│   ├── api/                               ← Cliente HTTP (axios)
│   │   ├── clienteHttp.js                 ← Instancia de axios configurada
│   │   ├── authApi.js                     ← Endpoints de autenticación
│   │   ├── cotizacionesApi.js             ← Endpoints de cotizaciones
│   │   ├── reservasApi.js                 ← Endpoints de reservas
│   │   ├── clientesApi.js                 ← Endpoints de clientes
│   │   └── ...                            ← Un archivo por módulo
│   │
│   ├── stores/                            ← Pinia (estado global)
│   │   ├── authStore.js                   ← Sesión, token, rol del usuario
│   │   ├── uiStore.js                     ← Sidebar, modales, tema
│   │   └── ...                            ← Stores específicos si se necesitan
│   │
│   ├── composables/                       ← Funciones reutilizables (Composition API)
│   │   ├── useAuth.js
│   │   ├── usePaginacion.js
│   │   ├── useExportar.js
│   │   └── ...
│   │
│   ├── componentes/                       ← Componentes reutilizables
│   │   ├── layout/
│   │   │   ├── AppLayout.vue
│   │   │   ├── Sidebar.vue
│   │   │   ├── Navbar.vue
│   │   │   └── Footer.vue
│   │   ├── ui/                            ← Componentes genéricos (botones, tablas, etc.)
│   │   │   ├── TablaGenerica.vue
│   │   │   ├── ModalConfirmacion.vue
│   │   │   ├── SelectorFecha.vue
│   │   │   └── ...
│   │   └── modulos/                       ← Componentes específicos del negocio
│   │       ├── cotizaciones/
│   │       │   ├── FormularioCotizacion.vue
│   │       │   ├── DetalleCotizacion.vue
│   │       │   └── ...
│   │       ├── reservas/
│   │       │   ├── Flujo5Pasos.vue
│   │       │   ├── Paso1Cliente.vue
│   │       │   ├── Paso2Viaje.vue
│   │       │   ├── Paso3Pasajeros.vue
│   │       │   ├── Paso4Pago.vue
│   │       │   └── Paso5Confirmacion.vue
│   │       └── ...
│   │
│   ├── vistas/                            ← Páginas (rutas)
│   │   ├── login/
│   │   │   └── LoginView.vue
│   │   ├── dashboard/
│   │   │   └── DashboardView.vue
│   │   ├── cotizaciones/
│   │   │   ├── CotizacionesListView.vue
│   │   │   ├── CotizacionDetalleView.vue
│   │   │   └── CotizacionNuevaView.vue
│   │   ├── reservas/
│   │   │   ├── ReservasListView.vue
│   │   │   ├── ReservaDetalleView.vue
│   │   │   └── ReservaNuevaView.vue
│   │   ├── clientes/
│   │   │   ├── ClientesListView.vue
│   │   │   └── ClienteDetalleView.vue
│   │   ├── reportes/
│   │   │   └── ReportesView.vue
│   │   └── admin/
│   │       ├── UsuariosView.vue
│   │       └── ConfiguracionView.vue
│   │
│   ├── router/
│   │   └── index.js                       ← Configuración de rutas Vue Router
│   │
│   └── App.vue                            ← Componente raíz
│
├── public/
├── index.html
├── vite.config.js
├── package.json
└── tailwind.config.js
```

---

## 5. Flujo de una Operación Típica (Ejemplo: Crear Reserva)

```
FRONTEND                              BACKEND (Laravel)                     BASE DE DATOS
────────                              ─────────────────                     ─────────────
1. Usuario llena                           │                                      │
   formulario Paso 1-5                     │                                      │
         │                                 │                                      │
2. Servicio API hace:                      │                                      │
   POST /api/reservas                      │                                      │
   {                                       │                                      │
     "id_cliente": 1,                      │                                      │
     "id_cotizacion": 5,                   │                                      │
     "pasajeros": [...],                   │                                      │
     "pagos": [...]                        │                                      │
   }                                       │                                      │
         │                                 │                                      │
         └────────────────────────────► 3. Pasa por Middleware:                    │
                                          ✔ Sanctum (token válido)               │
                                          ✔ CORS                                   │
                                          ✔ Rol (Vendedor o Admin)                │
                                              │                                    │
                                         4. FormRequest valida                    │
                                          datos de entrada                         │
                                              │                                    │
                                         5. Controlador llama a:                   │
                                          CrearReservaUseCase                      │
                                              │                                    │
                                         6. UseCase:                               │
                                          a) Valida reglas de negocio             │
                                            - R1: Cotización existe y          │
                                              no convertida aún                    │
                                            - R2: Precios truncados a 2 dec       │
                                            - R4: Soft delete check              │
                                          b) Crea entidad Reserva                 │
                                          c) Guarda via Repositorio ───────────► INSERT reservas
                                          d) Guarda pasajeros ───────────────────► INSERT pasajeros
                                          e) Guarda pagos ───────────────────────► INSERT pagos
                                          f) Dispara evento:                      │
                                            ReservaCreada                         │
                                              │                                    │
                                         7. Controlador                           │
                                          devuelve Response JSON                  │
                                              │                                    │
         ◄─────────────────────────────────┘                                      │
         │                                                                        │
8. TanStack Query caching                                                         │
   Pinia actualiza estado                                                         │
         │                                                                        │
9. Vue router navega a                                                             │
   /reservas/{id}                                                                  │
         │                                                                        │
10. Usuario ve reserva creada                                                     │
```

---

## 6. Convenciones Generales

### 6.1 Nomenclatura de Endpoints

```
Recurso          Endpoint                  Método   Propósito
────────         ────────                  ──────   ─────────
/auth            /api/auth/login           POST     Iniciar sesión
                 /api/auth/logout          POST     Cerrar sesión
                 /api/auth/me              GET      Obtener usuario actual

/cotizaciones    /api/cotizaciones         GET      Listar
                 /api/cotizaciones         POST     Crear
                 /api/cotizaciones/{id}    GET      Obtener
                 /api/cotizaciones/{id}    PUT      Actualizar
                 /api/cotizaciones/{id}    DELETE   Anular (soft delete)
                 /api/cotizaciones/{id}/enviar       POST    Enviar
                 /api/cotizaciones/{id}/aprobar      POST    Aprobar
                 /api/cotizaciones/{id}/rechazar     POST    Rechazar
                 /api/cotizaciones/{id}/perder       POST    Marcar como perdido
                 /api/cotizaciones/{id}/convertir    POST    Convertir a reserva

/reservas        /api/reservas              GET      Listar
                 /api/reservas              POST     Crear
                 /api/reservas/{id}         GET      Obtener
                 /api/reservas/{id}         PUT      Actualizar
                 /api/reservas/{id}         DELETE   Anular
                 /api/reservas/{id}/pagos   POST     Registrar pago
                 /api/reservas/{id}/pdf     GET      Generar PDF

/clientes        /api/clientes              GET      Listar
                 /api/clientes              POST     Crear
                 /api/clientes/{id}         GET      Obtener
                 /api/clientes/buscar       GET      Buscar (por dni, email, teléfono)

/reportes        /api/reportes/ventas       GET      Reporte de ventas
                 /api/reportes/conversion   GET      Reporte de conversión
                 /api/reportes/rendimiento  GET      Rendimiento por asesor
```

### 6.2 Estructura de Respuesta API

```json
{
  "exitoso": true,
  "datos": { ... },
  "mensaje": "Reserva creada correctamente",
  "errores": null
}
```

```json
{
  "exitoso": false,
  "datos": null,
  "mensaje": "Error de validación",
  "errores": {
    "id_cliente": ["El cliente no existe o está inactivo"],
    "pasajeros": ["Debe haber al menos 1 pasajero"]
  }
}
```

### 6.3 Paginación

```json
{
  "exitoso": true,
  "datos": [ ... ],
  "meta": {
    "pagina_actual": 1,
    "por_pagina": 15,
    "total": 234,
    "ultima_pagina": 16
  }
}
```

### 6.4 Códigos de Estado HTTP

| Código | Uso |
|--------|-----|
| 200 | OK - Éxito |
| 201 | Creado (POST) |
| 400 | Error de validación |
| 401 | No autenticado |
| 403 | No autorizado (rol sin permiso) |
| 404 | No encontrado |
| 409 | Conflicto (regla de negocio violada) |
| 422 | Datos inválidos |
| 500 | Error interno |

---

## 7. Decisiones Técnicas Clave

| Decisión | Opción Elegida | Alternativa Descartada | Motivo |
|----------|---------------|----------------------|--------|
| Frontend framework | Vue 3 + Vite | React, Angular | Curva baja, mismo poder, composición limpia |
| UI Components | PrimeVue | Element Plus, Naive UI | Más componentes, buen soporte, temas |
| HTTP Client | Axios + TanStack Query | fetch puro, SWR | Caching automático, refetch, estados loading/error |
| Estado global | Pinia | Vuex 4, solo provide/inject | Oficial, TypeScript, devtools |
| CSS | Tailwind CSS | Bootstrap, SCSS puro | Utility-first, rápido de prototipar, pequeño en prod |
| API Auth | Laravel Sanctum (tokens) | JWT manual, Passport | Nativo de Laravel, simple, seguro |
| Base de datos | PostgreSQL | MySQL, SQLite | Integridad referencial, RLS, ventanas, escalabilidad |
| PDF | DomPDF | Snappy, Browsershot | Nativo PHP, sin dependencias externas |
| Validación | Form Requests | Validación en controladores | Separación de concerns |
| Pruebas | PHPUnit (backend) + Vitest (frontend) | Pest, Jest | Ecosistema estándar |

---

## 8. Seguridad

| Aspecto | Implementación |
|---------|---------------|
| Autenticación | Sanctum token-based. Token se envía en header `Authorization: Bearer {token}` |
| Autorización | Policies de Laravel + Spatie Permission (matriz RBAC) |
| CORS | Configurado para aceptar solo el origen del frontend |
| Contraseñas | Hash con Bcrypt (nativo de Laravel) |
| SQL Injection | Eloquent ORM lo previene (prepared statements) |
| XSS | Vue 3 escapa HTML automáticamente. Laravel Blade escapa por defecto |
| CSRF | Sanctum SPA protection (solo si es necesario en forms) |
| Rate Limiting | Laravel throttle middleware |
| HTTPS | Obligatorio en producción. Redirect automático |

---

## 9. Escalabilidad (SiteGround → VPS)

### Fase 1 — SiteGround (Inicio)
```
Frontend: Archivos estáticos en public_html/
Backend:  Laravel en subdirectorio api/ o subdominio api.adventur.com
BD:       PostgreSQL en plan compartido de SiteGround
Colas:    Sync driver (síncrono, sin Redis)
Almacén:  Disco local (storage/app/public)
```

### Fase 2 — VPS (Crecimiento)
```
Frontend: Nginx sirviendo SPA estática
Backend:  PHP-FPM + Nginx (o Laravel Octane con RoadRunner)
BD:       PostgreSQL en el mismo VPS o RDS externo
Colas:    Redis + Laravel Horizon (asíncrono)
Almacén:  S3-Compatible (MinIO, DigitalOcean Spaces, AWS S3)
Cache:    Redis (caché de consultas frecuentes)
Horizont: Escalar: más workers de cola, más servidores de app
```

### Fase 3 — Multi-agencia (Futuro)
```
BD:       Esquema multi-tenancy con id_agencia como filtro
Backend:  Misma base, diferentes workers por agencia
Frontend: Mismo SPA, branding configurable por agencia
```

---

## 10. Stack Completo

```
┌─────────────────────────────────────────────────────────────┐
│                    STACK TECNOLÓGICO                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Backend:                                                   │
│    - PHP 8.4+                                               │
│    - Laravel 12+                                            │
│    - Laravel Sanctum (autenticación API)                    │
│    - Laravel Queues (colas asíncronas)                      │
│    - Laravel Horizon (monitoreo de colas)                   │
│    - PostgreSQL 16+                                         │
│    - DomPDF (generación de PDF)                             │
│                                                             │
│  Frontend:                                                  │
│    - Vue 3 (Composition API)                                │
│    - Vite (build tool)                                      │
│    - Vue Router 4 (enrutamiento SPA)                        │
│    - Pinia (estado global)                                  │
│    - PrimeVue 4 (componentes UI)                            │
│    - TanStack Query (caching de API)                        │
│    - Axios (cliente HTTP)                                   │
│    - Tailwind CSS (estilos)                                 │
│    - Chart.js + vue-chartjs (gráficos)                      │
│                                                             │
│  DevOps/Despliegue:                                         │
│    - SiteGround (fase inicial)                              │
│    - Nginx o Apache                                         │
│    - Git + GitHub/GitLab                                    │
│    - GitHub Actions (CI/CD)                                 │
│                                                             │
│  Calidad:                                                   │
│    - PHPUnit + Pest (backend)                               │
│    - Vitest + Vue Test Utils (frontend)                     │
│    - Larastan (análisis estático)                           │
│    - ESLint + Prettier (frontend)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
