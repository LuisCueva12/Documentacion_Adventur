# Regla de Control Decimal (Garantía de Cero Descuadres)

**Carpeta:** 03 — Motor Matemático y Financiero  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Regla Inmutable (R2): Truncamiento a 2 Decimales

> **TODAS LAS OPERACIONES MONETARIAS** se realizan con TRUNCAMIENTO a 2 decimales.
> Queda prohibido el uso de redondeo bancario o redondeo matemático en cualquier cálculo financiero del sistema.

---

## 2. Definición Formal de TRUNC(x, 2)

```
TRUNC(x, 2) = └ x × 100 ┘ / 100
```

Donde `└ n ┘` es la función **piso (floor)** que devuelve el entero más grande ≤ n.

### Ejemplos de Aplicación

| Valor Original | Cálculo Intermedio | TRUNC(x, 2) |
|:--------------:|:------------------:|:-----------:|
| 152.959 | `└15295.9┘ / 100 = 15295 / 100` | **152.95** |
| 152.951 | `└15295.1┘ / 100 = 15295 / 100` | **152.95** |
| 152.950 | `└15295.0┘ / 100 = 15295 / 100` | **152.95** |
| 152.949 | `└15294.9┘ / 100 = 15294 / 100` | **152.94** |
| 100.009 | `└10000.9┘ / 100 = 10000 / 100` | **100.00** |
| 100.001 | `└10000.1┘ / 100 = 10000 / 100` | **100.00** |

---

## 3. Propiedades del Truncamiento

### 3.1 Consistencia Aditiva

```
TRUNC(a + b, 2) = TRUNC(TRUNC(a, 2) + TRUNC(b, 2), 2)
```

Esta propiedad garantiza que **la suma de partes truncadas siempre sea ≤ el total truncado**.

### 3.2 Ausencia de Descuadre (Prueba Matemática)

Dado que cada pago individual se trunca a 2 decimales:

```
Pago 1: TRUNC(x₁, 2)
Pago 2: TRUNC(x₂, 2)
...
Pago N: TRUNC(xₙ, 2)

Total pagado = Σ TRUNC(xᵢ, 2)   [para i = 1..n, con fecha_anulacion IS NULL]
Saldo = TRUNC(TotalReserva - TotalPagado, 2)
```

**Propiedad garantizada:** `TotalPagado + Saldo = TotalReserva` **SIEMPRE**.

### 3.3 Contraejemplo con Redondeo Bancario

Si el sistema usara redondeo bancario (redondeo al más cercano, .5 hacia arriba):

```
Total Reserva = 100.00
3 pagos de 33.33... cada uno

Redondeo bancario de 33.333... → 33.33 (cada uno)
Suma = 33.33 + 33.33 + 33.33 = 99.99
DESCUADRE: 99.99 != 100.00 → FALTA 1 CÉNTIMO
```

**Este error NO PUEDE OCURRIR** con truncamiento a 2 decimales, porque:

```
33.333... → TRUNC(33.333..., 2) = └3333.3┘ / 100 = 3333 / 100 = 33.33
Suma = 99.99
Saldo = TRUNC(100.00 - 99.99, 2) = TRUNC(0.01, 2) = 0.01
Verificación: 99.99 + 0.01 = 100.00 ✓
```

---

## 4. Puntos de Aplicación Obligatoria del TRUNC

| # | Operación | Variable | Aplicación |
|---|-----------|----------|------------|
| 1 | Cálculo de Base Total | `base_total` | `TRUNC(movilidad+guia+entradas+alimentacion+extras, 2)` |
| 2 | Cálculo de Margen | `monto_margen` | `TRUNC(base_total * (margen/100), 2)` |
| 3 | Cálculo de Descuento | `monto_descuento` | `TRUNC(SM * (descuento/100), 2)` |
| 4 | Cálculo de Total | `total` | `TRUNC(SM - monto_descuento, 2)` |
| 5 | Cálculo de Precio por Persona | `precio_por_persona` | `TRUNC(total / num_pasajeros, 2)` |
| 6 | Registro de cada Pago | `pago.monto` | `TRUNC(monto_ingresado, 2)` — el usuario ingresa el monto manualmente, pero se verifica que coincida con TRUNC |
| 7 | Cálculo de Saldo | `saldo` | `TRUNC(monto_total - total_pagado, 2)` |
| 8 | Total Pagado (Suma) | `total_pagado` | `TRUNC(SUM(pagos.monto), 2)` |

---

## 5. Validación Final (Prueba de Fuego)

```
PARA CADA reserva EN el sistema:
    SI (reserva.total_pagado + reserva.saldo) != reserva.monto_total:
        ERROR CRÍTICO: Inconsistencia financiera en reserva {reserva.codigo}
    FIN SI
FIN PARA CADA
```

Esta validación se ejecuta **automáticamente** después de cada operación de pago (creación, anulación, modificación). Si falla, el sistema bloquea la operación y registra una alerta inmediata.

---

## 6. Especificación de la Función TRUNC

```
FUNCIÓN TRUNC(valor, decimales):
    // valor: número decimal a truncar
    // decimales: número de decimales (siempre 2 en este sistema)
    
    SI valor ES NULO:
        RETORNAR 0.00
    FIN SI
    
    SI decimales < 0:
        ERROR("Los decimales deben ser >= 0")
    FIN SI
    
    factor = 10 ^ decimales    // 10^2 = 100
    resultado = PISO(valor * factor) / factor
    
    // PISO(n) = máximo entero ≤ n
    // PISO(3.14159 * 100) = PISO(314.159) = 314
    // 314 / 100 = 3.14
    
    RETORNAR resultado
FIN FUNCIÓN
```

**Importante:** La función `PISO` (floor) debe comportarse correctamente con números negativos. En este sistema, todos los montos son ≥ 0, pero si por alguna razón se procesa un número negativo, `PISO(-3.14) = -4` (no `-3`), lo cual preserva la consistencia.
