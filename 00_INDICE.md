# Sistema Adventur Travel — Índice de Documentación Técnica

**Proyecto:** Sistema de Ventas, Cotizaciones, Reservas y Confirmaciones  
**Empresa:** HORIZONTE ANDINO COMPANY E.I.R.L.  
**Versión:** 1.0 — Especificación Agnóstica (Cero Tecnología)  
**Fecha:** Julio 2026

---

## Estructura de la Documentación

```
Docs/
├── CARPETA_01_Arquitectura_Flujos_Estados/     (3 archivos — 634 líneas)
│   ├── 01_Maquina_Estados_Cotizacion.md         — FSM 7 estados, transiciones, visibilidad por rol
│   ├── 02_Condicion_Conversion_Reserva.md       — Regla R1, condiciones lógicas y financieras
│   └── 03_Flujo_5_Pasos_Reservas.md             — Validaciones bloqueantes paso a paso
│
├── CARPETA_02_Diccionario_Datos/                (3 archivos — 589+ líneas)
│   ├── 01_Entidades_Catalogos.md                — 18 entidades (E01..E18), propósitos, ciclos de vida
│   ├── 02_Diccionario_Campos.md                 — Todos los campos, tipos, validaciones (incl. E17, E18)
│   └── 03_Relaciones_Cardinalidad.md            — 1:1, 1:N, autorreferencial, integridad (incl. alojamientos, servicios_reserva)
│
├── CARPETA_03_Motor_Matematico/                 (4 archivos — 663 líneas)
│   ├── 01_Formulas_Cotizador.md                 — SB, SM, MD, TV, PPP con pseudocódigo
│   ├── 02_Gestion_Caja_Reservas.md              — Saldo en tiempo real, adelanto mínimo
│   ├── 03_KPI_Gerenciales.md                    — TCV, TPC, TPV, TPP, TRP, SV
│   └── 04_Regla_Control_Decimal.md              — TRUNC(2), garantía de cero descuadres
│
├── CARPETA_04_Reglas_Negocio/                   (6 archivos — 621+ líneas)
│   ├── 01_Matriz_RBAC.md                        — 6 roles × 15 entidades, límites de descuento
│   ├── 02_Auditoria_Inmutable.md                — 14 eventos auditables, estructura inmutable
│   ├── 03_Soft_Delete.md                        — Prohibición de DELETE físico, cascada lógica
│   ├── 04_Descuentos_Vencimientos.md            — 9 reglas de bloqueo DB-01 a DB-09
│   ├── 05_Requisitos_Funcionales_RF.md          — RF-01 a RF-20 con trazabilidad a carpetas
│   └── 06_Requisitos_No_Funcionales_NF.md       — NF-01 a NF-08 con criterios y verificación
│
├── CARPETA_05_Salidas_Sistema/                  (3 archivos — 563+ líneas)
│   ├── 01_Documento_PDF_2_Paginas.md            — Estructura de datos + maquetas P1 y P2
│   ├── 02_Reportes_Gestion.md                   — 4 reportes + dashboard, filtros, formatos
│   └── 03_Criterios_Aceptacion.md               — CA-01 a CA-10, checklist pre-producción, regla final
│
└── CARPETA_06_ARQUITECTURA_TECNICA/             (5 archivos)
    ├── 00_Vision_General.md                     — Stack, principios, diagrama general, escalabilidad
    ├── 01_Backend_Laravel_Hexagonal.md          — Hexagonal + DDD, capas, repositorios, casos de uso
    ├── 02_API_REST_Endpoints.md                 — Endpoints, ejemplos request/response, códigos de error
    ├── 03_Frontend_Vue_Vite.md                  — Componentes, rutas, stores, TanStack Query, PrimeVue
    ├── 04_Flujo_Datos.md                        — Flujo completo request→response, máquinas de estado
    └── 05_Despliegue.md                         — SiteGround → VPS, backup, CI/CD, costos
```

**Total: 24 archivos — 3,070+ líneas de especificación agnóstica + arquitectura técnica**

---

## Mapa de Módulos vs. Carpetas

| Módulo del Sistema | Cubierto en Carpeta |
|-------------------|---------------------|
| Dashboard | CARPETA 03 (KPIs) + CARPETA 05 (Dashboard Ejecutivo) |
| Leads / Clientes | CARPETA 01 (flujo) + CARPETA 02 (diccionario) |
| Cotizaciones | CARPETA 01 (FSM) + CARPETA 03 (fórmulas) + CARPETA 04 (reglas) |
| Plantillas | CARPETA 02 (entidad) |
| Reservas | CARPETA 01 (5 pasos) + CARPETA 03 (caja) |
| Pasajeros | CARPETA 01 (paso 3) + CARPETA 02 (diccionario) |
| Pagos | CARPETA 03 (gestión caja, control decimal) |
| Reportes | CARPETA 03 (KPIs) + CARPETA 05 (reportes, dashboard) |
| Configuración | CARPETA 04 (RBAC, auditoría) |
| Documentos PDF | CARPETA 05 (estructura, maquetas) |
| Arquitectura Técnica | CARPETA 06 (stack, endpoints, frontend, flujo, despliegue) |

---

## Reglas Inmutables del Sistema

| ID | Regla | Descrita en |
|----|-------|-------------|
| R1 | Separación estricta Cotización (propuesta) vs. Reserva (compromiso) | 01/02 |
| R2 | Truncamiento a 2 decimales en TODAS las operaciones monetarias | 03/04 |
| R3 | Auditoría de solo inserción, NUNCA modificación | 04/02 |
| R4 | Prohibición de eliminación física (soft delete obligatorio) | 04/03 |

---

## Documentos Fuente

- `Proyecto_Detallado_Software_Ventas_Cotizaciones_Adventur-2026.pdf`
- `Proyecto_Sistema_Reservas_Adventur- 2026.pdf`

Ambos documentos aprobados por el cliente (Horizonte Andino Company E.I.R.L.). Esta especificación técnica es la interpretación agnóstica (libre de tecnología) de los requisitos contenidos en dichos documentos.
