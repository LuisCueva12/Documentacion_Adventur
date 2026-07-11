# Fórmulas del Cotizador

**Carpeta:** 03 — Motor Matemático y Financiero  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Cálculo del Subtotal Base (SB)

```
SB = movilidad + guia + entradas + alimentacion + extras
```

| Variable | Fuente | Tipo Lógico | Límite |
|----------|--------|-------------|--------|
| `movilidad` | Entrada del vendedor (valor por defecto desde plantilla) | Decimal(10,2) | `>= 0.00` |
| `guia` | Entrada del vendedor (valor por defecto desde plantilla) | Decimal(10,2) | `>= 0.00` |
| `entradas` | Entrada del vendedor (valor por defecto desde plantilla) | Decimal(10,2) | `>= 0.00` |
| `alimentacion` | Entrada del vendedor (valor por defecto desde plantilla) | Decimal(10,2) | `>= 0.00` |
| `extras` | Entrada manual del vendedor (servicios adicionales) | Decimal(10,2) | `>= 0.00` |

**Precondición:** Todos los sumandos deben ser `>= 0.00`. El sistema rechaza valores negativos con error.

---

## 2. Cálculo del Total de Venta (TV)

### 2.1 Método Estándar (Margen Porcentual + Descuento Porcentual)

```
PASO 1: Subtotal con Margen (SM)
SM = SB * (1 + (porcentaje_margen / 100))

PASO 2: Monto de Descuento (MD)
MD = SM * (porcentaje_descuento / 100)

PASO 3: Total de Venta (TV)
TV = SM - MD
```

**Ecuación única:**

```
TV = [SB * (1 + (margen/100))] * (1 - (descuento/100))
```

### 2.2 Desglose por Componentes

| Componente | Fórmula | Ejemplo (SB=560, margen=15%, desc=5%) |
|------------|---------|---------------------------------------|
| Subtotal Base (SB) | `movilidad+guia+entradas+alimentacion+extras` | 560.00 |
| Monto Margen | `SB * (margen/100)` = `560 * 0.15` | 84.00 |
| SM (Base + Margen) | `SB + monto_margen` = `560 + 84` | 644.00 |
| Monto Descuento | `SM * (descuento/100)` = `644 * 0.05` | 32.20 |
| **Total Venta (TV)** | **`SM - monto_descuento`** = **`644 - 32.20`** | **611.80** |

### 2.3 Casos Borde

| Escenario | SB | margen | descuento | TV | Explicación |
|-----------|:--:|:------:|:---------:|:--:|-------------|
| Sin margen ni descuento | 560.00 | 0% | 0% | 560.00 | `TV = SB` |
| Solo margen | 560.00 | 15% | 0% | 644.00 | `TV = SB * 1.15` |
| Solo descuento | 560.00 | 0% | 10% | 504.00 | `TV = SB * 0.90` |
| Descuento > margen (pérdida planificada) | 560.00 | 10% | 20% | 492.80 | `TV < SB` permitido |
| Descuento = 100% | 560.00 | 0% | 100% | 0.00 | Bloqueado por regla de negocio (máx 50%) |

---

## 3. Cálculo del Precio por Persona (PPP)

```
PPP = TRUNC(TV / num_pasajeros, 2)
```

| Variable | Tipo Lógico | Límite |
|----------|-------------|--------|
| `TV` | Decimal(10,2) | `>= 0.00` |
| `num_pasajeros` | Entero | `>= 1` |
| `PPP` | Decimal(10,2) | `>= 0.00` |

**Ejemplo:**
- Si `TV = 611.80` y `num_pasajeros = 4`: `PPP = 611.80 / 4 = TRUNC(152.95, 2) = 152.95`
- Si `TV = 450.00` y `num_pasajeros = 2`: `PPP = 450.00 / 2 = 225.00`

---

## 4. Pseudocódigo del Motor de Cálculo

