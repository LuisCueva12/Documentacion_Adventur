# Catálogo de Entidades del Sistema

**Carpeta:** 02 — Diccionario de Datos Agnóstico  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Listado Completo de Entidades

| # | Entidad | Propósito | Ciclo de Vida |
|---|---------|-----------|---------------|
| E01 | `usuarios` | Personas que operan el sistema | Activo → Inactivo (nunca se elimina) |
| E02 | `clientes` | Compradores/receptores de servicio turístico | Creado → Activo → Eliminado Lógico |
| E03 | `leads` | Prospectos no calificados que consultan | Nuevo → Cotizado/Perdido/Convertido a Cliente |
| E04 | `cotizaciones` | Propuestas comerciales enviadas al cliente | FSM 7 estados (ver CARPETA 01) |
| E05 | `cotizacion_detalle` | Desglose de ítems/servicios de una cotización | Vinculado al ciclo de cotización padre |
| E06 | `plantillas` | Modelos predefinidos para agilizar cotizaciones | Activo → Inactivo |
| E07 | `rutas` | Destinos/paquetes turísticos que ofrece Adventur | Activo → Inactivo |
| E08 | `seguimientos` | Acciones de post-envío para dar seguimiento al lead | Programado → Realizado/Vencido/Reprogramado |
| E09 | `motivos_perdida` | Catálogo de razones por las que no se cierra una venta | Siempre activo (configurable) |
| E10 | `reservas` | Servicios turísticos confirmados formalmente | Borrador → Pendiente → Confirmada → Finalizada/Anulada |
| E11 | `pasajeros` | Personas que viajan en una reserva específica | Vinculado al ciclo de la reserva padre |
| E12 | `pagos` | Transacciones monetarias aplicadas a una reserva | Registrado → Anulado |
| E13 | `politicas` | Condiciones legales y comerciales (PDF página 2) | Activo → Inactivo |
| E14 | `auditoria` | Traza histórica e inmutable de cambios en el sistema | Solo inserción, NUNCA modificación ni eliminación |
| E15 | `canales_origen` | Catálogo de procedencia del lead/consulta | Siempre activo |
| E16 | `secuencias_codigos` | Control de correlativos únicos por año y prefijo | Solo inserción y actualización de contador |
| E17 | `alojamientos` | Hospedaje asociado a una reserva (hotel, habitación, plan comidas) | Vinculado al ciclo de la reserva padre |
| E18 | `servicios_reserva` | Servicios operativos adicionales contratados en una reserva | Vinculado al ciclo de la reserva padre |

---

## 2. Mapa de Dependencias entre Entidades

```
USUARIOS
  ├── Crea: CLIENTES, LEADS, COTIZACIONES, SEGUIMIENTOS, RESERVAS, PAGOS
  ├── Asigna: LEADS, COTIZACIONES (vendedor responsable)
  ├── Supervisa: USUARIOS (relación supervisor-subordinado)
  └── Registra: AUDITORIA

LEADS
  └── Se convierte en: CLIENTES
  └── Genera: COTIZACIONES

CLIENTES
  ├── Tiene: COTIZACIONES
  └── Tiene: RESERVAS

PLANTILLAS
  ├── Pertenece a: RUTAS
  └── Es usada por: COTIZACIONES

COTIZACIONES
  ├── Contiene: COTIZACION_DETALLE
  ├── Tiene: SEGUIMIENTOS
  ├── Puede generar: RESERVAS (máximo 1)
  └── Puede tener: MOTIVOS_PERDIDA

RESERVAS
  ├── Contiene: PASAJEROS
  ├── Tiene: PAGOS
  ├── Tiene: ALOJAMIENTOS (opcional, solo rutas multi-día)
  ├── Tiene: SERVICIOS_RESERVA
  └── Proviene de: COTIZACIONES

AUDITORIA
  └── Registra cambios en: TODAS LAS ENTIDADES
```

---

## 3. Ciclo de Vida Detallado por Entidad

### E01: usuarios
```
CREACIÓN → ACTIVO → INACTIVO
  ↑                        │
  └────────────────────────┘ (reactivación por Admin)
```
- NUNCA se elimina físicamente.
- Un usuario "inactivo" no puede iniciar sesión.

### E02: clientes
```
CREACIÓN → ACTIVO → ELIMINACIÓN LÓGICA (oculto)
```
- El cliente eliminado lógicamente no aparece en búsquedas por defecto.
- Admin puede consultar clientes eliminados.
- No se puede crear una cotización o reserva para un cliente eliminado.

### E03: leads
```
CREACIÓN (NUEVO) → COTIZADO (se creó cotización) → se fusiona con cotizaciones
                 → PERDIDO (se descartó)
                 → CLIENTE (se identificó como existente)
```

### E10: reservas
```
BORRADOR → PENDIENTE → CONFIRMADA → FINALIZADA
             │              │
             ▼              ▼
          ANULADA        ANULADA (con penalidad si aplica)
```
- Finalizada: fecha de viaje ya ocurrió y se marcó como ejecutada.
- Anulada: cliente canceló antes o después de la confirmación.
- NUNCA se elimina físicamente una reserva.

### E12: pagos
```
REGISTRADO → ANULADO (solo por Contabilidad o Admin con motivo obligatorio)
```
- Un pago anulado NO se descuenta del saldo (se actualiza el saldo).
- NUNCA se elimina físicamente un pago.

### E17: alojamientos
```
CREACIÓN (asociado a reserva) → MODIFICACIÓN → ELIMINACIÓN LÓGICA
```
- Opcional: solo aplica cuando `reserva.alojamiento_requerido = TRUE`.
- Una reserva puede tener múltiples alojamientos (noches en diferentes hoteles).
- Vinculado al ciclo de vida de la reserva padre (soft delete en cascada).

### E18: servicios_reserva
```
CREACIÓN (asociado a reserva) → MODIFICACIÓN → ELIMINACIÓN LÓGICA
```
- Representa servicios adicionales contratados: seguro, cenas especiales, tours extra, etc.
- Opcional: no todas las reservas tienen servicios extras.
- Vinculado al ciclo de vida de la reserva padre.

### E14: auditoria
```
CREACIÓN (INSERT)
```
- Solo operación de inserción.
- NUNCA se actualiza ni elimina.
- Es el registro forense del sistema.
