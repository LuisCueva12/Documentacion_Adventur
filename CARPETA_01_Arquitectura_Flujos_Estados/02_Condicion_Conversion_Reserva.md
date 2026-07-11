# Condición Lógica y Financiera para Conversión Cotización → Reserva

**Carpeta:** 01 — Arquitectura de Flujos y Estados  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Regla Inmutable (R1): Separación Estricta

> **Una Cotización es una PROPUESTA COMERCIAL NO VINCULANTE.**
> **Una Reserva es un COMPROMISO FORMAL DE SERVICIO.**

Ambas entidades son independientes. No existe estado intermedio que fusione los conceptos. La Cotización genera una propuesta de valor; la Reserva materializa el acuerdo comercial.

---

## 2. Condición Necesaria y Suficiente para la Conversión

```
CONVERTIR_A_RESERVA(cotizacion) → ÉXITO ↔ TODAS las siguientes condiciones son VERDADERAS:
```

| # | Condición | Expresión Lógica | Validación |
|---|-----------|-----------------|------------|
| C1 | Estado de cotización es APROBADO | `cotizacion.estado = "APROBADO"` | Irreversible: si no está aprobado, no se puede convertir |
| C2 | Cotización no está vencida | `cotizacion.valid_until >= CURRENT_DATE` | Si está vencida, error + opción de renovar |
| C3 | No existe reserva previa vinculada | `NOT EXISTS (SELECT 1 FROM reservas WHERE id_cotizacion = cotizacion.id)` | Una cotización genera máximo UNA reserva |
| C4 | Cliente existe y está activo | `clientes.id IS NOT NULL AND clientes.fecha_eliminacion IS NULL` | No se puede reservar para un cliente eliminado |
| C5 | Flujo de 5 pasos completado | Ver documentación `03_Flujo_5_Pasos_Reservas.md` | Cada paso debe validarse secuencialmente |

## 3. Condición para Confirmar la Reserva (Pendiente → Confirmada)

Una vez creada la reserva (estado `BORRADOR` o `PENDIENTE`), puede pasar a `CONFIRMADA`:

```
CONFIRMAR_RESERVA(reserva) → ÉXITO ↔ TODAS:
    (reserva.estado = "BORRADOR" O reserva.estado = "PENDIENTE")
    AND (reserva.adelanto_recibido >= reserva.monto_total * PORCENTAJE_ADELANTO_MINIMO)
    AND (reserva.fecha_eliminacion IS NULL)
    AND (reserva.fecha_anulacion IS NULL)
```

**Donde:**

| Variable | Definición | Valor por Defecto | ¿Configurable? |
|----------|-----------|-------------------|----------------|
| `PORCENTAJE_ADELANTO_MINIMO` | Fracción decimal del total que debe pagarse como adelanto para confirmar | `0.30` (30%) | Sí |
| `adelanto_recibido` | Suma de todos los pagos no anulados de la reserva | `SUM(pagos.monto WHERE pagos.id_reserva = reserva.id AND pagos.fecha_anulacion IS NULL)` | Automático |

---

## 4. Implicaciones Financieras

### Escenario A: Adelanto >= 30% → Confirmación Automática
```
total_pagado = 200.00
monto_total = 611.80
PORCENTAJE_ADELANTO_MINIMO = 0.30

200.00 >= 611.80 * 0.30
200.00 >= 183.54 → VERDADERO

reserva.estado → "CONFIRMADA"
```

### Escenario B: Adelanto < 30% → Permanece Pendiente
```
total_pagado = 100.00
monto_total = 611.80

100.00 >= 183.54 → FALSO

reserva.estado → "PENDIENTE"
```

### Escenario C: Pago Completo
```
total_pagado = 611.80
monto_total = 611.80
saldo = 0.00

reserva.estado → "CONFIRMADA"
pago.es_adelanto → FALSE
```

---

## 5. Secuencia Lógica de Conversión (Pseudocódigo)

```
FUNCIÓN ConvertirCotizacionEnReserva(cotizacion_id, datos_flujo_5_pasos):
    cotizacion = OBTENER(cotizaciones WHERE id = cotizacion_id)
    
    // Validar condiciones de conversión
    SI cotizacion.estado != "APROBADO":
        ERROR("La cotización debe estar en estado APROBADO para convertir")
    FIN SI
    
    SI cotizacion.valid_until < CURRENT_DATE:
        ERROR("Cotización vencida. Debe renovarse. Fecha expiración: {cotizacion.valid_until}")
    FIN SI
    
    SI EXISTE(reservas WHERE id_cotizacion = cotizacion.id):
        ERROR("Esta cotización ya fue convertida a la reserva {reserva_existente.codigo}")
    FIN SI
    
    // Ejecutar flujo de 5 pasos (cada paso valida antes de permitir el siguiente)
    resultado_flujo = EjecutarFlujo5Pasos(datos_flujo_5_pasos)
    
    SI resultado_flujo.exitoso == FALSE:
        ERROR("Error en paso {resultado_flujo.paso_fallido}: {resultado_flujo.mensaje}")
    FIN SI
    
    // Crear la reserva
    nueva_reserva = CREAR(reservas {
        codigo: GENERAR_CODIGO("ADV"),
        id_cotizacion: cotizacion.id,
        id_cliente: resultado_flujo.id_cliente,
        fecha_viaje: resultado_flujo.fecha_viaje,
        hora_salida: resultado_flujo.hora_salida,
        lugar_recojo: resultado_flujo.lugar_recojo,
        num_pasajeros: resultado_flujo.num_pasajeros,
        monto_total: cotizacion.total,
        pasajeros: resultado_flujo.pasajeros,
        total_pagado: resultado_flujo.total_pagado,
        saldo: resultado_flujo.monto_total - resultado_flujo.total_pagado,
        estado: "CONFIRMADA" SI resultado_flujo.total_pagado >= cotizacion.total * PORCENTAJE_ADELANTO_MINIMO
                SINO "PENDIENTE",
        creado_por: usuario_actual.id
    })
    
    // Registrar el pago si existe
    SI resultado_flujo.monto_pago > 0:
        CREAR(pagos {
            id_reserva: nueva_reserva.id,
            monto: resultado_flujo.monto_pago,
            metodo_pago: resultado_flujo.metodo_pago,
            referencia: resultado_flujo.referencia_pago,
            fecha_pago: resultado_flujo.fecha_pago,
            es_adelanto: resultado_flujo.monto_pago < cotizacion.total,
            recibido_por: usuario_actual.id
        })
    FIN SI
    
    // Actualizar estado de la cotización
    cotizacion.estado = "RESERVA"
    cotizacion.id_reserva = nueva_reserva.id
    
    // Registrar auditoría
    REGISTRAR_AUDITORIA(
        accion = "Conversión Cotización → Reserva",
        tipo_entidad = "Cotización",
        id_entidad = cotizacion.id,
        valores_anteriores = "estado=APROBADO",
        valores_nuevos = "estado=RESERVA|id_reserva={nueva_reserva.id}|codigo_reserva={nueva_reserva.codigo}"
    )
    
    RETORNAR nueva_reserva
FIN FUNCIÓN
```
