# Reportes de Control de Gestión

**Carpeta:** 05 — Salidas del Sistema  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Reporte: Ventas vs. Cobrado vs. Pendiente

### Propósito
Mostrar la salud financiera de las reservas en un período determinado.

### Filtros Lógicos

| Filtro | Tipo | ¿Obligatorio? | Opciones |
|--------|------|:-------------:|----------|
| Fecha inicio | Fecha | SÍ | — |
| Fecha fin | Fecha | SÍ | — |
| Vendedor | ID | NO | Un vendedor específico o "Todos" |
| Ruta | ID | NO | Una ruta específica o "Todas" |
| Estado reserva | Lista | NO | "Confirmada", "Finalizada", "Todas las activas" |

### Variables de Salida

| Variable | Fórmula | Tipo |
|----------|---------|------|
| Código de reserva | `reserva.codigo` | Texto(20) |
| Cliente | `cliente.nombre_completo` | Texto(255) |
| Ruta | `ruta.nombre` | Texto(255) |
| Fecha de viaje | `reserva.fecha_viaje` | Fecha |
| Total reservas en período | `COUNT(reservas WHERE fecha_viaje BETWEEN inicio AND fin)` | Entero |
| Monto total vendido | `SUM(reservas.monto_total)` | Decimal(10,2) |
| Monto total cobrado | `SUM(reservas.total_pagado)` | Decimal(10,2) |
| Monto total pendiente | `SUM(reservas.saldo)` | Decimal(10,2) |
| % Cobrado vs. Vendido | `(total_cobrado / total_vendido) * 100` | Decimal(5,2) |
| Desglose por método de pago | `GROUP BY pagos.metodo_pago, SUM(pagos.monto)` | Tabla |

### Formato de Salida

```
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ CÓDIGO   │ CLIENTE  │ RUTA     │ F.VIAJE  │ VENDIDO  │ COBRADO  │ PENDIENTE│ ESTADO   │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ ADV-...  │ Pérez    │ Cumbemayo│15/08/26  │ 611.80   │ 611.80   │   0.00   │ Finaliz. │
│ ADV-...  │ López    │ Granja   │22/08/26  │ 450.00   │ 200.00   │ 250.00   │ Confirm. │
│ ADV-...  │ García   │ Express  │05/08/26  │ 380.00   │ 380.00   │   0.00   │ Finaliz. │
│ ...      │ ...      │ ...      │...       │ ...      │ ...      │ ...      │ ...      │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ TOTALES  │          │          │          │10,450.00 │ 7,800.00 │ 2,650.00 │  74.64%  │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

DESGLOSE POR MÉTODO DE PAGO:
┌──────────────────────┬──────────┬──────────┐
│ MÉTODO               │ MONTO    │    %     │
├──────────────────────┼──────────┼──────────┤
│ Yape                 │ 3,200.00 │  41.03%  │
│ Transferencia        │ 2,500.00 │  32.05%  │
│ Efectivo             │ 1,500.00 │  19.23%  │
│ Plin                 │   600.00 │   7.69%  │
├──────────────────────┼──────────┼──────────┤
│ TOTAL                │ 7,800.00 │ 100.00%  │
└──────────────────────┴──────────┴──────────┘
```

---

## 2. Reporte: Rutas Más Cotizadas

### Propósito
Identificar los paquetes/destinos con mayor demanda comercial para ajustar la oferta.

### Filtros Lógicos

| Filtro | Tipo | ¿Obligatorio? | Opciones |
|--------|------|:-------------:|----------|
| Fecha inicio | Fecha | SÍ | — |
| Fecha fin | Fecha | SÍ | — |
| Top N | Entero | SÍ | 5, 10, 15 o "Todas" |

### Variables de Salida

| Variable | Fórmula |
|----------|---------|
| Ruta | `rutas.nombre` |
| Cotizaciones emitidas | `COUNT(cotizaciones WHERE id_ruta = ruta.id)` |
| Reservas concretadas | `COUNT(reservas WHERE cotizacion.id_ruta = ruta.id)` |
| Tasa de conversión por ruta | `(reservas_concretadas / cotizaciones_emitidas) * 100` |
| Ingreso generado | `SUM(reservas.monto_total WHERE cotizacion.id_ruta = ruta.id)` |

### Formato de Salida

