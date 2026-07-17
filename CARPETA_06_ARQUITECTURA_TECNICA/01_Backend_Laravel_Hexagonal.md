# Backend — Laravel con Arquitectura Hexagonal

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 01 — Backend Laravel + Hexagonal

---

## 1. Principio de Organización

Laravel NO se modifica. La arquitectura hexagonal convive dentro de Laravel usando sus mecanismos nativos:

- **Service Providers** → para inyectar las dependencias (bind interfaces → implementations)
- **Autoloading PSR-4** → namespaces personalizados funcionan sin tocar nada
- **Artisan** → comandos personalizados sin modificar el core

```
app/
├── Core/                      ← Namespace: App\Core\ (PSR-4 automático)
├── Infraestructura/           ← Namespace: App\Infraestructura\
├── Http/                      ← Namespace: App\Http\ (Laravel nativo)
├── Providers/                 ← Service Providers nativos
└── ... (Laravel sigue intacto)
```

---

## 2. Diagrama de Capas

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERFACES DE ENTRADA                        │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ Middleware  │  │ Controllers  │  │ Form Requests          │  │
│  │ Sanctum     │  │ Api/*        │  │ (Validación)           │  │
│  │ CORS        │  │              │  │                        │  │
│  │ Throttle    │  │              │  │                        │  │
│  └────────────┘  └──────┬───────┘  └────────────────────────┘  │
├──────────────────────────┼──────────────────────────────────────┤
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              CAPA DE APLICACIÓN (Use Cases)               │  │
│  │                                                          │  │
│  │  No sabe qué framework lo llama. Recibe DTO,             │  │
│  │  ejecuta lógica, devuelve resultado.                     │  │
│  │                                                          │  │
│  │  Ej: CrearCotizacionUseCase                              │  │
│  │    - Valida reglas de negocio                            │  │
│  │    - Usa repositorios (vía interfaz)                     │  │
│  │    - Dispara eventos de dominio                          │  │
│  │    - Devuelve DTO de respuesta                           │  │
│  └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                          │                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              CAPA DE DOMINIO                              │  │
│  │                                                          │  │
│  │  Lo único que importa. 0 dependencias externas.          │  │
│  │                                                          │  │
│  │  Entidades  │  Value Objects  │  Reglas  │  Eventos      │  │
│  │  (E01-E18)  │  (Precio, Rol,  │  (R1-R4) │  (Dominio)    │  │
│  │             │   Estado, etc)  │          │               │  │
│  │                                                          │  │
│  │  Repositorios (Interfaces)                               │  │
│  │  ┌──────────────────────────────────────────────────┐    │  │
│  │  │ InterfazRepositorioCotizacion {                   │    │  │
│  │  │   obtenerPorId(int): ?Cotizacion                 │    │  │
│  │  │   guardar(Cotizacion): void                      │    │  │
│  │  │   eliminar(int): void                            │    │  │
│  │  │ }                                                │    │  │
│  │  └──────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                          │                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              INFRAESTRUCTURA (Adaptadores)                │  │
│  │                                                          │  │
│  │  Implementa las interfaces del dominio.                  │  │
│  │  Solo aquí hay código de Laravel/Eloquent/PostgreSQL.    │  │
│  │                                                          │  │
│  │  Ej: RepositorioCotizacionEloquent implements            │  │
│  │        InterfazRepositorioCotizacion                     │  │
│  │      → Usa Modelo Eloquent internamente                  │  │
│  │      → Convierte Model → Entidad y viceversa            │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.1 Regla de Oro: el Core no conoce a Laravel/Eloquent

Para que la arquitectura quede **bien definida** y pueda **crecer o acoplar otros sistemas**
(POS, WooCommerce, otra BD, microservicios) sin romperse, se aplica una única regla
estricta de aislamiento del Dominio y la Aplicación:

> **`app/Core` NUNCA importa `App\Infraestructura`, ni `DB`/`Schema`, ni Modelos Eloquent.**
> Toda dependencia externa entra por **puerto (interfaz)** y se resuelve en Infraestructura.

### Qué está permitido / prohibido en `app/Core`

| Situación | Estado | Dónde vive |
|-----------|--------|-----------|
| `use App\Infraestructura\...` | ❌ Prohibido | — |
| `use Illuminate\Support\Facades\DB` / `Schema` | ❌ Prohibido | Solo en `app/Infraestructura` |
| `extends Model` (Eloquent) | ❌ Prohibido | Solo en `app/Infraestructura/Persistencia/Modelos` |
| `use Illuminate\Support\Facades\Event` | ✅ Aceptado | Dispatcher de eventos de dominio |
| `new DatosEmpresa()` leyendo `config()` | ❌ Prohibido | El dato entra por constructor desde el adaptador |

**Comando de auditoría (debe dar 0 resultados):**

```bash
grep -rn "use App\\Infraestructura" app/Core   # -> 0
grep -rn "use Illuminate\\Support\\Facades\\\(DB\|Schema\)" app/Core  # -> 0
```

### Centralización de dependencias

- **Todos los `bind(Interfaz::class, Implementacion::class)` viven en**
  `app/Providers/CoreServiceProvider.php` (método `register`).
- `app/Providers/AppServiceProvider.php` queda reservado a configuración de framework
  (p.ej. forzar HTTPS en producción), no a bindings de negocio.
- Gracias a esta regla, los Casos de Uso son 100% portables y testeables con mocks.

---

## 3. Flujo de una Request (detallado)

```
1. Navegador → GET /api/cotizaciones
                                      │
2. routes/api.php                     │
   Route::apiResource('cotizaciones', │
     [CotizacionControlador::class]); │
                                      │
3. Middleware (Kernel)                 │
   - sanctum: verifica token Bearer   │
   - cors: headers permitidos         │
   - throttle: 60 requests/minuto     │
                                      │
4. CotizacionControlador@index        │
   - Inyecta: ListarCotizacionesUseCase (por DI)
   - Crea: ListarCotizacionesDTO      │
   - Llama: $useCase->ejecutar($dto)  │
                                      │
5. ListarCotizacionesUseCase           │
   - Llama: $repositorio->listar(...) │
     (usa interfaz, no sabe qué BD)   │
   - Convierte a DTOs de respuesta    │
   - Devuelve: ResultadoDTO           │
                                      │
6. Controlador                         │
   - Envuelve en API Resource         │
   - Devuelve: response()->json()     │
                                      │
7. Navegador ← JSON Response           │
```

### 3.1 Patrón aplicado al módulo administrativo

La administración se divide por responsabilidad y no utiliza un controlador monolítico:

```text
FormRequest → DTO inmutable → Caso de uso → Puerto → Adaptador Eloquent
                    └──────── ServicioAuditoria + UnidadDeTrabajo ────────┘
```

- `UsuarioAdministracionControlador`, `RolAdministracionControlador` y `PoliticaDocumentoControlador` no importan modelos, `DB` ni clases de Infraestructura.
- Cada escritura tiene un caso de uso independiente; la validación sintáctica vive en `FormRequest` y las reglas cruzadas se garantizan nuevamente en Aplicación.
- El hash de contraseña pertenece al adaptador de persistencia y nunca se expone en DTOs de salida ni auditoría.
- Usuarios y políticas se auditan mediante `ServicioAuditoria` dentro de la misma unidad de trabajo.
- Las consultas administrativas usan puertos de Aplicación porque devuelven proyecciones paginadas, no agregados completos del Dominio.

---

## 4. Contratos (Interfaces) del Dominio

### 4.1 Repositorio Base (concepto)

A modo de referencia, todo repositorio de dominio sigue este contrato mínimo (en la
práctica cada `InterfazRepositorio*` declara sus propios métodos tipados con la entidad
correspondiente, no hay una interfaz genérica compartida):

```php
interface InterfazRepositorioBase
{
    public function guardar(object $entidad): void;
    public function eliminar(int $id): void;
    public function obtenerPorId(int $id): ?object;
}
```

### 4.2 Puertos de Repositorio (persistencia)

Cada entidad de dominio tiene su interfaz en `Core/Dominio/Repositorios/`. Ejemplo real
de `InterfazRepositorioPago`:

```php
interface InterfazRepositorioPago
{
    public function guardar(Pago $pago): void;
    public function obtenerPorId(int $id): ?Pago;
    public function listarPorReserva(int $idReserva): array;   // devuelve Pago[]
    public function anular(Pago $pago): void;
    public function sumarPagosReserva(int $idReserva): Precio;
}
```

**Lista real de puertos de repositorio** (`app/Core/Dominio/Repositorios/`):

| Puerto | Implementación Eloquent |
|--------|-------------------------|
| `InterfazRepositorioCotizacion` | `RepositorioCotizacionEloquent` |
| `InterfazRepositorioReserva` | `RepositorioReservaEloquent` |
| `InterfazRepositorioCliente` | `RepositorioClienteEloquent` |
| `InterfazRepositorioUsuario` | `RepositorioUsuarioEloquent` |
| `InterfazRepositorioPago` | `RepositorioPagoEloquent` |
| `InterfazRepositorioPasajero` | `RepositorioPasajeroEloquent` |
| `InterfazRepositorioLead` | `RepositorioLeadEloquent` |
| `InterfazRepositorioSeguimiento` | `RepositorioSeguimientoEloquent` |
| `InterfazRepositorioPlantilla` | `RepositorioPlantillaEloquent` |
| `InterfazRepositorioPaquete` | `RepositorioPaqueteEloquent` |
| `InterfazRepositorioAlojamiento` | `RepositorioAlojamientoEloquent` |
| `InterfazRepositorioHotel` | `RepositorioHotelEloquent` |
| `InterfazRepositorioMovilidad` | `RepositorioMovilidadEloquent` |
| `InterfazRepositorioServicioReserva` | `RepositorioServicioReservaEloquent` |
| `InterfazRepositorioMotivoPerdida` | `RepositorioMotivoPerdidaEloquent` |
| `InterfazRepositorioAuditoria` | `RepositorioAuditoriaEloquent` |
| `InterfazRepositorioSecuenciaCodigo` | `RepositorioSecuenciaCodigoEloquent` |
| `InterfazRepositorioItemCotizacion` | `RepositorioItemCotizacionEloquent` |
| `InterfazRepositorioPoliticaConfiguracion` | `RepositorioPoliticaConfiguracionEloquent` |

> **Nota:** `cotizacion_detalle` y `politicas_configuracion` no tienen entidad de dominio
> propia; se exponen vía DTO de aplicación (`ItemCotizacionDto`) o entidad
> (`PoliticaConfiguracion`) a través de sus respectivos puertos, sin que el Core toque Eloquent.

### 4.3 Puertos de Fuente Externa (Anti-Corrupción)

Para **acoplar varios sistemas** (WooCommerce, APIs de hoteles/movilidades) sin que el
dominio aprenda sus formatos, existen puertos de entrada/salida externa resueltos por
adaptadores HTTP en `Infraestructura/Adaptadores/`:

| Puerto | Adaptador | Origen |
|--------|-----------|--------|
| `InterfazFuentePaquetes` | `AdaptadorWooCommerce` | WooCommerce (REST) |
| `InterfazFuenteHoteles` | `AdaptadorHotelHttp` | API de hoteles (HTTP) |
| `InterfazFuenteMovilidades` | `AdaptadorMovilidadHttp` | API de movilidades (HTTP) |

### 4.4 Puertos de Servicio (salida / infraestructura)

Casos de uso que necesitan una capacidad externa (PDF, transacción) dependen de estos
puertos en `Core/Aplicacion/Puertos/`:

| Puerto | Implementación | Responsabilidad |
|--------|----------------|-----------------|
| `InterfazGeneradorDocumentoPdf` | `GeneradorDocumentoPdfDomPdf` | Renderizar `DocumentoPdfDto` a binario PDF |
| `InterfazUnidadDeTrabajo` | `UnidadDeTrabajoEloquent` | Envolver operaciones en `DB::transaction` |
| `InterfazRepositorioUsuarioAdministracion` | `RepositorioUsuarioAdministracionEloquent` | Proyección y comandos administrativos de usuarios |
| `InterfazRepositorioRolAdministracion` | `RepositorioRolAdministracionEloquent` | Consulta desacoplada del catálogo de roles |

```php
// Core/Aplicacion/Puertos/InterfazGeneradorDocumentoPdf.php
interface InterfazGeneradorDocumentoPdf
{
    public function generar(DocumentoPdfDto $datos): string;  // devuelve binario PDF
}

// Core/Aplicacion/Puertos/InterfazUnidadDeTrabajo.php
interface InterfazUnidadDeTrabajo
{
    /** @template T  @param callable(): T $operacion  @return T */
    public function transaccion(callable $operacion): mixed;
}
```

> **`DatosEmpresa` (Value Object del dominio) no lee `config()`.** El adaptador
> `GeneradorDocumentoPdfDomPdf` (Infraestructura) lee `config('empresa')` y construye
> `DatosEmpresa::desdeArray($config)`, manteniendo el Core libre de facades de Laravel.

---

## 5. Casos de Uso (Application Layer)

Cada caso de uso es una clase con un solo método público `ejecutar()`.

### 5.1 Módulo Cotizaciones

| Caso de Uso | Entrada | Salida | Reglas que valida |
|-------------|---------|--------|-------------------|
| `CrearCotizacionUseCase` | id_cliente, items, id_vendedor | CotizacionDTO | R1 (no crear cotización si ya existe una activa sin convertir) |
| `EnviarCotizacionUseCase` | id_cotizacion | void | Estado debe ser Borrador |
| `AprobarCotizacionUseCase` | id_cotizacion, descuento | void | Estado debe ser Visto, descuento según rol |
| `RechazarCotizacionUseCase` | id_cotizacion, motivo | void | Estado debe ser Enviado o Visto |
| `PerderCotizacionUseCase` | id_cotizacion, id_motivo, observacion | void | Estado Visto o Aprobado |
| `ConvertirCotizacionAReservaUseCase` | id_cotizacion | ReservaDTO | R1, estado debe ser Aprobado |
| `ListarCotizacionesUseCase` | filtros, pagina | PaginacionDTO | — |
| `ObtenerCotizacionUseCase` | id | CotizacionDTO | — |

### 5.2 Módulo Reservas

| Caso de Uso | Entrada | Salida | Reglas |
|-------------|---------|--------|--------|
| `CrearReservaUseCase` | datos reserva, pasajeros, pagos | ReservaDTO | R1, R2, R4 |
| `AnularReservaUseCase` | id, motivo | void | R4 |
| `RegistrarPagoUseCase` | id_reserva, monto, metodo | PagoDTO | R2, R4 |
| `AnularPagoUseCase` | id_pago | void | R4 |
| `GenerarPdfReservaUseCase` | id_reserva | PDF (binario) | — |
| `GenerarPdfCotizacionUseCase` | id_cotizacion | PDF (binario) | — |
| `ListarReservasUseCase` | filtros, pagina | PaginacionDTO | — |
| `ObtenerReservaUseCase` | id | ReservaDTO | — |

### 5.3 Módulo Clientes

| Caso de Uso | Entrada | Salida |
|-------------|---------|--------|
| `CrearClienteUseCase` | datos cliente | ClienteDTO |
| `BuscarClienteUseCase` | termino | array ClienteDTO |
| `ObtenerClienteUseCase` | id | ClienteDTO |

### 5.4 Módulo Global

| Caso de Uso | Entrada | Salida | Reglas |
|-------------|---------|--------|--------|
| `GenerarSecuenciaCodigoUseCase` | prefijo, año | CodigoCorrelativo | Secuencia única y correlativa |
| `GenerarReporteVentasUseCase` | fecha_inicio, fecha_fin, id_vendedor | ReporteDTO | — |
| `GenerarReporteConversionUseCase` | fecha_inicio, fecha_fin | ReporteDTO | — |
| `RegistrarAuditoriaUseCase` | accion, entidad, id_entidad, valores | void | R3 (solo INSERT) |
| `AutenticarUsuarioUseCase` | email, contraseña | AuthDTO (token + usuario) | — |

---

## 6. Mapeo Modelo Eloquent → Entidad de Dominio

La infraestructura es responsable de convertir entre el mundo Eloquent y el mundo Dominio.

### Ejemplo: RepositorioCotizacionEloquent

```
┌─────────────────────────────────────────────┐
│         RepositorioCotizacionEloquent        │
│                                             │
│  implementa: InterfazRepositorioCotizacion  │
│                                             │
│  guardar(Cotizacion $c): void               │
│    → $modelo = new ModeloCotizacion()       │
│    → $modelo->fill($c->toArray())           │
│    → $modelo->save()                        │
│    → $c->setId($modelo->id)                 │
│                                             │
│  obtenerPorId(int $id): ?Cotizacion          │
│    → $modelo = ModeloCotizacion::find($id)  │
│    → if (!$modelo) return null              │
│    → return $this->toEntidad($modelo)       │
│                                             │
│  toEntidad(ModeloCotizacion $m): Cotizacion  │
│    → reconstruye entidad desde datos BD     │
│    → new Cotizacion(                        │
│        id: $m->id,                          │
│        codigo: new CodigoCorrelativo(...),  │
│        estado: new EstadoCotizacion(...),   │
│        ...                                   │
│      )                                      │
└─────────────────────────────────────────────┘
```

### Patrón: Data Mapper (no Active Record para el dominio)

| Concepto | Implementación |
|----------|---------------|
| Modelo Eloquent | `ModeloCotizacion extends Model` — solo mapea tabla `cotizaciones` |
| Entidad Dominio | `Cotizacion` — clase plana con comportamiento, sin herencia |
| Repositorio | `RepositorioCotizacionEloquent` — mapea entre ambos mundos |
| **El dominio NUNCA ve un Modelo Eloquent. Jamás.** |

---

## 7. Service Providers (Inyección de Dependencias)

**Todos los `bind` de puertos → adaptadores se concentran en `CoreServiceProvider`
(método `register`).** Es el único lugar donde se casan interfaces de `Core` con
implementaciones de `Infraestructura`. `AppServiceProvider` solo maneja configuración de
framework (p.ej. `URL::forceScheme('https')` en producción).

```php
// app/Providers/CoreServiceProvider.php

class CoreServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // 1) Repositorios de dominio (Eloquent)
        $this->app->bind(
            InterfazRepositorioCotizacion::class,
            RepositorioCotizacionEloquent::class
        );
        $this->app->bind(
            InterfazRepositorioReserva::class,
            RepositorioReservaEloquent::class
        );
        // ... todos los InterfazRepositorio* -> Repositorio*Eloquent
        $this->app->bind(
            InterfazRepositorioItemCotizacion::class,
            RepositorioItemCotizacionEloquent::class
        );
        $this->app->bind(
            InterfazRepositorioPoliticaConfiguracion::class,
            RepositorioPoliticaConfiguracionEloquent::class
        );

        // 2) Fuentes externas (Anti-Corrupción)
        $this->app->bind(
            InterfazFuentePaquetes::class,
            AdaptadorWooCommerce::class
        );
        $this->app->bind(
            InterfazFuenteHoteles::class,
            AdaptadorHotelHttp::class
        );
        $this->app->bind(
            InterfazFuenteMovilidades::class,
            AdaptadorMovilidadHttp::class
        );

        // 3) Servicios de salida (infraestructura)
        $this->app->bind(
            InterfazGeneradorDocumentoPdf::class,
            GeneradorDocumentoPdfDomPdf::class
        );
        $this->app->bind(
            InterfazUnidadDeTrabajo::class,
            UnidadDeTrabajoEloquent::class
        );
    }
}
```

> Si un Caso de Uso necesita una nueva dependencia, se crea el **puerto en `Core`** y su
> **adaptador en `Infraestructura`**, y se registra aquí. Nunca se instancia una clase de
> `Infraestructura` directamente desde `Core`.

---

## 8. Eventos de Dominio

Los eventos ocurren en el dominio y se manejan en infraestructura.

### 8.1 Eventos definidos

| Evento | Disparado por | Escuchado por |
|--------|---------------|---------------|
| `CotizacionCreada` | CrearCotizacionUseCase | — |
| `CotizacionEnviada` | EnviarCotizacionUseCase | Crear registro en seguimientos |
| `CotizacionAprobada` | AprobarCotizacionUseCase | Notificar vendedor |
| `CotizacionConvertida` | ConvertirAReservaUseCase | — |
| `ReservaCreada` | CrearReservaUseCase | — |
| `ReservaConfirmada` | (cambio a estado Confirmada) | Generar PDF, Enviar correo cliente |
| `ReservaAnulada` | AnularReservaUseCase | Liberar secuencia |
| `PagoRegistrado` | RegistrarPagoUseCase | Verificar si saldo = 0 → confirmar reserva |
| `PagoAnulado` | AnularPagoUseCase | Recalcular saldo |
| `ClienteCreado` | CrearClienteUseCase | — |

---

## 9. Validación (Múltiples Capas)

| Capa | Qué valida | Dónde |
|------|-----------|-------|
| **1. Form Request** | Formato de datos (tipo, required, max length, email format) | `app/Http/Peticiones/*` |
| **2. Caso de Uso** | Reglas de negocio (R1, R2, R3, R4, RF) | `app/Core/Aplicacion/CasosUso/*` |
| **3. Entidad/ValueObject** | Integridad interna del dato (constructor) | `app/Core/Dominio/Entidades/*` + `ObjetosValor/*` |
| **4. Base de Datos** | Constraints (FK, UNIQUE, CHECK, NOT NULL) | Migraciones PostgreSQL |

**Ejemplo:** Crear cotización

```
Request POST /api/cotizaciones
  │
  ▼
FormRequest: ¿id_cliente es entero? ¿items es array?         ← Capa 1
  │
  ▼
CrearCotizacionUseCase::ejecutar():
  ├─ ¿Cliente existe y está activo? (repositorio)            ← Capa 2
  ├─ ¿Vendedor existe? (repositorio)                         ← Capa 2
  ├─ ¿R1? No crear cotización si cliente tiene cotización
  │      activa sin convertir                                ← Capa 2
  │
  ▼
Cotizacion::__construct():
  ├─ Precio::__construct(): trunca a 2 decimales (R2)        ← Capa 3
  ├─ EstadoCotizacion::__construct(): valida estado inicial  ← Capa 3
  │
  ▼
Guardar en BD → FK, UNIQUE, CHECK                           ← Capa 4
```

---

## 10. Manejo de Excepciones

### 10.1 Excepciones de Dominio

```php
namespace App\Core\Aplicacion\Excepciones;

class ExcepcionNegocio extends \RuntimeException
{
    public function __construct(
        string $mensaje,
        public readonly string $codigo = 'REGLA_NEGOCIO',
        int $httpStatus = 409
    ) {
        parent::__construct($mensaje, $httpStatus);
    }
}

class ExcepcionNoEncontrado extends ExcepcionNegocio
{
    public function __construct(string $entidad, int $id)
    {
        parent::__construct(
            "{$entidad} con ID {$id} no encontrada",
            'NO_ENCONTRADO',
            404
        );
    }
}

class ExcepcionValidacion extends ExcepcionNegocio
{
    public function __construct(array $errores, string $mensaje = 'Datos inválidos')
    {
        parent::__construct($mensaje, 'VALIDACION', 422);
        $this->errores = $errores;
    }
}

class ExcepcionTransicionEstadoInvalida extends ExcepcionNegocio
{
    public function __construct(string $estadoActual, string $nuevoEstado)
    {
        parent::__construct(
            "No se puede pasar de {$estadoActual} a {$nuevoEstado}",
            'TRANSICION_INVALIDA',
            409
        );
    }
}

class ExcepcionNoAutorizado extends ExcepcionNegocio
{
    public function __construct(string $mensaje = 'No tienes permiso para esta acción')
    {
        parent::__construct($mensaje, 'NO_AUTORIZADO', 403);
    }
}
```

### 10.2 Handler Global (app/Exceptions/Handler.php)

```php
public function register(): void
{
    $this->reportable(function (ExcepcionNegocio $e) {
        // No reportar como error, es esperado
    });

    $this->renderable(function (ExcepcionNegocio $e, Request $request) {
        return response()->json([
            'exitoso' => false,
            'datos' => null,
            'mensaje' => $e->getMessage(),
            'errores' => $e instanceof ExcepcionValidacion ? $e->errores : null,
        ], $e->getCode() ?: 400);
    });
}
```

---

## 11. Auditoría (R3)

### Arquitectura

```
Caso de Uso (dispara)
  │
  ▼
Evento de Dominio
  │
  ▼
Listener (en infraestructura)
  │
  ├─ Crea registro en tabla auditoria
  ├─ INSERT únicamente (R3)
  └─ Nunca UPDATE, nunca DELETE
```

### Implementación

La auditoría NO se hace con traits de Eloquent. Se hace explícita desde los Casos de Uso o desde Listeners de Eventos.

```php
// Dentro de un Caso de Uso:
public function ejecutar(CrearCotizacionDTO $dto): CotizacionDTO
{
    // ... lógica de negocio ...

    $this->eventDispatcher->despachar(
        new CotizacionCreada(
            cotizacionId: $cotizacion->id(),
            codigo: $cotizacion->codigo(),
            usuarioId: $dto->idVendedor,
        )
    );

    return new CotizacionDTO(/*...*/);
}

