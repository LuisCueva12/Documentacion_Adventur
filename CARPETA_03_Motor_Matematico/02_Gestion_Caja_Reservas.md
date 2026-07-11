# Gestión de Caja y Reservas

**Carpeta:** 03 — Motor Matemático y Financiero  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Cálculo de Saldo Pendiente en Tiempo Real

### 1.1 Fórmula Principal

```
SALDO_PENDIENTE = MONTO_TOTAL_RESERVA - SUM(PAGOS_VIGENTES)
```

**Donde:**

| Variable | Definición |
|----------|------------|
| `MONTO_TOTAL_RESERVA` | `reservas.monto_total`. Valor fijo tomado de la cotización origen al convertir |
| `PAGOS_VIGENTES` | Conjunto de pagos donde: `pagos.id_reserva = reserva.id` AND `pagos.fecha_anulacion IS NULL` |
| `SUM(PAGOS_VIGENTES)` | Suma de todos los montos de pagos no anulados |

### 1.2 Regla de Recalculo Automático

El sistema recalcula `total_pagado` y `saldo` en la entidad `reservas` CADA VEZ que ocurre uno de estos eventos:

| Evento | Acción |
|--------|--------|
| Se inserta un nuevo pago | Recalcular |
| Se anula un pago existente | Recalcular |
| Se modifica el monto de un pago (excepcional) | Recalcular |

### 1.3 Pseudocódigo del Recalculo

```
FUNCIÓN RecalcularSaldoReserva(reserva_id):
    total_pagado = SUM(pagos.monto
                       WHERE pagos.id_reserva = reserva_id
                       AND pagos.fecha_anulacion IS NULL)
    
    reserva = OBTENER(reservas WHERE id = reserva_id)
    reserva.total_pagado = total_pagado
    reserva.saldo = TRUNC(reserva.monto_total - total_pagado, 2)
    
    // Validación de consistencia
    SI reserva.saldo < 0:
        ERROR("INCONSISTENCIA: El saldo no puede ser negativo. "
              + "Total: " + reserva.monto_total
              + ", Pagado: " + total_pagado
              + ". Verificar pagos registrados.")
    FIN SI
    
    // Actualizar estado según nivel de pago
    SI total_pagado >= reserva.monto_total * PORCENTAJE_ADELANTO_MINIMO
       Y reserva.estado = "PENDIENTE":
        reserva.estado = "CONFIRMADA"
    FIN SI
    
    SI total_pagado >= reserva.monto_total:
        reserva.saldo = 0.00
    FIN SI
    
    ACTUALIZAR(reserva)
    RETORNAR reserva
FIN FUNCIÓN
```

### 1.4 Ejemplo con Múltiples Abonos

| Evento | Monto Pagado | Total Pagado | Saldo | Estado |
|--------|:-----------:|:-----------:|:-----:|:------:|
| Creación reserva (total=611.80) | — | 0.00 | 611.80 | Pendiente |
| Pago 1: Adelanto Yape | 200.00 | 200.00 | 411.80 | Confirmada (200 >= 183.54) |
| Pago 2: Transferencia | 300.00 | 500.00 | 111.80 | Confirmada |
| Pago 3: Efectivo | 111.80 | 611.80 | 0.00 | Confirmada (saldo = 0) |
| Anulación Pago 1 (motivo: error) | -200.00 | 411.80 | 200.00 | Confirmada |

---

## 2. Porcentaje Mínimo de Adelanto Recomendado

### 2.1 Fórmula

```
ADELANTO_RECOMENDADO = MONTO_TOTAL_RESERVA * PORCENTAJE_ADELANTO_MINIMO
```

| Variable | Valor por Defecto | ¿Configurable? | Descripción |
|----------|:-----------------:|:--------------:|-------------|
| `PORCENTAJE_ADELANTO_MINIMO` | `0.30` (30%) | Sí | Fracción decimal del total que debe pagarse como adelanto |

### 2.2 Comportamiento del Sistema

| Escenario | Acción del Sistema |
|-----------|-------------------|
| Adelanto >= 30% | Reserva pasa automáticamente a estado `CONFIRMADA` |
| Adelanto < 30% | Reserva permanece en estado `PENDIENTE`. Sistema muestra advertencia: "El adelanto mínimo recomendado es del 30% (S/ X.XX)" |
| Sin adelanto (monto = 0) | Reserva se crea en estado `BORRADOR`. Sistema muestra alerta: "La reserva no tiene pagos registrados" |
| Pago completo (100%) | Reserva pasa a `CONFIRMADA`. Adelanto se marca como `FALSE` (es pago completo) |

### 2.3 Ejemplos

```
Reserva A: Total = 611.80
  Adelanto 30% = 611.80 * 0.30 = 183.54
  Si paga 200.00 → CONFIRMADA (200.00 >= 183.54)
  Si paga 100.00 → PENDIENTE (100.00 < 183.54)

Reserva B: Total = 450.00
  Adelanto 30% = 450.00 * 0.30 = 135.00
  Si paga 135.00 → CONFIRMADA (135.00 >= 135.00)
```

---

## 3. Regla de Consistencia Financiera (Garantía de Integridad)

```
SIEMPRE SE DEBE CUMPLIR:
    reserva.total_pagado + reserva.saldo = reserva.monto_total
```

Esta ecuación es la **prueba de fuego** del módulo financiero. Si en algún momento no se cumple, el sistema está en estado de inconsistencia y debe bloquear cualquier operación adicional hasta que se resuelva.

**Verificación automatizada:**
```
FUNCIÓN VerificarConsistenciaFinanciera():
    PARA_CADA reserva EN reservas WHERE fecha_eliminacion IS NULL:
        suma_esperada = reserva.total_pagado + reserva.saldo
        SI suma_esperada != reserva.monto_total:
            GENERAR_ALERTA("Inconsistencia en reserva " + reserva.codigo
                           + ": suma(" + suma_esperada + ") != monto(" + reserva.monto_total + ")")
        FIN SI
    FIN PARA_CADA
FIN FUNCIÓN
```