```
RUTAS MÁS COTIZADAS — Julio 2026
┌──────────┬──────────────────────────┬──────────┬──────────┬──────────┬──────────┐
│ #        │ RUTA                     │ COTIZ.   │ RESERV.  │ CONV.%   │ INGRESO  │
├──────────┼──────────────────────────┼──────────┼──────────┼──────────┼──────────┤
│ 1        │ Cumbemayo + Otuzco       │    35    │    12    │  34.29%  │ S/7,340  │
│ 2        │ Cajamarca Express        │    28    │     8    │  28.57%  │ S/3,040  │
│ 3        │ Granja Porcón            │    22    │     3    │  13.64%  │ S/1,350  │
│ 4        │ Ventanillas de Otuzco    │    15    │     2    │  13.33%  │ S/  800  │
│ 5        │ Baños del Inca           │    10    │     1    │  10.00%  │ S/  350  │
├──────────┼──────────────────────────┼──────────┼──────────┼──────────┼──────────┤
│          │ TOTAL                    │   110    │    26    │  23.64%  │S/12,880  │
└──────────┴──────────────────────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 3. Reporte: Motivos de Pérdida de Ventas

### Propósito
Analizar las causas por las que no se cierran las ventas y tomar acciones correctivas.

### Filtros Lógicos

| Filtro | Tipo | ¿Obligatorio? | Opciones |
|--------|------|:-------------:|----------|
| Fecha inicio | Fecha | SÍ | — |
| Fecha fin | Fecha | SÍ | — |
| Vendedor | ID | NO | Un vendedor o "Todos" |
| Motivo de pérdida | ID | NO | Un motivo específico o "Todos" |

### Variables de Salida

| Variable | Fórmula |
|----------|---------|
| Código del motivo | `motivos_perdida.codigo` |
| Motivo de pérdida | `motivos_perdida.nombre` |
| Total de ocurrencias | `COUNT(cotizaciones WHERE id_motivo_perdida = motivo.id)` |
| Porcentaje del total | `(ocurrencias / TOTAL_PERDIDAS) * 100` |
| Monto total perdido | `SUM(cotizaciones.total WHERE perdida)` |

### Formato de Salida

```
MOTIVOS DE PÉRDIDA — Julio 2026
┌──────────┬──────────────────────────────────┬──────────┬──────────┬──────────┐
│ CÓDIGO   │ MOTIVO                           │ OCURR.   │    %     │ MONTO    │
├──────────┼──────────────────────────────────┼──────────┼──────────┼──────────┤
│ L-01     │ Precio demasiado alto            │    15    │  34.09%  │ S/8,250  │
│ L-03     │ Eligió otra agencia              │    12    │  27.27%  │ S/6,400  │
│ L-02     │ No respondió / No contestó       │     8    │  18.18%  │ S/4,920  │
│ L-04     │ Cambió de fecha / pospuso        │     5    │  11.36%  │ S/2,800  │
│ L-05     │ Presupuesto insuficiente         │     3    │   6.82%  │ S/1,950  │
│ L-10     │ Otro (especificar)               │     1    │   2.27%  │ S/  450  │
├──────────┼──────────────────────────────────┼──────────┼──────────┼──────────┤
│          │ TOTAL                            │    44    │ 100.00%  │S/24,770  │
└──────────┴──────────────────────────────────┴──────────┴──────────┴──────────┘
```

---

## 4. Reporte: Rendimiento por Asesor

### Propósito
Evaluar la productividad individual de los vendedores para incentivos y coaching.

### Filtros Lógicos

| Filtro | Tipo | ¿Obligatorio? | Opciones |
|--------|------|:-------------:|----------|
| Fecha inicio | Fecha | SÍ | — |
| Fecha fin | Fecha | SÍ | — |
| Vendedor | ID | NO | Un vendedor específico o "Todos" |

### Variables de Salida

| Variable | Fórmula |
|----------|---------|
| Asesor | `usuarios.nombre` |
| Leads registrados | `COUNT(leads WHERE asignado_a = usuario.id)` |
| Cotizaciones creadas | `COUNT(cotizaciones WHERE asignado_a = usuario.id)` |
| Cotizaciones enviadas | `COUNT(cotizaciones WHERE asignado_a = usuario.id AND estado >= "Enviado")` |
| Reservas concretadas | `COUNT(reservas WHERE creado_por = usuario.id)` |
| Tasa de conversión | `(reservas / cotizaciones_enviadas) * 100` |
| Ingreso generado | `SUM(reservas.monto_total WHERE creado_por = usuario.id)` |
| Ticket promedio vendido | `AVG(reservas.monto_total WHERE creado_por = usuario.id)` |
| Seguimientos vencidos | `COUNT(seguimientos WHERE estado="Vencido" AND creado_por = usuario.id)` |
| Tasa de pérdida | `(cotizaciones_perdidas / cotizaciones_enviadas) * 100` |

### Formato de Salida

```
RENDIMIENTO POR ASESOR — Julio 2026
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ ASESOR   │ LEADS    │ COTIZ.   │ ENVIADAS │ RESERV.  │ CONV.%   │ INGRESO  │ TICKET   │ SEG.VENC │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Pérez,J  │    42    │    38    │    35    │    12    │  34.29%  │ S/7,340  │ S/611.67 │    2     │
│ López,A  │    38    │    32    │    30    │     8    │  26.67%  │ S/4,920  │ S/615.00 │    5     │
│ García,M │    25    │    20    │    18    │     5    │  27.78%  │ S/2,800  │ S/560.00 │    1     │
│ Torres,C │    15    │    12    │    10    │     1    │  10.00%  │ S/  450  │ S/450.00 │    7     │
├──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ TOTAL    │   120    │   102    │    93    │    26    │  27.96%  │S/15,510  │ S/596.54 │   15     │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 5. Dashboard Ejecutivo (Resumen General)

