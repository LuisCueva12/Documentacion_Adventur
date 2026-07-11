# Criterios de Aceptación del Sistema (CA-01 a CA-10)

**Carpeta:** 05 — Salidas del Sistema  
**Versión:** 1.0 — Especificación Agnóstica  
**Fuente:** Documento 2 — Proyecto Sistema de Reservas (Capítulo 16)

---

## Criterios de Aceptación

El sistema se considera **ACEPTADO para producción** cuando cumple **TODOS** los siguientes criterios:

| ID | Criterio | Módulo | Tipo | ¿Cómo se Verifica? |
|----|----------|--------|------|--------------------|
| CA-01 | **Reserva completa sin herramientas externas:** El sistema permite crear una reserva completa (titular + viaje + pasajeros + pago + confirmación) sin usar Word, Excel ni ningún programa externo | Reservas | Funcional | Prueba UAT: vendedor crea reserva de principio a fin exclusivamente en el sistema |
| CA-02 | **Códigos correlativos únicos por año:** Los códigos COT, ADV, CLI y LEAD son únicos, secuenciales y se reinician automáticamente al cambiar de año calendario. No existen duplicados | Global | Técnico | Prueba: crear 100 registros concurrentes. Verificar que no existan 2 códigos iguales. Verificar reinicio el 1 de enero |
| CA-03 | **Cálculo correcto de pagos y saldo:** Los montos pagados y el saldo pendiente se calculan correctamente sin errores decimales. La suma de pagos parciales más el saldo siempre iguala el total de la reserva | Pagos | Matemático | Prueba: registrar 3 pagos parciales, anular 1, verificar que `total_pagado + saldo = total_reserva` se cumple siempre |
| CA-04 | **PDF sin errores visuales:** El PDF de confirmación de 2 páginas reproduce todos los campos aprobados sin textos superpuestos, cortados o desbordados. El formato es consistente con la identidad corporativa (logo, colores, tipografía) | Documentos | Visual | Revisión visual de 10 PDFs generados con diferentes cantidades de datos (1 pasajero, 10 pasajeros, textos largos) |
| CA-05 | **Control de acceso por roles:** Los usuarios solo visualizan y modifican la información permitida según su rol definido en la matriz RBAC. Un vendedor no ve datos de otros vendedores. Operaciones solo ve reservas confirmadas | Seguridad | RBAC | Prueba: cada rol intenta acceder a funcionalidades no permitidas. Verificar error 403 o dato oculto |
| CA-06 | **Reportes coinciden con datos reales:** Los reportes de ventas, conversión, rutas y rendimiento muestran datos que coinciden exactamente con los registros del sistema consultados directamente | Reportes | Integridad | Prueba: generar reporte y comparar 10 datos aleatorios contra consulta directa a los registros |
| CA-07 | **Funcionamiento responsive:** El sistema funciona correctamente en computadora (1920×1080), tablet (768×1024) y celular (375×667). Todos los botones y formularios son accesibles y usables en los 3 tamaños | UX | Técnico | Prueba en 3 tamaños de pantalla × 4 navegadores = 12 combinaciones. Sin elementos superpuestos ni botones inaccesibles |
| CA-08 | **Backup y recovery probados:** Existe un procedimiento de backup automático diario documentado y probado. Se realizó una restauración exitosa en entorno de staging a partir del backup | DevOps | Infraestructura | Prueba: ejecutar backup, eliminar datos, restaurar desde backup, verificar integridad de datos |
| CA-09 | **Capacitación realizada:** Se realizaron mínimo 2 sesiones de capacitación (1 para vendedores/operaciones, 1 para administradores/gerencia). Se entregaron credenciales de administrador y manual de usuario | Proyecto | Documentación | Asistencia registrada. Manual entregado. Credenciales de admin entregadas y verificadas |
| CA-10 | **Documentación completa entregada:** Se entregó el código fuente, la estructura completa de la base de datos (DDL), el diccionario de datos y la documentación técnica completa del sistema (5 carpetas de especificación) | Proyecto | Documentación | Verificar existencia de todos los artefactos: código, DDL, diccionario, especificaciones |

---

## Matriz de Cumplimiento (Checklist Pre-Producción)

| CA ID | Estado | Observaciones | Fecha de Verificación | Verificador |
|-------|:------:|---------------|:---------------------:|:-----------:|
| CA-01 | ⬜ Pendiente | | | |
| CA-02 | ⬜ Pendiente | | | |
| CA-03 | ⬜ Pendiente | | | |
| CA-04 | ⬜ Pendiente | | | |
| CA-05 | ⬜ Pendiente | | | |
| CA-06 | ⬜ Pendiente | | | |
| CA-07 | ⬜ Pendiente | | | |
| CA-08 | ⬜ Pendiente | | | |
| CA-09 | ⬜ Pendiente | | | |
| CA-10 | ⬜ Pendiente | | | |

---

## Regla de Aceptación Final

```
ACEPTACION_FINAL ↔ CA-01 AND CA-02 AND CA-03 AND CA-04 AND CA-05
                   AND CA-06 AND CA-07 AND CA-08 AND CA-09 AND CA-10
```

**Todos los criterios deben cumplirse.** No existe aceptación parcial. Si algún criterio no se cumple, el sistema no puede pasar a producción.
