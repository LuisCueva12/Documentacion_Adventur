# Relaciones Lógicas y Cardinalidad entre Entidades

**Carpeta:** 02 — Diccionario de Datos Agnóstico  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Simbología de Cardinalidad

| Símbolo | Significado |
|---------|-------------|
| `1` | Exactamente uno |
| `0..1` | Cero o uno (opcional) |
| `N` | Muchos (cero o más) |
| `1..N` | Uno o más (mínimo obligatorio) |

---

## 2. Tabla de Relaciones

| Entidad A | Cardinalidad A | Entidad B | Cardinalidad B | Regla de Negocio |
|-----------|:--------------:|-----------|:--------------:|------------------|
| `usuarios` | `1` | `leads` (asignado_a) | `N` | Un vendedor gestiona muchos leads. Un lead tiene exactamente un vendedor asignado |
| `usuarios` | `1` | `clientes` (creado_por) | `N` | Un usuario puede crear muchos clientes. Un cliente es creado por exactamente un usuario |
| `usuarios` | `1` | `cotizaciones` (asignado_a) | `N` | Un vendedor tiene muchas cotizaciones asignadas. Una cotización tiene exactamente un vendedor responsable |
| `usuarios` | `1` | `cotizaciones` (creado_por) | `N` | Un usuario puede crear muchas cotizaciones |
| `usuarios` | `1` | `seguimientos` (creado_por) | `N` | Un usuario agenda muchos seguimientos |
| `usuarios` | `1` | `reservas` (creado_por) | `N` | Un vendedor crea muchas reservas |
| `usuarios` | `1` | `pagos` (recibido_por) | `N` | Un usuario recibe/registra muchos pagos |
| `usuarios` | `1` | `auditoria` | `N` | Un usuario genera muchos registros de auditoría |
| `usuarios` | `1` | `usuarios` (supervisor) | `0..N` | **Autorreferencial.** Un Supervisor tiene varios Vendedores a cargo. Un Vendedor tiene cero o un Supervisor |
| `leads` | `1` | `cotizaciones` | `0..N` | Un lead puede derivar en múltiples cotizaciones en el tiempo (o ninguna si se pierde antes de cotizar) |
| `leads` | `0..1` | `clientes` | `1` | Un lead puede convertirse en un cliente. Un cliente puede originarse de un lead o no |
| `clientes` | `1` | `cotizaciones` | `0..N` | Un cliente puede tener múltiples cotizaciones a lo largo del tiempo |
| `clientes` | `1` | `reservas` | `0..N` | Un cliente puede tener múltiples reservas (viajes recurrentes) |
| `cotizaciones` | `1` | `cotizacion_detalle` | `0..N` | Una cotización tiene cero o muchas líneas de detalle (ítems/servicios) |
| `cotizaciones` | `1` | `seguimientos` | `0..N` | Una cotización puede tener múltiples seguimientos programados |
| `cotizaciones` | `N` | `plantillas` | `1` | Muchas cotizaciones pueden usar la misma plantilla como modelo |
| `cotizaciones` | `N` | `rutas` | `1` | Muchas cotizaciones pueden referenciar la misma ruta/destino |
| `cotizaciones` | `1` | `reservas` | `0..1` | **Relación exclusiva.** Una cotización puede generar CERO o UNA reserva. Nunca más de una |
| `cotizaciones` | `N` | `motivos_perdida` | `0..1` | Una cotización perdida tiene exactamente cero o un motivo. Un motivo puede aplicarse a muchas cotizaciones |
| `plantillas` | `N` | `rutas` | `1` | Muchas plantillas pueden asociarse a una misma ruta |
| `reservas` | `1` | `pasajeros` | `1..N` | Una reserva tiene uno o muchos pasajeros (mínimo 1 obligatorio) |
| `reservas` | `1` | `pagos` | `0..N` | Una reserva puede tener cero o muchos pagos (múltiples abonos permitidos) |
| `reservas` | `N` | `clientes` | `1` | Muchas reservas pertenecen a un mismo cliente |
| `reservas` | `1` | `alojamientos` | `0..N` | Una reserva puede tener cero o muchos alojamientos (noches en hoteles). Opcional, solo rutas multi-día |
| `reservas` | `1` | `servicios_reserva` | `0..N` | Una reserva puede tener cero o muchos servicios adicionales contratados |

---

## 3. Diagrama Relacional (Texto)