```
FUNCIÓN CalcularCotizacion(
    movilidad, guia, entradas, alimentacion, extras,
    num_pasajeros, porcentaje_margen, porcentaje_descuento
) → (base_total, monto_margen, monto_descuento, total, precio_por_persona)

    // ═══════════════════════════════════════════════
    // 1. Validar entradas
    // ═══════════════════════════════════════════════
    SI (movilidad < 0 O guia < 0 O entradas < 0 O alimentacion < 0 O extras < 0):
        ERROR("Los valores base no pueden ser negativos")
    FIN SI
    SI (num_pasajeros < 1):
        ERROR("Debe haber al menos 1 pasajero")
    FIN SI
    SI (porcentaje_margen < 0):
        ERROR("El margen no puede ser negativo")
    FIN SI
    SI (porcentaje_descuento < 0):
        ERROR("El descuento no puede ser negativo")
    FIN SI

    // ═══════════════════════════════════════════════
    // 2. Calcular base total (truncado a 2 decimales)
    // ═══════════════════════════════════════════════
    base_total = TRUNC(movilidad + guia + entradas + alimentacion + extras, 2)

    // ═══════════════════════════════════════════════
    // 3. Calcular margen
    // ═══════════════════════════════════════════════
    monto_margen = TRUNC(base_total * (porcentaje_margen / 100), 2)
    subtotal_con_margen = base_total + monto_margen

    // ═══════════════════════════════════════════════
    // 4. Calcular descuento sobre (base + margen)
    // ═══════════════════════════════════════════════
    monto_descuento = TRUNC(subtotal_con_margen * (porcentaje_descuento / 100), 2)

    // ═══════════════════════════════════════════════
    // 5. Calcular total
    // ═══════════════════════════════════════════════
    total = TRUNC(subtotal_con_margen - monto_descuento, 2)

    // ═══════════════════════════════════════════════
    // 6. Calcular precio por persona
    // ═══════════════════════════════════════════════
    precio_por_persona = TRUNC(total / num_pasajeros, 2)

    // ═══════════════════════════════════════════════
    // 7. Validar reglas de descuento según rol
    // ═══════════════════════════════════════════════
    SI (porcentaje_descuento > LIMITE_SEGUN_ROL(usuario_actual.rol)):
        REQUIERE_APROBACION = TRUE
        SI (NO_EXISTE_APROBACION()):
            ERROR("El descuento del " + porcentaje_descuento + "% requiere aprobación")
        FIN SI
    FIN SI

    // ═══════════════════════════════════════════════
    // 8. Validar descuento máximo absoluto
    // ═══════════════════════════════════════════════
    SI (porcentaje_descuento > 50):
        ERROR("El descuento no puede exceder el 50% del subtotal con margen")
    FIN SI

    RETORNAR (base_total, monto_margen, monto_descuento, total, precio_por_persona)
FIN FUNCIÓN
```

---

## 5. Tabla de Validación de Límites

| Regla | Condición | Acción del Sistema |
|-------|-----------|-------------------|
| Valores base negativos | Cualquier `base_* < 0` | ERROR: bloquea guardado |
| Pasajeros inválido | `num_pasajeros < 1` | ERROR: bloquea guardado |
| Margen negativo | `porcentaje_margen < 0` | ERROR: bloquea guardado |
| Descuento negativo | `porcentaje_descuento < 0` | ERROR: bloquea guardado |
| Descuento > 10% (Vendedor) | `porcentaje_descuento > 10` AND rol = "Vendedor" | Bloquea, requiere aprobación Supervisor |
| Descuento > 15% (Supervisor) | `porcentaje_descuento > 15` AND rol = "Supervisor" | Bloquea, requiere aprobación Admin |
| Descuento > 50% (Admin) | `porcentaje_descuento > 50` | Bloquea absolute, no permitido |
| Base total > 1,000,000 | `base_total > 1000000.00` | Alerta de monto inusualmente alto (no bloquea) |
