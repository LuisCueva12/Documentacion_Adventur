# Flujo de Datos — Request a Respuesta

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 04 — Flujo de Datos

---

## 1. Flujo General de una Operación

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND (Vue 3)                                   │
│                                                                             │
│  ┌──────────┐    ┌────────────┐    ┌──────────┐    ┌───────────────────┐   │
│  │ Usuario  │───→│   Vista    │───→│ Servicio │───→│   TanStack Query  │   │
│  │ (click)  │    │  (Vue)     │    │  (Axios) │    │   (Caching)        │   │
│  └──────────┘    └────────────┘    └────┬─────┘    └───────────────────┘   │
│                                         │                                   │
│         ◄───────── JSON ◄───────────────┤                                   │
│                                         │                                   │
└─────────────────────────────────────────┼───────────────────────────────────┘
                                          │ HTTPS
                                          │ POST /api/reservas
                                          │ Authorization: Bearer {token}
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          LARAVEL API (Backend)                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     MIDDLEWARE (Kernel)                              │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │  │
│  │  │  CORS    │  │ Sanctum  │  │Throttle  │  │    Log / Debug     │  │  │
│  │  │(headers) │  │(token)   │  │(60/min)  │  │    (si aplica)     │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘  │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
│                             │                                             │
│  ┌──────────────────────────▼───────────────────────────────────────────┐  │
│  │                     FORM REQUEST (Validación)                        │  │
│  │  ¿campos required? ¿formato email? ¿monto > 0? ¿JSON válido?        │  │
│  │  Si falla → 422 con errores                                         │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
│                             │                                             │
│  ┌──────────────────────────▼───────────────────────────────────────────┐  │
│  │                     CONTROLADOR (Api/ReservaControlador)              │  │
│  │  1. Toma request validada                                            │  │
│  │  2. Crea DTO de entrada (CrearReservaDTO)                           │  │
│  │  3. Llama: $casoUso->ejecutar($dto)                                 │  │
│  │  4. Toma ResultadoDTO y lo convierte en JSON response               │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
│                             │                                             │
│  ┌──────────────────────────▼───────────────────────────────────────────┐  │
│  │                     CASO DE USO (CrearReservaUseCase)                │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ REGLA R1: ¿Cotización origen existe y no está convertida?    │  │  │
│  │  │ → $repositorioCotizacion->obtenerPorId($dto->idCotizacion)    │  │  │
│  │  │ → validar estado == Aprobado                                  │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ REGLA R2: Todos los montos truncados a 2 decimales           │  │  │
│  │  │ → $precio = new Precio($monto)   // trunca en constructor    │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ Crear entidad Reserva                                        │  │  │
│  │  │ → $reserva = new Reserva(                                    │  │  │
│  │  │     codigo: generarCodigo('ADV'),                            │  │  │
│  │  │     cliente: $cliente,                                       │  │  │
│  │  │     ...                                                       │  │  │
│  │  │   )                                                           │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ Persistir (vía interfaz, no sabe qué BD)                     │  │  │
│  │  │ → $repositorioReserva->guardar($reserva)                     │  │  │
│  │  │ → $repositorioPasajero->guardarVarios($reserva, $pasajeros)  │  │  │
│  │  │ → $repositorioPago->guardar($pago)                           │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ Disparar evento de dominio                                   │  │  │
│  │  │ → $this->eventDispatcher->despachar(                         │  │  │
│  │  │     new ReservaCreada($reserva->id())                        │  │  │
│  │  │   )                                                           │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────┐  │  │
│  │  │ Devolver resultado                                           │  │  │
│  │  │ → return new CrearReservaResultadoDTO(                       │  │  │
│  │  │     id: $reserva->id(),                                      │  │  │
│  │  │     codigo: $reserva->codigo(),                              │  │  │
│  │  │     estado: $reserva->estado(),                              │  │  │
│  │  │     saldoPendiente: $reserva->calcularSaldo()                │  │  │
│  │  │   )                                                           │  │  │
│  │  └──────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
│                             │                                             │
│  ┌──────────────────────────▼───────────────────────────────────────────┐  │
│  │                     INFRAESTRUCTURA (Persistencia)                    │  │
│  │                                                                      │  │
│  │  RepositorioReservaEloquent::guardar(Reserva $r)                     │  │
│  │    1. Convierte Entidad → Modelo Eloquent                           │  │
│  │    2. $modelo = new ModeloReserva()                                 │  │
│  │    3. $modelo->fill([...datos...])                                   │  │
│  │    4. $modelo->save()  → INSERT INTO reservas (...)                  │  │
│  │    5. $r->setId($modelo->id)                                        │  │
│  │                                                                      │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
│                             │                                             │
│  ┌──────────────────────────▼───────────────────────────────────────────┐  │
│  │                     BASE DE DATOS (PostgreSQL)                       │  │
│  │                                                                      │  │
│  │  BEGIN;                                                              │  │
│  │                                                                      │  │
│  │  INSERT INTO reservas (codigo, id_cliente, id_cotizacion, ...)       │  │
│  │  VALUES ('ADV-2026-0047', 5, 125, ...)                               │  │
│  │  RETURNING id;  → 47                                                 │  │
│  │                                                                      │  │
│  │  INSERT INTO pasajeros (id_reserva, nombre, documento, ...)          │  │
│  │  VALUES (47, 'María Fernández', '12345678', ...);                    │  │
│  │  VALUES (47, 'Carlos Fernández', '87654321', ...);                   │  │
│  │                                                                      │  │
│  │  INSERT INTO pagos (id_reserva, monto, metodo_pago, ...)             │  │
│  │  VALUES (47, 500.00, 'Yape', ...);                                   │  │
│  │                                                                      │  │
│  │  UPDATE secuencias_codigos SET ultimo_numero = ultimo_numero + 1     │  │
│  │  WHERE prefijo = 'ADV' AND año = 2026;                               │  │
│  │                                                                      │  │
│  │  INSERT INTO auditoria (accion, tipo_entidad, id_entidad, ...)       │  │
│  │  VALUES ('Creación', 'Reserva', 47, ...);                            │  │
│  │                                                                      │  │
│  │  COMMIT;                                                             │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  RESPUESTA (200):                                                           │
│  {                                                                          │
│    "exitoso": true,                                                         │
│    "datos": { "id": 47, "codigo": "ADV-2026-0047", ... },                  │
│    "mensaje": "Reserva creada correctamente"                                │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Flujo de la Máquina de Estados (Cotización)

```
         ┌──────────┐
         │ Borrador │  ← Se crea así
         └────┬─────┘
              │ POST /api/cotizaciones/{id}/enviar
              ▼
         ┌──────────┐
         │ Enviado  │  ← Se envía al cliente
         └────┬─────┘
              │ Cliente abre link
              ▼
         ┌──────────┐
         │  Visto   │  ← Cliente vio la cotización
         └────┬─────┘
              │
      ┌───────┴───────┐
      │               │
      ▼               ▼
  ┌──────────┐   ┌──────────┐
  │ Aprobado │   │Rechazado │  ← Terminal
  └────┬─────┘   └──────────┘
      │               │
      ▼               ▼
  ┌──────────┐   ┌──────────┐
  │Convertido│   │ Perdido  │  ← Terminal
  └──────────┘   └──────────┘
```

### Eventos disparados en cada transición:

| Transición | Evento de Dominio | Efecto en Sistema |
|-----------|-------------------|-------------------|
| Borrador → Enviado | `CotizacionEnviada` | Crea seguimiento "Programado" para 3 días después |
| Enviado → Visto | `CotizacionVista` | Actualiza seguimiento a "Visto", notifica vendedor |
| Visto → Aprobado | `CotizacionAprobada` | Registra descuento si aplica, actualiza seguimiento |
| Visto → Rechazado | `CotizacionRechazada` | Registra motivo, notifica vendedor |
| Aprobado → Convertido | `CotizacionConvertida` | Automáticamente crea reserva en estado Borrador |
| * → Perdido | `CotizacionPerdida` | Registra motivo, cierra seguimientos |

---

## 3. Flujo de la Máquina de Estados (Reserva)

```
         ┌───────────┐
         │ Borrador  │  ← Se crea así (desde cotización aprobada)
         └─────┬─────┘
               │ Completa datos + pago mínimo
               ▼
         ┌───────────┐
         │ Pendiente │  ← Cliente confirmó, pagó adelanto
         └─────┬─────┘
               │
      ┌────────┴────────┐
      │                 │
      ▼                 ▼
  ┌───────────┐   ┌───────────┐
  │ Confirmada│   │ Anulada   │  ← Terminal (soft delete)
  └─────┬─────┘   └───────────┘
        │
        │ Viaje realizado
        ▼
  ┌───────────┐
  │Finalizada │  ← Terminal
  └───────────┘
```

### Condiciones para cada transición:

| Transición | Condición |
|-----------|-----------|
| Borrador → Pendiente | Pasajeros completos + pago adelanto ≥ 30% del total |
| Pendiente → Confirmada | Saldo pendiente = 0 (todos los pagos registrados) |
| Pendiente → Anulada | Motivo de anulación + soft delete |
| Confirmada → Anulada | Solo si faltan > 48h para el viaje |
| Confirmada → Finalizada | Fecha viaje ya pasó + operaciones confirma |

---

## 4. Flujo de Pagos

```
Reserva creada con monto_total = 1250.00
  │
  ├─ Pago 1: 500.00  → saldo = 750.00  → estado: Pendiente
  ├─ Pago 2: 300.00  → saldo = 450.00  → estado: Pendiente
  ├─ Pago 3: 450.00  → saldo = 0.00    → estado: Confirmada ★
  │
  └─ Anular Pago 2 (motivo: devolución)
       → total_pagado = 950.00
       → saldo = 300.00
       → estado: Pendiente (vuelve atrás)
```

### Validaciones:

```
suma_pagos ≤ monto_total   [IR-06]
cada_pago.truncado_2_decimales  [R2]
adelanto_mínimo = monto_total × 0.30
  → primer_pago.monto ≥ adelanto_mínimo
```

### Eventos:

```
PagoRegistrado → ¿saldo == 0? → ReservaConfirmada
PagoAnulado    → ¿saldo > 0?  → ReservaPendiente (si estaba Confirmada)
```

---

## 5. Flujo de Generación de PDF

```
Usuario hace click en "Descargar PDF"
  │
  ▼
GET /api/reservas/{id}/pdf
  │
  ▼
GenerarPDFReservaUseCase::ejecutar(id)
  │
  ├─ 1. Obtener reserva con todos sus datos (repositorio)
  ├─ 2. Obtener políticas activas (repositorio)
  ├─ 3. Construir vista HTML con los datos
  │     ┌─────────────────────────────────────┐
  │     │ PÁGINA 1:                           │
  │     │ - Logo + datos empresa              │
  │     │ - Código de reserva                 │
  │     │ - Datos del cliente                 │
  │     │ - Datos del viaje                   │
  │     │ - Detalle de pasajeros              │
  │     │ - Resumen de pagos                  │
  │     │ - Código de verificación único       │
  │     ├─────────────────────────────────────┤
  │     │ PÁGINA 2:                           │
  │     │ - Políticas de cancelación           │
  │     │ - Políticas de pago                 │
  │     │ - Políticas de responsabilidad       │
  │     │ - Políticas generales               │
  │     └─────────────────────────────────────┘
  ├─ 4. Generar PDF (ServicioPdf::generar(html))
  ├─ 5. Registrar en auditoría
  └─ 6. Devolver PDF como respuesta binaria
```

---

## 6. Flujo de Búsqueda de Clientes (Autocomplete)

```
Usuario escribe "1234" en campo de búsqueda
  │
  ▼
Componente detecta pausa de 300ms (debounce)
  │
  ▼
GET /api/clientes/buscar?q=1234
  │
  ▼
Controlador → BuscarClienteUseCase
  │
  ├─ Buscar por DNI:  "1234%"
  ├─ Buscar por RUC:  "1234%"
  ├─ Buscar por email: "%1234%"
  ├─ Buscar por teléfono: "%1234%"
  ├─ Buscar por nombre: "%1234%"
  │
  ▼
Resultados (máx 10):
  [
    { id: 5, nombre: "María Fernández", documento: "DNI 12345678" },
    { id: 8, nombre: "José Martínez", documento: "DNI 12349876" }
  ]
  │
  ▼
Dropdown muestra opciones. Usuario selecciona.
  │
  ▼
Se cargan datos completos del cliente.
```

---

## 7. Flujo de Generación de Códigos Correlativos

```
Nueva cotización:
  │
  ▼
GenerarSecuenciaCodigoUseCase::ejecutar('COT', 2026)
  │
  ▼
1. Iniciar transacción (SERIALIZABLE isolation)
2. SELECT ultimo_numero FROM secuencias_codigos
   WHERE prefijo = 'COT' AND año = 2026
   FOR UPDATE  ← bloquea fila contra concurrencia
3. nuevo_numero = ultimo_numero + 1
4. UPDATE secuencias_codigos SET ultimo_numero = nuevo_numero
   WHERE prefijo = 'COT' AND año = 2026
5. COMMIT
6. Devolver: "COT-2026-" + str_pad(nuevo_numero, 4, '0')
```

**Garantía:** El `FOR UPDATE` + transacción serializable asegura que aunque 10 usuarios creen cotizaciones al mismo tiempo, cada uno recibe un número único y secuencial. Sin duplicados. Sin saltos.

---

## 8. Flujo de Reportes

```
Usuario selecciona filtros (fecha desde/hasta, vendedor)
  │
  ▼
GET /api/reportes/ventas?fecha_desde=2026-01-01&fecha_hasta=2026-06-30
  │
  ▼
GenerarReporteVentasUseCase
  │
  ▼
Consultas a BD:
  ├─ SELECT COUNT(*), SUM(monto_total) FROM reservas
  │   WHERE fecha_creacion BETWEEN ? AND ?
  │   AND estado != 'Anulada'
  │
  ├─ SELECT u.nombre, COUNT(r.id), SUM(r.monto_total)
  │   FROM reservas r
  │   JOIN usuarios u ON r.creado_por = u.id
  │   WHERE r.fecha_creacion BETWEEN ? AND ?
  │   GROUP BY u.id, u.nombre
  │
  ▼
Armar DTO de respuesta
  │
  ▼
Frontend recibe datos → Chart.js renderiza gráficos
  │
  ▼
Usuario puede exportar a Excel (SheetJS)
```

---

## 9. Flujo de Error (Ejemplo: Violación de Regla de Negocio)

```
Usuario intenta aprobar cotización sin ser Supervisor
  │
  ▼
POST /api/cotizaciones/5/aprobar
  │
  ▼
Middleware: ✅ Token válido
  │
  ▼
Controlador: Llama AprobarCotizacionUseCase
  │
  ▼
UseCase: Verifica permisos del usuario:
  ├─ ¿Usuario tiene rol Supervisor? → NO
  │
  ▼
Lanza: ExcepcionNoAutorizado(
  "Solo los Supervisores pueden aprobar cotizaciones"
)
  │
  ▼
Handler de excepciones captura:
  └─ Response 403:
     {
       "exitoso": false,
       "datos": null,
       "mensaje": "Solo los Supervisores pueden aprobar cotizaciones",
       "errores": null
     }
  │
  ▼
Frontend recibe 403:
  └─ useNotificacion().mostrarError(response)
  └─ Toast: "❌ Solo los Supervisores pueden aprobar cotizaciones"
```

---

## 10. Flujo de Datos entre Componentes (Frontend)

```
Vista (CotizacionDetalleView.vue)
  │
  ├─ Carga datos con TanStack Query:
  │   useQuery(['cotizacion', id], () => cotizacionesApi.obtener(id))
  │
  ├─ Renderiza componentes hijos:
  │   ├─ <BadgeEstado :estado="cotizacion.estado" />
  │   ├─ <DetalleCotizacion :cotizacion="cotizacion" />
  │   ├─ <ItemsCotizacion :items="cotizacion.items" />
  │   ├─ <HistorialEstados :historial="cotizacion.historial" />
  │   └─ <AccionesCotizacion
  │         :estado="cotizacion.estado"
  │         :estadosPermitidos="cotizacion.estados_permitidos"
  │         @enviar="manejarEnviar"
  │         @aprobar="manejarAprobar"
  │         @rechazar="manejarRechazar"
  │      />
  │
  └─ Acciones del usuario:
      └─ Click en "Enviar"
          └─ mutation.mutate(id)
              └─ onSuccess: invalidar query, mostrar toast, redirigir
```

### Comunicación entre componentes (solo props + eventos, sin store):

| Componente Padre | Componente Hijo | Props | Eventos |
|-----------------|-----------------|-------|---------|
| CotizacionDetalleView | AccionesCotizacion | estado, estadosPermitidos | @enviar, @aprobar, @rechazar, @perder, @convertir |
| CotizacionNuevaView | FormularioCotizacion | — | @guardar |
| Flujo5Pasos | Paso1/Paso2/... | pasoActual, datos | @siguiente, @anterior, @completar |
| ReservaDetalleView | ListaPasajeros | pasajeros | @editar, @eliminar |
| DashboardView | GraficoVentas | datos | — |

### Store (Pinia) solo cuando:

| Store | ¿Para qué? |
|-------|-----------|
| authStore | Sesión del usuario (disponible globalmente) |
| uiStore | Sidebar, tema, notificaciones |
| filtrosStore | Filtros persistentes entre vistas |
| Cualquier otra | Solo si 3+ componentes no relacionados necesitan el mismo estado |
