# Reglas de Descuentos y Vencimientos

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Bloqueo de Descuento Excesivo

### 1.1 Condición de Bloqueo

```
BLOQUEAR ← (porcentaje_descuento > LIMITE_SEGUN_ROL(rol_usuario))
            AND (NO_EXISTE_APROBACION())
```

Donde:

| Condición | Descripción |
|-----------|-------------|
| `porcentaje_descuento > LIMITE_SEGUN_ROL(rol)` | El descuento supera el máximo que el rol puede aplicar sin aprobación |
| `NO_EXISTE_APROBACION()` | No hay un registro de aprobación vinculado a esta cotización |

### 1.2 Límites por Rol

| Rol | Límite sin Aprobación | Requiere Aprobación de |
|-----|:---------------------:|:----------------------:|
| Vendedor | 10% | Supervisor |
| Supervisor | 15% | Administrador |
| Administrador | 50% | — (autónomo) |
| Cualquier rol | > 50% | **BLOQUEADO ABSOLUTO** |

### 1.3 Mecanismo de Bloqueo

El sistema NO permite persistir la cotización si:

```
FUNCIÓN ValidarYPersistirCotizacion(cotizacion, usuario):
    resultado_validacion = ValidarDescuento(
        cotizacion.porcentaje_descuento,
        usuario.rol,
        cotizacion.descuento_aprobado_por
    )
    
    SI resultado_validacion != APROBADO:
        MOSTRAR_ERROR_EN_PANTALLA("No se puede guardar: " + resultado_validacion.mensaje)
        BLOQUEAR_PERSISTENCIA()
        RETORNAR
    FIN SI
    
    // Validar vencimiento si la cotización ya existe
    SI cotizacion.id != NULL:
        SI cotizacion.valid_until < CURRENT_DATE:
            MOSTRAR_ALERTA("Esta cotización está vencida desde " + cotizacion.valid_until)
            // No bloquea el guardado de campos no financieros
            // Pero bloquea cambios de estado a "Aprobado" o "Reserva"
        FIN SI
    FIN SI
    
    PERSISTIR(cotizacion)
FIN FUNCIÓN
```

### 1.4 Flujo de Aprobación de Descuento

```
VENDEDOR crea cotización con descuento > 10%
  │
  ▼
SISTEMA BLOQUEA: "Requiere aprobación de Supervisor"
  │
  ▼
VENDEDOR envía notificación al Supervisor
  │
  ▼
SUPERVISOR revisa la cotización
  ├── APRUEBA (si descuento ≤ 15%):
  │     descuento_aprobado_por = supervisor.id
  │     Se registra auditoría A05
  │     Cotización se desbloquea
  │
  └── RECHAZA:
        Vendedor debe reducir el descuento
        Mensaje: "El Supervisor rechazó el descuento. Motivo: [motivo]"
```

---

## 2. Bloqueo de Cotización Vencida

### 2.1 Condición de Bloqueo

```
BLOQUEAR ← (cotizacion.estado ∈ {"Enviado", "En seguimiento", "Aprobado"})
            AND (CURRENT_DATE > cotizacion.valid_until)
```

### 2.2 Consecuencias de una Cotización Vencida

| Acción | ¿Permitida? | Comportamiento |
|--------|:-----------:|----------------|
| Visualizar cotización | SÍ | Muestra alerta visual: fondo amarillo/rojo + texto "VENCIDA" |
| Editar notas internas | SÍ | Solo notas, no montos ni estados |
| Cambiar estado a "Aprobado" | **NO** | Error: "Cotización vencida. Debe renovarse." |
| Convertir a "Reserva" | **NO** | Error: "Cotización vencida. Debe renovarse." |
| Enviar nuevamente | **NO** | Error: "Cotización vencida. Renueve antes de reenviar." |
| Marcar como "Perdido" | SÍ | Se puede cerrar como perdida |
| Renovar (crear nueva) | SÍ | Crea nueva cotización con datos copiados + nueva validez |
| Reactivar (Admin/Sup.) | SÍ | Extiende validez forzadamente |

### 2.3 Indicador Visual de Vencimiento

| Estado de Validez | Color de Fondo | Texto |
|-------------------|:--------------:|-------|
| `valid_until >= CURRENT_DATE + 7` | Normal | — |
| `valid_until BETWEEN CURRENT_DATE AND CURRENT_DATE + 7` | Amarillo | "Próximo a vencer" |
| `valid_until < CURRENT_DATE` | Rojo | "VENCIDA" |

### 2.4 Pseudocódigo de Renovación

