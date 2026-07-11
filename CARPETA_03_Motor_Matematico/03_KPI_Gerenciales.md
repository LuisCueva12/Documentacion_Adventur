# KPI Gerenciales

**Carpeta:** 03 — Motor Matemático y Financiero  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Tasa de Conversión de Ventas (TCV)

### Fórmula

```
TCV = (RESERVAS_CONFIRMADAS / COTIZACIONES_ENVIADAS) × 100
```

### Definición de Variables

| Variable | Definición | Filtro de Período |
|----------|-----------|-------------------|
| `RESERVAS_CONFIRMADAS` | Número de reservas en estado `Confirmada` o `Finalizada` | `reservas.fecha_creacion BETWEEN fecha_inicio AND fecha_fin` |
| `COTIZACIONES_ENVIADAS` | Número de cotizaciones en estado `Enviado`, `En seguimiento`, `Aprobado` o `Reserva` | `cotizaciones.fecha_envio BETWEEN fecha_inicio AND fecha_fin` |

### Precisión

```
2 decimales. Formato: XX.XX%
```

### Ejemplo

```
Período: Julio 2026
Cotizaciones enviadas: 85
Reservas confirmadas: 23
TCV = (23 / 85) × 100 = 27.06%
```

### Filtros Aplicables

| Filtro | Tipo | Opciones |
|--------|------|----------|
| Período | Rango de fechas | Inicio — Fin (obligatorio) |
| Vendedor | ID | Un vendedor específico o "Todos" |
| Ruta | ID | Una ruta específica o "Todas" |
| Canal de origen | Código | Canal específico o "Todos" |

---

## 2. Ticket Promedio Cotizado (TPC)

### Fórmula

```
TPC = SUM(COTIZACIONES.total) / COUNT(COTIZACIONES.id)
```

### Definición de Variables

| Variable | Definición | Filtro |
|----------|-----------|--------|
| `SUM(COTIZACIONES.total)` | Sumatoria del campo `total` de todas las cotizaciones en el período | `cotizaciones.estado != "Nuevo"` (solo cotizaciones con valor económico) |
| `COUNT(COTIZACIONES.id)` | Número total de cotizaciones en el período | Mismo filtro |

### Precisión

```
2 decimales. Formato: S/ X,XXX.XX
```

### Ejemplo

```
Período: Julio 2026
Suma total cotizado: S/ 36,193.00
Número de cotizaciones: 85
TPC = 36,193.00 / 85 = S/ 425.80
```

---

## 3. Ticket Promedio Vendido (TPV)

### Fórmula

```
TPV = SUM(RESERVAS.monto_total) / COUNT(RESERVAS.id)
```

### Definición de Variables

| Variable | Definición | Filtro |
|----------|-----------|--------|
| `SUM(RESERVAS.monto_total)` | Sumatoria del campo `monto_total` de reservas confirmadas/finalizadas | `reservas.estado IN ("Confirmada", "Finalizada")` |
| `COUNT(RESERVAS.id)` | Número de reservas en esos estados | Mismo filtro |

### Precisión

```
2 decimales. Formato: S/ X,XXX.XX
```

### Ejemplo

```
Período: Julio 2026
Suma total vendido: S/ 12,620.10
Número de reservas: 23
TPV = 12,620.10 / 23 = S/ 548.70
```

---

## 4. Tasa de Pérdida por Precio (TPP)

### Fórmula

```
TPP = (PERDIDAS_POR_PRECIO / TOTAL_PERDIDAS) × 100
```

### Definición de Variables

| Variable | Definición |
|----------|-----------|
| `PERDIDAS_POR_PRECIO` | Cotizaciones perdidas cuyo `id_motivo_perdida` corresponde al motivo con código `L-01` ("Precio demasiado alto") |
| `TOTAL_PERDIDAS` | Total de cotizaciones que cambiaron a estado `Perdido` en el período |

### Precisión

```
2 decimales. Formato: XX.XX%
```

### Ejemplo

```
Período: Julio 2026
Perdidas por precio (L-01): 15
Total de pérdidas: 44
TPP = (15 / 44) × 100 = 34.09%
```

---

## 5. Tiempo de Respuesta Promedio (TRP)

### Fórmula

```
TRP = PROMEDIO(DURACION_EN_HORAS)
```

### Definición de Variables

| Variable | Definición |
|----------|-----------|
| `DURACION_EN_HORAS` | `(cotizacion.fecha_envio - lead.fecha_creacion)` en horas |
| Condición | Solo cotizaciones que tengan un `id_lead` asociado (relación lead → cotización) |

### Precisión

```
1 decimal. Formato: X.X horas
```

### Ejemplo

```
Cotización 1: lead creado 10/07 14:00, cotización enviada 10/07 16:30 → 2.5 horas
Cotización 2: lead creado 11/07 09:00, cotización enviada 11/07 09:45 → 0.75 horas
Cotización 3: lead creado 12/07 08:00, cotización enviada 12/07 10:00 → 2.0 horas

TRP = (2.5 + 0.75 + 2.0) / 3 = 1.75 horas
```

---

## 6. Seguimientos Vencidos (SV)

### Fórmula

```
SV = COUNT(seguimientos WHERE estado = "Vencido")
```

### Visualización

- **Número absoluto:** Total de seguimientos vencidos en el período
- **Desglose por vendedor:** Lista de vendedores con sus respectivos conteos
- **Detalle:** Lista de cotizaciones afectadas con enlace directo

### Regla de Vencimiento Automático

```
Un seguimiento PROGRAMADO pasa a VENCIDO cuando:
    CURRENT_TIMESTAMP > fecha_programada + 24 horas
    
El sistema ejecuta esta verificación:
    - Al cargar el dashboard (filtro visual)
    - Mediante un proceso programado que se ejecuta cada hora
```

---

## 7. Resumen de KPIs (Tabla Maestra)

| KPI | Fórmula | Unidad | Frecuencia Recomendada | ¿Desglosable por? |
|-----|---------|--------|----------------------|-------------------|
| TCV | `(Reservas / Cotizaciones Enviadas) × 100` | % | Mensual | Vendedor, Ruta, Canal |
| TPC | `Suma(total cotizado) / N° cotizaciones` | S/ | Mensual | Vendedor, Ruta |
| TPV | `Suma(total vendido) / N° reservas` | S/ | Mensual | Vendedor, Ruta |
| TPP | `(Perdidas por precio / Total perdidas) × 100` | % | Mensual | Vendedor |
| TRP | `Promedio(hora_envío - hora_consulta)` | horas | Mensual | Vendedor, Canal |
| SV | `Count(seguimientos vencidos)` | N° | Semanal | Vendedor |