### Propósito
Un solo vistazo con los indicadores más importantes del período actual para la toma de decisiones gerenciales.

### Variables de Salida

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                         DASHBOARD EJECUTIVO                                  ║
║                      Período: Julio 2026                                     ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║   ┌─────────────────────────────────────────────────────────────────────┐   ║
║   │  MÉTRICAS COMERCIALES                                               │   ║
║   ├─────────────────────────────────────────────────────────────────────┤   ║
║   │  Leads registrados:                  120                            │   ║
║   │  Cotizaciones emitidas:              102                            │   ║
║   │  Cotizaciones enviadas:               93                            │   ║
║   │  Tasa de conversión:                 27.96%   ▲ (+2.5% vs mes ant.)│   ║
║   │  Seguimientos vencidos:               15      ⚠️                     │   ║
║   └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                              ║
║   ┌─────────────────────────────────────────────────────────────────────┐   ║
║   │  MÉTRICAS FINANCIERAS                                              │   ║
║   ├─────────────────────────────────────────────────────────────────────┤   ║
║   │  Ingreso total vendido:              S/ 15,510.00                  │   ║
║   │  Ingreso total cobrado:              S/ 11,420.00  (73.63%)        │   ║
║   │  Saldo pendiente por cobrar:         S/  4,090.00                  │   ║
║   │  Ticket promedio cotizado:           S/    425.80                  │   ║
║   │  Ticket promedio vendido:            S/    596.54                  │   ║
║   └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                              ║
║   ┌─────────────────────────────────────────────────────────────────────┐   ║
║   │  MÉTRICAS OPERATIVAS                                               │   ║
║   ├─────────────────────────────────────────────────────────────────────┤   ║
║   │  Reservas confirmadas:               26                            │   ║
║   │  Reservas finalizadas:               18                            │   ║
║   │  Reservas anuladas:                   3                            │   ║
║   │  Tasa de anulación:                 11.54%                         │   ║
║   └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                              ║
║   ┌─────────────────────────────────────────────────────────────────────┐   ║
║   │  TOP 5 RUTAS MÁS VENDIDAS                                          │   ║
║   ├─────────────────────────────────────────────────────────────────────┤   ║
║   │  1. Cumbemayo + Otuzco               S/ 7,340    (47.32%)          │   ║
║   │  2. Cajamarca Express                S/ 3,040    (19.60%)          │   ║
║   │  3. Granja Porcón                    S/ 1,350    ( 8.70%)          │   ║
║   │  4. Ventanillas de Otuzco            S/   800    ( 5.16%)          │   ║
║   │  5. Baños del Inca                   S/   350    ( 2.26%)          │   ║
║   └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                              ║
║   ┌─────────────────────────────────────────────────────────────────────┐   ║
║   │  TOP 3 MOTIVOS DE PÉRDIDA                                          │   ║
║   ├─────────────────────────────────────────────────────────────────────┤   ║
║   │  1. Precio demasiado alto             15 casos    (34.09%)          │   ║
║   │  2. Eligió otra agencia               12 casos    (27.27%)          │   ║
║   │  3. No respondió / No contestó         8 casos    (18.18%)          │   ║
║   └─────────────────────────────────────────────────────────────────────┘   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 6. Formatos de Exportación Soportados

| Formato | Tipo | Uso Principal |
|---------|------|---------------|
| **Tabular (pantalla)** | HTML/CSS estructurado | Visualización en el sistema |
| **PDF** | Documento de 1 página | Impresión / archivo físico |
| **CSV** | Texto delimitado por comas | Procesamiento en Excel/Google Sheets |
| **JSON** | Estructura clave-valor | Integraciones futuras |

### Estructura Común de Exportación

```
REGISTRO_EXPORTACION {
    nombre_reporte:      Texto           // "Ventas vs Cobrado vs Pendiente"
    filtros_aplicados:   {
        fecha_inicio:    Fecha,
        fecha_fin:       Fecha,
        parametros:      { campo: valor, ... }
    }
    fecha_generacion:    FechaHora,
    generado_por:        Texto           // Nombre del usuario que generó el reporte
    total_registros:     Entero,
    datos:               [[Lista de filas con valores]],
    totales:             { campo: valor, ... }   // Sumatorias al pie
}
```