```
FUNCIÓN RenovarCotizacion(quote_id_original):
    quote_original = OBTENER(cotizaciones WHERE id = quote_id_original)
    
    // Validar que esté en un estado renovable
    SI quote_original.estado NO ESTÁ EN {"Enviado", "En seguimiento"}:
        ERROR("Solo se pueden renovar cotizaciones en estado Enviado o En seguimiento")
    FIN SI
    
    // Clonar la cotización original con nuevos valores
    nueva_cotizacion = CLONAR(quote_original)
    nueva_cotizacion.id = NULL  // Nuevo registro
    nueva_cotizacion.codigo = GENERAR_CODIGO("COT")
    nueva_cotizacion.estado = "Cotizado"
    nueva_cotizacion.fecha_creacion = CURRENT_TIMESTAMP
    nueva_cotizacion.valid_until = CURRENT_DATE + quote_original.dias_validez
    nueva_cotizacion.notas_internas = (quote_original.notas_internas 
        + " | RENOVADO de " + quote_original.codigo + " (venció " + quote_original.valid_until + ")")
    
    // Marcar la original como renovada (nuevo estado interno)
    quote_original.estado = "Renovada"
    
    // Registrar auditoría
    REGISTRAR_AUDITORIA(
        accion = "Reactivación",
        tipo_entidad = "Cotización",
        id_entidad = quote_original.id,
        valores_anteriores = "estado=" + quote_original.estado_previo + "|valid_until=" + quote_original.valid_until,
        valores_nuevos = "estado=Renovada|nueva_cotizacion=" + nueva_cotizacion.codigo
    )
    
    PERSISTIR(quote_original)
    PERSISTIR(nueva_cotizacion)
    
    RETORNAR nueva_cotizacion
FIN FUNCIÓN
```

### 2.5 Pseudocódigo de Reactivación (Admin/Supervisor)

```
FUNCIÓN ReactivarCotizacion(quote_id, razon):
    usuario = OBTENER_USUARIO_ACTUAL()
    
    SI usuario.rol NO ESTÁ EN {"Administrador", "Supervisor"}:
        ERROR("Solo Administradores y Supervisores pueden reactivar cotizaciones vencidas")
    FIN SI
    
    quote = OBTENER(cotizaciones WHERE id = quote_id)
    
    SI quote.valid_until >= CURRENT_DATE:
        ERROR("La cotización no está vencida. No requiere reactivación")
    FIN SI
    
    // Extender validez
    quote.valid_until = CURRENT_DATE + quote.dias_validez
    quote.notas_internas = (quote.notas_internas 
        + " | REACTIVADO por " + usuario.nombre 
        + " - Razón: " + razon)
    
    REGISTRAR_AUDITORIA(
        accion = "Reactivación",
        tipo_entidad = "Cotización",
        id_entidad = quote.id,
        valores_anteriores = "valid_until=" + quote.valid_until_anterior,
        valores_nuevos = "valid_until=" + quote.valid_until + "|razon=" + razon
    )
    
    RETORNAR quote
FIN FUNCIÓN
```

---

## 3. Resumen de Reglas de Bloqueo

| ID | Regla | Condición de Bloqueo | Acción del Sistema |
|----|-------|---------------------|-------------------|
| DB-01 | Descuento excede límite del Vendedor | `%desc > 10%` AND rol=VENDEDOR AND sin aprobación | Error: "Requiere aprobación de Supervisor" |
| DB-02 | Descuento excede límite del Supervisor | `%desc > 15%` AND rol=SUPERVISOR AND sin aprobación | Error: "Requiere aprobación de Administrador" |
| DB-03 | Descuento excede límite absoluto | `%desc > 50%` | Error: "El descuento no puede exceder el 50%" |
| DB-04 | Cotización vencida → Aprobar | `valid_until < CURRENT_DATE` AND intento cambiar a "Aprobado" | Error: "Cotización vencida. Debe renovarse." |
| DB-05 | Cotización vencida → Convertir a Reserva | `valid_until < CURRENT_DATE` AND intento de conversión | Error: "Cotización vencida. Debe renovarse." |
| DB-06 | Cotización vencida → Enviar | `valid_until < CURRENT_DATE` AND intento de envío | Advertencia: "La cotización vencerá pronto. Renueve para extender validez." |
| DB-07 | Reserva con pago que excede saldo | `monto_pago > saldo_pendiente` | Error: "El pago excede el saldo pendiente (S/ X.XX)" |
| DB-08 | Seguimiento con fecha pasada | `scheduled_at <= CURRENT_TIMESTAMP` | Error: "La fecha de seguimiento debe ser futura" |
| DB-09 | Eliminación física de registro protegido | Intento de DELETE físico en entidad protegida | Error: "No está permitido eliminar físicamente este registro. Use la opción 'Anular'." |
