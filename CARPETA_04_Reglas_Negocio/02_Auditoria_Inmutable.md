# Regla de Auditoría Inmutable

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Regla Inmutable (R3): Solo Inserción, Nunca Modificación

> Los registros de auditoría son de **SOLO INSERCIÓN (INSERT)**.
> **NUNCA** se actualizan, modifican ni eliminan.
> Constituyen la evidencia forense del sistema.

---

## 2. Eventos que Disparan Auditoría Obligatoria

| ID | Evento | Entidad Afectada | Valores a Capturar | Justificación |
|----|--------|------------------|---------------------|---------------|
| A01 | Creación de cotización | `cotizaciones` | `{"estado": "nuevo"}` | Saber quién y cuándo inició el proceso comercial |
| A02 | Envío de cotización | `cotizaciones` | `{"estado": "→enviado", enviado_via, fecha_envio}` | Trazabilidad del contacto con el cliente |
| A03 | Programación de seguimiento | `seguimientos` | `{"fecha_programada", "medio_contacto"}` | Control de acciones de postventa |
| A04 | Reprogramación de seguimiento | `seguimientos` | `{"fecha_anterior" → "fecha_nueva"}` | Historial de cambios en fechas acordadas |
| A05 | Aprobación de descuento | `cotizaciones` | `{"%_descuento_anterior" → "nuevo", "aprobado_por"}` | Control financiero y de autorizaciones |
| A06 | Conversión cotización → reserva | `cotizaciones` + `reservas` | `{"cotizacion_id", "reserva_id", "estado": "aprobado" → "reserva"}` | Trazabilidad del cierre de venta |
| A07 | Registro de pago | `pagos` | `{"monto", "metodo_pago", "saldo_anterior" → "saldo_nuevo"}` | Control de caja |
| A08 | Anulación de pago | `pagos` | `{"monto_anulado", "motivo", "saldo_anterior" → "saldo_nuevo"}` | Trazabilidad de correcciones financieras |
| A09 | Anulación de reserva | `reservas` | `{"estado": "→anulada", "motivo", "anulado_por"}` | Control de cancelaciones |
| A10 | Reactivación de cotización perdida | `cotizaciones` | `{"estado": "perdido" → "cotizado", "razon_reactivacion"}` | Excepción documentada con autorización |
| A11 | Modificación de montos en reserva confirmada | `reservas` | `{"campo_modificado": "anterior" → "nuevo"}` | Protección contra cambios no autorizados |
| A12 | Creación/modificación de usuario | `usuarios` | `{"rol", "activo"}` | Seguridad del sistema |
| A13 | Eliminación lógica de cualquier entidad | Cualquiera | `{"fecha_eliminacion": "→timestamp"}` | Saber quién y cuándo ocultó un registro |
| A14 | Reactivación de usuario inactivo | `usuarios` | `{"activo": "false" → "true"}` | Control de accesos |

---

## 3. Estructura del Registro de Auditoría

Cada registro de auditoría contiene OBLIGATORIAMENTE:

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|:-----------:|-------------|
| `id` | Entero autogenerado | SÍ | Identificador único del registro de auditoría |
| `id_usuario` | ID | NO | Referencia a `usuarios.id`. NULL si la acción fue ejecutada automáticamente por el sistema |
| `accion` | Texto(50) | SÍ | Una de: `Creación`, `Modificación`, `Envío`, `Aprobación`, `Anulación`, `Pago`, `Conversión`, `Reactivación` |
| `tipo_entidad` | Texto(100) | SÍ | Nombre de la entidad afectada: `Cotización`, `Reserva`, `Pago`, `Plantilla`, `Usuario`, etc. |
| `id_entidad` | Entero | SÍ | ID del registro específico que fue afectado |
| `valores_anteriores` | Texto(L) | NO | Pares `campo=valor` separados por `\|`. Ej: `estado=enviado\|total=500.00` |
| `valores_nuevos` | Texto(L) | NO | Pares `campo=valor` separados por `\|`. Ej: `estado=aprobado\|total=500.00` |
| `direccion_ip` | Texto(45) | NO | Dirección IPv4 o IPv6 desde donde se ejecutó la acción |
| `agente_usuario` | Texto(L) | NO | Cadena User-Agent del navegador o aplicación cliente |
| `fecha_creacion` | FechaHora | SÍ | Timestamp de creación. **INMUTABLE: nunca se modifica** |

---

## 4. Ejemplos Reales de Registros de Auditoría

### Ejemplo 1: Envío de Cotización

```
Evento: Vendedor Juan Pérez (ID=5) envía cotización COT-2026-0124

Registro:
  id: 1523
  id_usuario: 5
  accion: "Envío"
  tipo_entidad: "Cotización"
  id_entidad: 124
  valores_anteriores: "estado=cotizado|fecha_envio=NULL|enviado_via="
  valores_nuevos: "estado=enviado|fecha_envio=2026-07-26 15:30:00|enviado_via=WhatsApp"
  direccion_ip: "190.119.XX.XX"
  agente_usuario: "Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."
  fecha_creacion: "2026-07-26 15:30:05"
```

### Ejemplo 2: Conversión a Reserva

```
Evento: Cotización COT-2026-0124 convertida a reserva ADV-2026-0047

Registro:
  id_usuario: 5
  accion: "Conversión"
  tipo_entidad: "Cotización"
  id_entidad: 124
  valores_anteriores: "estado=aprobado|id_reserva=NULL"
  valores_nuevos: "estado=reserva|id_reserva=47|codigo_reserva=ADV-2026-0047"
```

### Ejemplo 3: Anulación de Reserva

```
Evento: Supervisor Carlos López (ID=2) anula reserva ADV-2026-0047

Registro:
  id_usuario: 2
  accion: "Anulación"
  tipo_entidad: "Reserva"
  id_entidad: 47
  valores_anteriores: "estado=confirmada|fecha_anulacion=NULL"
  valores_nuevos: "estado=anulada|fecha_anulacion=2026-07-28 10:00:00|motivo=Cliente canceló por enfermedad"
```

---

## 5. Comportamiento del Sistema ante la Auditoría

### 5.1 Consistencia de Datos

```
SI valores_anteriores == NULL Y valores_nuevos != NULL:
    → Es una operación de CREACIÓN

SI valores_anteriores != NULL Y valores_nuevos != NULL:
    → Es una operación de MODIFICACIÓN o CAMBIO DE ESTADO
    → Los pares deben diferir en al menos un campo
```

### 5.2 Restricciones de Acceso

- Solo el rol **Administrador** puede consultar todos los registros de auditoría.
- El rol **Supervisor** puede consultar auditoría de los registros de su equipo.
- Ningún otro rol tiene acceso a la entidad `auditoria`.

### 5.3 Retención de Datos

Los registros de auditoría se conservan por **tiempo indefinido**. No existe política de purge o eliminación. Si porrazones de capacidad se requiere archivar, debe hacerse una copia de seguridad antes de cualquier purga, y esa copia debe estar disponible para consulta.