```
USUARIOS ──1:N──> LEADS (asignado_a)
USUARIOS ──1:N──> CLIENTES (creado_por)
USUARIOS ──1:N──> COTIZACIONES (asignado_a, creado_por)
USUARIOS ──1:N──> SEGUIMIENTOS (creado_por)
USUARIOS ──1:N──> RESERVAS (creado_por)
USUARIOS ──1:N──> PAGOS (recibido_por)
USUARIOS ──1:N──> AUDITORIA
USUARIOS ──1:0..N──> USUARIOS (supervisor / autorreferencial)

LEADS ──1:0..N──> COTIZACIONES
LEADS ──0..1:1──> CLIENTES (conversión)

CLIENTES ──1:0..N──> COTIZACIONES
CLIENTES ──1:0..N──> RESERVAS

COTIZACIONES ──1:0..N──> COTIZACION_DETALLE
COTIZACIONES ──1:0..N──> SEGUIMIENTOS
COTIZACIONES ──N:1──> PLANTILLAS
COTIZACIONES ──N:1──> RUTAS
COTIZACIONES ──1:0..1──> RESERVAS  *** RELACIÓN EXCLUSIVA ***
COTIZACIONES ──N:0..1──> MOTIVOS_PERDIDA

PLANTILLAS ──N:1──> RUTAS

RESERVAS ──1:1..N──> PASAJEROS
RESERVAS ──1:0..N──> PAGOS
RESERVAS ──1:0..N──> ALOJAMIENTOS
RESERVAS ──1:0..N──> SERVICIOS_RESERVA
```

---

## 4. Reglas de Integridad Referencial (Restricciones Lógicas)

| # | Regla | Descripción |
|---|-------|-------------|
| IR-01 | **Cliente obligatorio en reserva** | No se puede crear una reserva sin un cliente existente y activo |
| IR-02 | **Cotización origen única** | Una reserva debe tener exactamente una cotización origen, y esa cotización no puede generar más de una reserva |
| IR-03 | **Vendedor responsable** | Toda cotización y lead debe tener un vendedor asignado (no puede quedar huérfano) |
| IR-04 | **Supervisor válido** | Si un vendedor tiene supervisor asignado, ese supervisor debe existir y tener rol `Supervisor` |
| IR-05 | **Pasajero titular único** | En cada reserva, exactamente un pasajero debe tener `es_titular = TRUE` |
| IR-06 | **Pago no puede exceder total** | La suma de pagos vigentes de una reserva nunca puede superar `reserva.monto_total` |
| IR-07 | **Motivo obligatorio en pérdida** | No se puede marcar una cotización como `Perdido` sin seleccionar un motivo del catálogo |
| IR-08 | **Soft delete en cascada lógica** | Si una cotización se elimina lógicamente, sus seguimientos y detalle también se ocultan. Lo mismo aplica para reservas → pasajeros + pagos |
| IR-09 | **Usuario creador siempre registrado** | Toda entidad que tenga `creado_por` debe tener un valor no nulo referenciando a un usuario existente |
| IR-10 | **Integridad de secuencias** | El código generado para cada entidad (COT, ADV, CLI, LEAD) debe ser único y secuencial por año |
| IR-11 | **Alojamiento requiere reserva activa** | Un alojamiento solo puede existir vinculado a una reserva existente y no eliminada. No existe alojamiento huérfano |
| IR-12 | **Servicio adicional vinculado a reserva** | Un servicio reserva solo puede existir vinculado a una reserva existente y no eliminada |

---

## 5. Ejemplos de Navegación Lógica

### Ejemplo 1: De un lead a su reserva (navegación completa)
```
lead.codigo = "LEAD-2026-0043"
  → lead.id_cotizaciones[0].codigo = "COT-2026-0124"
    → cotizacion.id_reserva.codigo = "ADV-2026-0047"
      → reserva.pasajeros[0].nombre = "María Fernández"
      → reserva.pasajeros[1].nombre = "Carlos Fernández"
      → reserva.pagos[0].monto = 200.00
```

### Ejemplo 2: De un pago a su vendedor
```
pago.id = 1
  → pago.id_reserva.codigo = "ADV-2026-0047"
    → pago.id_reserva.creado_por.nombre = "Juan Pérez"
    → pago.id_reserva.id_cliente.nombre = "María Fernández"
```

### Ejemplo 3: Seguimientos vencidos de un equipo
```
supervisor.nombre = "Carlos López"
  → supervisor.subordinados[0].nombre = "Juan Pérez"
    → subordinado.cotizaciones[0].seguimientos[0].estado = "Vencido"
```
