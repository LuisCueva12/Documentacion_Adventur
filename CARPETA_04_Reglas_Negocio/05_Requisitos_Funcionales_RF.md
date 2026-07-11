# Requisitos Funcionales del Sistema (RF-01 a RF-20)

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica  
**Fuente:** Documento 2 — Proyecto Sistema de Reservas (Capítulo 8)

---

## Listado de Requisitos Funcionales

| ID | Requisito | Módulo | Prioridad | Dependencia |
|----|-----------|--------|:---------:|-------------|
| RF-01 | El sistema debe permitir el inicio de sesión mediante correo electrónico y contraseña, validando el rol del usuario para restringir el acceso según sus permisos definidos en la matriz RBAC | Seguridad | Crítica | — |
| RF-02 | El sistema debe generar códigos correlativos únicos por año para cada entidad (COT-YYYY-NNNN, ADV-YYYY-NNNN, CLI-YYYY-NNNN, LEAD-YYYY-NNNN) garantizando que no existan duplicados incluso bajo concurrencia | Global | Crítica | Secuencias (E16) |
| RF-03 | El sistema debe permitir buscar clientes por número de documento (DNI, Pasaporte, RUC), teléfono o correo electrónico, mostrando resultados en tiempo real | Clientes | Alta | Clientes (E02) |
| RF-04 | El sistema debe permitir crear reservas en estado "Borrador" para ser completadas posteriormente, sin exigir todos los campos obligatorios desde el inicio | Reservas | Alta | Reservas (E10) |
| RF-05 | El sistema debe permitir registrar múltiples pasajeros por reserva (mínimo 1, sin límite superior explícito), calculando la edad automáticamente desde la fecha de nacimiento | Pasajeros | Alta | Pasajeros (E11) |
| RF-06 | El sistema debe calcular automáticamente la edad del pasajero: `edad = (CURRENT_DATE - fecha_nacimiento).years` cuando se proporciona la fecha de nacimiento | Pasajeros | Media | Pasajeros (E11) |
| RF-07 | El sistema debe registrar información de contacto de emergencia y notas de salud para cada pasajero, siendo obligatorio el contacto y teléfono de emergencia | Pasajeros | Alta | Pasajeros (E11) |
| RF-08 | El sistema debe permitir seleccionar paquetes/servicios turísticos predefinidos (desde plantillas) y agregarlos a la reserva con precios y cantidades | Productos | Alta | Plantillas (E06) + Servicios (E18) |
| RF-09 | El sistema debe gestionar datos de transporte: movilidad asignada, guía asignado y conductor asignado por reserva, con campos editables por Operaciones | Reservas | Alta | Reservas (E10) |
| RF-10 | El sistema debe gestionar datos de alojamiento: nombre del hotel, tipo de habitación, tipo de cama, plan de comidas, fechas de check-in y check-out. Estos campos son condicionales (solo si la ruta lo requiere) | Alojamiento | Media | Alojamientos (E17) |
| RF-11 | El sistema debe calcular automáticamente el precio total de la reserva y el saldo pendiente: `saldo = total - SUM(pagos)`, actualizándose en tiempo real con cada pago registrado o anulado | Pagos | Crítica | Pagos (E12) |
| RF-12 | El sistema debe permitir registrar múltiples pagos parciales por reserva, indicando monto, método de pago, referencia, fecha y comprobante (PDF/JPG/PNG) | Pagos | Alta | Pagos (E12) |
| RF-13 | El sistema debe generar automáticamente un PDF de confirmación de 2 páginas (página 1: datos comerciales y operativos; página 2: políticas y condiciones) con código de verificación único | Documentos | Crítica | PDF Motor |
| RF-14 | El sistema debe permitir descargar, imprimir y compartir (WhatsApp/Email) el PDF de confirmación desde el detalle de la reserva | Documentos | Alta | PDF Motor |
| RF-15 | El sistema debe mostrar un panel de "Próximas salidas" con las reservas confirmadas cuya fecha de viaje esté próxima, ordenadas cronológicamente | Dashboard | Media | Reservas (E10) |
| RF-16 | El sistema debe generar reportes exportables a PDF y Excel con los indicadores de gestión: ventas, conversión, rutas más cotizadas, motivos de pérdida y rendimiento por asesor | Reportes | Alta | Reportes (CARPETA 05) |
| RF-17 | El sistema debe mantener un historial de cambios (auditoría) de todas las acciones sensibles: creación, modificación de estado, envío, aprobación de descuento, registro/anulación de pagos y conversión a reserva | Auditoría | Crítica | Auditoría (E14) |
| RF-18 | El sistema debe permitir la cancelación lógica (anulación) de reservas sin eliminar físicamente el registro, registrando motivo, fecha y usuario que anula | Reservas | Alta | Reservas (E10) |
| RF-19 | El sistema debe contar con políticas configurables (cancelación, pago, responsabilidad, generales) que se inyectan automáticamente en la página 2 del PDF de confirmación | Configuración | Alta | Políticas (E13) |
| RF-20 | El sistema debe implementar control de acceso basado en roles (RBAC) con 6 roles: Administrador, Vendedor, Supervisor, Operaciones, Contabilidad y Gerencia, restringiendo acciones y visibilidad según la matriz definida | Seguridad | Crítica | RBAC (CARPETA 04) |

---

## Trazabilidad RF vs. Carpetas

| RF ID | Cubierto en |
|-------|-------------|
| RF-01 | CARPETA 04/01_Matriz_RBAC.md |
| RF-02 | CARPETA 02/02_Diccionario_Campos.md (E16) |
| RF-03 | CARPETA 01/03_Flujo_5_Pasos_Reservas.md (Paso 1) |
| RF-04 | CARPETA 01/02_Condicion_Conversion_Reserva.md |
| RF-05 | CARPETA 01/03_Flujo_5_Pasos_Reservas.md (Paso 3) |
| RF-06 | CARPETA 02/02_Diccionario_Campos.md (E11 - edad calculada) |
| RF-07 | CARPETA 02/02_Diccionario_Campos.md (E11 - emergencia) |
| RF-08 | CARPETA 02/02_Diccionario_Campos.md (E06, E18) |
| RF-09 | CARPETA 02/02_Diccionario_Campos.md (E10 - movilidad, guía, conductor) |
| RF-10 | CARPETA 02/02_Diccionario_Campos.md (E17) + CARPETA 01/03 (Paso 2 actualizado) |
| RF-11 | CARPETA 03/02_Gestion_Caja_Reservas.md |
| RF-12 | CARPETA 01/03_Flujo_5_Pasos_Reservas.md (Paso 4) |
| RF-13 | CARPETA 05/01_Documento_PDF_2_Paginas.md |
| RF-14 | CARPETA 05/01_Documento_PDF_2_Paginas.md |
| RF-15 | CARPETA 05/02_Reportes_Gestion.md (Dashboard Ejecutivo) |
| RF-16 | CARPETA 05/02_Reportes_Gestion.md |
| RF-17 | CARPETA 04/02_Auditoria_Inmutable.md |
| RF-18 | CARPETA 04/03_Soft_Delete.md |
| RF-19 | CARPETA 05/01_Documento_PDF_2_Paginas.md + CARPETA 02/02 (E13) |
| RF-20 | CARPETA 04/01_Matriz_RBAC.md |