// Listener:
class RegistrarAuditoriaListener
{
    public function manejar(CotizacionCreada $evento): void
    {
        $this->repositorioAuditoria->guardar(
            new Auditoria(
                accion: 'Creación',
                tipoEntidad: 'Cotización',
                idEntidad: $evento->cotizacionId,
                // R3: solo INSERT
            )
        );
    }
}
```

---

## 12. Pruebas

### 12.1 Pirámide de Pruebas

```
         ╱╲
        ╱  ╲          E2E (Pest+ Laravel Dusk / Playwright)
       ╱    ╲         Pruebas de integración (controlador → BD real)
      ╱──────╲
     ╱        ╲       Pruebas de aplicación (Casos de Uso con repositorios mock)
    ╱──────────╲
   ╱            ╲     Pruebas de dominio (Entidades, ValueObjects, Reglas)
  ╱──────────────╲
```

### 12.2 Pruebas de Dominio (100% portátiles, sin Laravel)

```php
// tests/Unit/Dominio/PrecioTest.php
test('precio trunca a 2 decimales sin redondear', function () {
    $precio = new Precio('150.456');

    expect($precio->valor())->toBe('150.45');
});

test('precio no puede ser negativo', function () {
    (new Precio('-10.00'));
})->throws(\InvalidArgumentException::class);
```

### 12.3 Pruebas de Aplicación (Casos de uso con mocks)

```php
// tests/Unit/Aplicacion/CrearCotizacionUseCaseTest.php
test('crea cotizacion exitosamente', function () {
    $repositorio = Mockery::mock(InterfazRepositorioCotizacion::class);
    $repositorio->shouldReceive('guardar')->once();

    $useCase = new CrearCotizacionUseCase($repositorio, /*...*/);

    $resultado = $useCase->ejecutar($dto);

    expect($resultado->codigo)->toContain('COT-');
});
```

---

## 13. Resumen de Dependencias

| Package | Uso |
|---------|-----|
| `laravel/sanctum` | Autenticación API via tokens |
| `spatie/laravel-permission` | RBAC (roles y permisos) |
| `barryvdh/laravel-dompdf` | Generación de PDF |
| `maatwebsite/laravel-excel` | Exportación a Excel |
| `owen-it/laravel-auditing` | (Opcional) Auditoría automática |
| `nunomaduro/larastan` | Análisis estático PHPStan |
| `laravel/horizon` | Monitoreo de colas (fase VPS) |
| `predis/predis` | Cliente Redis para colas (fase VPS) |
