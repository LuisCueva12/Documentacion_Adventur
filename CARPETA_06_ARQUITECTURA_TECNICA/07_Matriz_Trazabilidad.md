# Matriz de Trazabilidad — Reglas de Negocio ↔ Implementación Técnica

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 07 — Matriz de Trazabilidad  
**Versión:** 1.0 — Conexión Agnóstico ↔ Técnico

---

## 1. Reglas Inmutables (R1–R4)

| ID | Regla | Documento Agnóstico | Capa | Implementación Técnica | Archivo/Clase |
|----|-------|---------------------|------|----------------------|---------------|
| R1 | Separación Cotización vs Reserva | CARPETA_01/02 — Condición Conversión | Dominio | `InterfazRepositorioCotizacion` verifica que no exista reserva previa (`id_reserva IS NULL`). ValueObject `EstadoCotizacion` impide transición a `Convertido` si no está `Aprobado`. El caso de uso valida C1–C5 antes de convertir. | `app/Core/Dominio/Reglas/ReglaR1SeparacionCotizacionReserva.php`, `app/Core/Dominio/ObjetosValor/EstadoCotizacion.php`, `app/Core/Aplicacion/CasosUso/Cotizaciones/ConvertirCotizacionAReservaUseCase.php` |
| R2 | Truncamiento 2 decimales | CARPETA_03/04 — Regla Control Decimal | Dominio | ValueObject `Precio` trunca en constructor con `bcdiv($valor, 1, 2)`. Todos los campos `NUMERIC(10,2)` en PostgreSQL. Toda operación monetaria pasa por `Precio::truncar()`. | `app/Core/Dominio/ObjetosValor/Precio.php`, migraciones PostgreSQL (`NUMERIC(10,2)`) |
| R3 | Auditoría solo INSERT | CARPETA_04/02 — Auditoría Inmutable | Infraestructura | Listener de eventos de dominio ejecuta solo `INSERT` en tabla `auditoria`. La tabla no tiene triggers de UPDATE/DELETE. No se usa `SoftDeletes` en el modelo `Auditoria`. Particionamiento por año en PostgreSQL. | `app/Infraestructura/Persistencia/Repositorios/RepositorioAuditoriaEloquent.php`, `app/Infraestructura/Listeners/RegistrarAuditoriaListener.php`, base de datos tabla `auditoria` |
| R4 | Soft delete obligatorio | CARPETA_04/03 — Soft Delete | Infraestructura | Todas las entidades protegidas tienen campo `fecha_eliminacion TIMESTAMP DEFAULT NULL`. Repositorios filtran con `WHERE fecha_eliminacion IS NULL`. Modelos Eloquent con `SoftDeletes` trait. RLS opcional en PostgreSQL. | `app/Infraestructura/Persistencia/Eloquent/Modelos/` (todos los modelos), migraciones con `fecha_eliminacion`, scopes globales en repositorios |

---

## 2. Requisitos Funcionales (RF-01 a RF-20)

| RF ID | Requisito | Endpoint API | Caso de Uso | Controlador | Validación Adicional |
|-------|-----------|-------------|-------------|-------------|---------------------|
| RF-01 | Inicio de sesión con email y contraseña | `POST /api/auth/login` | `AutenticarUsuarioUseCase` | `AuthControlador@login` | Sanctum middleware, bcrypt en `Usuario::contrasena`, FormRequest `LoginPeticion` |
| RF-02 | Códigos correlativos únicos por año | Interno (llamado desde otros CU) | `GenerarSecuenciaCodigoUseCase` | — | Transacción con `SELECT ... FOR UPDATE`, secuencia `uq_secuencias_prefijo_anio`, reinicio automático por año |
| RF-03 | Buscar clientes por documento | `GET /api/clientes/buscar?q=` | `BuscarClienteUseCase` | `ClienteControlador@buscar` | LIKE con índices GIN y `varchar_pattern_ops`, debounce 300ms frontend, máx 10 resultados |
| RF-04 | Crear reservas en Borrador | `POST /api/reservas` | `CrearReservaUseCase` | `ReservaControlador@store` | Estado inicial = `Borrador`, validación `CrearReservaPeticion` |
| RF-05 | Registrar múltiples pasajeros por reserva | `POST /api/reservas` (incluye array pasajeros) | `CrearReservaUseCase` | `ReservaControlador@store` | Mínimo 1 pasajero, `CHK_reservas_num_pasajeros >= 1`, edad calculada automática |
| RF-06 | Calcular edad automática desde fecha nacimiento | `POST /api/reservas` (campo calculado) | `CrearReservaUseCase` | `ReservaControlador@store` | `edad = (CURRENT_DATE - fecha_nacimiento).years`, se ejecuta en ValueObject o helper de dominio |
| RF-07 | Contacto de emergencia y notas salud obligatorios | `POST /api/reservas` | `CrearReservaUseCase` | `ReservaControlador@store` | FormRequest valida `contacto_emergencia` y `telefono_emergencia` NOT NULL en cada pasajero |
| RF-08 | Seleccionar paquetes/servicios desde plantillas | `POST /api/cotizaciones` (incluye `id_plantilla`) | `CrearCotizacionUseCase` | `CotizacionControlador@store` | Validar `plantillas.activa = TRUE`, cascade de valores por defecto desde plantilla |
| RF-09 | Gestionar datos de transporte (movilidad, guía, conductor) | `PUT /api/reservas/{id}` | `ActualizarReservaUseCase` | `ReservaControlador@update` | Campos editables solo por Operaciones/Admin, validación de permisos en UseCase |
| RF-10 | Gestionar datos de alojamiento (condicional) | `PUT /api/reservas/{id}` (array alojamientos) | `ActualizarReservaUseCase` | `ReservaControlador@update` | `alojamiento_requerido` condiciona obligatoriedad, validación en FormRequest |
| RF-11 | Calcular saldo pendiente en tiempo real | `POST /api/reservas/{id}/pagos` | `RegistrarPagoUseCase` | `ReservaControlador@registrarPago` | `saldo = TRUNC(monto_total - SUM(pagos.monto WHERE fecha_anulacion IS NULL), 2)`, evento `PagoRegistrado` dispara recálculo |
| RF-12 | Registrar múltiples pagos parciales | `POST /api/reservas/{id}/pagos` | `RegistrarPagoUseCase` | `ReservaControlador@registrarPago` | FormRequest valida monto, método, referencia, comprobante (PDF/JPG/PNG ≤ 5MB) |
| RF-13 | Generar PDF de confirmación 2 páginas | `GET /api/reservas/{id}/pdf` | `GenerarPDFReservaUseCase` | `ReservaControlador@pdf` | Servicio `InterfazServicioPdf` (DomPdf), vista blade para página 1 y página 2, código QR de verificación |
| RF-14 | Descargar, imprimir y compartir PDF | Frontend desde detalle reserva | `GenerarPDFReservaUseCase` | `ReservaControlador@pdf` | Botón descarga + compartir vía Web Share API, `window.open` para impresión |
| RF-15 | Panel "Próximas salidas" en dashboard | `GET /api/dashboard/resumen` | `ObtenerDashboardResumenUseCase` | `DashboardControlador@resumen` | `WHERE estado = 'Confirmada' AND fecha_viaje BETWEEN CURRENT_DATE AND CURRENT_DATE + 30`, orden cronológico |
| RF-16 | Reportes exportables (PDF/Excel) | `GET /api/reportes/ventas`, `/conversion`, `/rendimiento` | `GenerarReporteVentasUseCase`, `GenerarReporteConversionUseCase` | `ReporteControlador@ventas`, `@conversion`, `@rendimiento` | Exportación con `maatwebsite/laravel-excel`, PDF con DomPdf, gráficos Chart.js en frontend |
| RF-17 | Historial de auditoría de acciones sensibles | `GET /api/auditoria` | `RegistrarAuditoriaUseCase` | `AuditoriaControlador@index` | Listeners de eventos de dominio, tabla `auditoria` solo INSERT, particionada por año |
| RF-18 | Cancelación lógica (anulación) de reservas | `DELETE /api/reservas/{id}` | `AnularReservaUseCase` | `ReservaControlador@destroy` | Soft delete + registro motivo/fecha/usuario, transición a estado `Anulada` |
| RF-19 | Políticas configurables en PDF | `GET/PUT /api/v1/admin/configuracion/politicas` | `ServicioPoliticasDocumento` | `AdminControlador@listarPoliticas` / `@actualizarPoliticas` | Cuatro políticas documentales ordenadas, edición versionada, auditoría e inyección filtrada en página 2; configuraciones numéricas quedan excluidas |
| RF-20 | Control de acceso RBAC (6 roles) | Middleware en rutas + frontend guards | `VerificarPermisoUseCase` | Middleware `RoleMiddleware` | `spatie/laravel-permission` o tabla `rol_permiso`, Vue Router guards, `v-if` en componentes |

---

## 3. Requisitos No Funcionales (NF-01 a NF-08)

| NF ID | Requisito | Categoría | ¿Dónde se Implementa? | ¿Cómo se Verifica? |
|-------|-----------|-----------|----------------------|-------------------|
| NF-01 | Seguridad de acceso: contraseñas cifradas, sesión con expiración 30 min, HTTPS | Seguridad | Sanctum middleware + bcrypt en `Usuario::contrasena` + `SESSION_LIFETIME=30` + Nginx SSL/TLS + CORS config | Pruebas de penetración, revisión de código, verificación de hash bcrypt en BD |
| NF-02 | Disponibilidad 24/7 con ventana de mantenimiento ≤ 2h semanales | Disponibilidad | Nginx + PHP-FPM + Supervisor (workers), sin single point of failure en Fase 2 | Monitoreo de uptime (Pingdom/UptimeRobot), alertas de caída |
| NF-03 | Rendimiento: carga < 3s, persistencia < 1s | Rendimiento | TanStack Query (caching 30s), índices en BD, eager loading, paginación, colas Redis (Fase 2) | Pruebas de carga con K6/JMeter, Laravel Debugbar en staging |
| NF-04 | Usabilidad: flujo guiado 5 pasos, ≤ 10 clics cotización desde lead | UX | Componente `Flujo5Pasos.vue` con PrimeVue Stepper, búsqueda rápida de clientes con debounce | Pruebas UAT cronometradas, grabación de sesiones de usuario |
| NF-05 | Compatibilidad: Chrome, Firefox, Edge, Safari últimas 2 versiones, responsive | Compatibilidad | Tailwind CSS responsive, PrimeVue DataTable responsive, prueba en 3 tamaños × 4 navegadores | Pruebas en matriz de 12 combinaciones, BrowserStack |
| NF-06 | Escalabilidad: multi-tenancy liviano, 100k cotizaciones + 50k reservas sin degradación | Escalabilidad | Tabla `agencias` preparada, índices compuestos, particionamiento por año, eager loading controlado | Prueba de carga con 100k registros, monitoreo de tiempos de respuesta < 5s |
| NF-07 | Backup diario automático con retención 30 días, procedimiento de restauración documentado | Backup | Script `backup-adventur.sh` vía cron diario 03:00 AM, `pg_dump` + `tar.gz` de storage | Prueba de restore en staging mensual, verificación de integridad con `php artisan adventur:verificar-integridad` |
| NF-08 | Privacidad: datos personales solo por roles autorizados, auditoría de operaciones sensibles | Privacidad | RBAC por endpoint + `spatie/laravel-permission`, scopes de visibilidad (`asignado_a`, `id_supervisor`), campos ocultos en API Resources | Revisión de permisos por rol, verificación de datos personales en API Responses |

---

## 4. Criterios de Aceptación (CA-01 a CA-10)

| CA ID | Criterio | Tipo | Prueba | ¿Dónde se Verifica? |
|-------|----------|------|--------|---------------------|
| CA-01 | Reserva completa sin herramientas externas | Funcional | Test E2E de flujo completo: login → buscar cliente → crear cotización → aprobar → convertir → registrar pasajeros → pago → confirmar → descargar PDF | `tests/Feature/FlujoCompletoReservaTest.php`, Playwright E2E |
| CA-02 | Códigos correlativos únicos por año, sin duplicados, reinicio anual | Técnico | Prueba concurrente: lanzar 100 requests simultáneas de creación, verificar unicidad de códigos | `tests/Unit/Aplicacion/GenerarSecuenciaCodigoUseCaseTest.php`, prueba de concurrencia con `sync` |
| CA-03 | Cálculo correcto de pagos y saldo, sin errores decimales | Matemático | Registrar 3 pagos parciales, anular 1, verificar `total_pagado + saldo = monto_total` | `tests/Unit/Dominio/PrecioTest.php`, `tests/Feature/RegistrarPagoTest.php` |
| CA-04 | PDF sin errores visuales, formato consistente | Visual | Generar 10 PDFs con datos variables (1 a 10 pasajeros), revisión visual de superposición y desborde | Prueba visual manual + test de regresión visual con Puppeteer/Playwright |
| CA-05 | Control de acceso por roles: 403 en acciones no permitidas | RBAC | Cada rol intenta acceder a funcionalidades fuera de su matriz, verificar error 403 | `tests/Feature/Authorization/RbacTest.php`, pruebas de middleware por endpoint |
| CA-06 | Reportes coinciden exactamente con datos del sistema | Integridad | Generar reporte, comparar 10 datos aleatorios contra consulta SQL directa | `tests/Feature/Reportes/CoincidenciaDatosTest.php` |
| CA-07 | Funcionamiento responsive en 3 tamaños × 4 navegadores | Técnico | Prueba en 12 combinaciones sin errores de renderizado ni elementos inaccesibles | Playwright matrix tests, BrowserStack |
| CA-08 | Backup y recovery probados exitosamente | Infraestructura | Backup → eliminar datos → restaurar → verificar integridad | Prueba manual mensual en staging, script `verificar-integridad` |
| CA-09 | Capacitación realizada (2 sesiones mínimas) | Documentación | Asistencia registrada, manual entregado, credenciales de admin verificadas | Checklist pre-producción (documento externo) |
| CA-10 | Documentación completa entregada (código, DDL, diccionario, 5 carpetas) | Documentación | Verificar existencia de todos los artefactos en el repositorio | Checklist en `Docs/00_INDICE.md`, revisión de cobertura de documentación |

---

## 5. Máquina de Estados — Cotización (FSM)

| Estado | Transiciones Permitidas | Endpoint | Caso de Uso | Regla Validada |
|--------|------------------------|----------|-------------|---------------|
| Propuesta | → Enviado, Perdido | `POST /api/cotizaciones/{id}/enviar` | `EnviarCotizacionUseCase` | Solo vendedor asignado o admin/supervisor, cotización no vencida |
| Enviado | → Visto, Perdido | (interno, cuando cliente abre link) / `POST /api/cotizaciones/{id}/perder` | `RegistrarVistoCotizacionUseCase` / `PerderCotizacionUseCase` | Token de visualización único, motivo obligatorio si se pierde |
| Visto | → Aprobado, Perdido | `POST /api/cotizaciones/{id}/aprobar` / `POST /api/cotizaciones/{id}/perder` | `AprobarCotizacionUseCase` / `PerderCotizacionUseCase` | Límite de descuento por rol (DB-01/02/03), cotización no vencida |
| Aprobado | → Convertido, Perdido | `POST /api/cotizaciones/{id}/convertir` / `POST /api/cotizaciones/{id}/perder` | `ConvertirCotizacionAReservaUseCase` / `PerderCotizacionUseCase` | R1 (no reserva previa), C1–C5, cotización no vencida |
| Convertido | (terminal) | — | — | Una vez convertida no admite más transiciones |
| Perdido | (terminal — reactivable solo por Admin/Supervisor) | `POST /api/cotizaciones/{id}/reactivar` | `ReactivarCotizacionUseCase` | Solo Admin/Supervisor, motivo de reactivación obligatorio, genera auditoría A10 |

---

## 6. Máquina de Estados — Reserva (FSM)

| Estado | Transiciones Permitidas | Condición | Endpoint | Caso de Uso |
|--------|------------------------|-----------|----------|-------------|
| Borrador | → Pendiente | Datos completos (pasajeros, viaje, alojamiento si aplica) | `PUT /api/reservas/{id}` | `ActualizarReservaUseCase` |
| Pendiente | → Confirmada, Anulada | Adelanto ≥ 30% del total o anulación con motivo | `POST /api/reservas/{id}/pagos` | `RegistrarPagoUseCase` (confirma automáticamente si alcanza 30%) |
| Pendiente | → Anulada | Motivo de anulación obligatorio | `DELETE /api/reservas/{id}` | `AnularReservaUseCase` |
| Confirmada | → Finalizada, Anulada | Viaje ya realizado (cron diario 06:00) o anulación con >48h antes del viaje | Interno (cron) `php artisan reservas:finalizar` / `DELETE /api/reservas/{id}` | `FinalizarReservaUseCase` / `AnularReservaUseCase` |
| Finalizada | (terminal) | — | — | — |
| Anulada | (terminal) | — | — | — |

---

## 7. Módulos vs Capas

| Módulo | Entidades de Dominio | Repositorio (Interfaz) | Casos de Uso | Controlador API | Frontend (Vue) |
|--------|---------------------|------------------------|-------------|-----------------|----------------|
| **Cotizaciones** | `Cotizacion`, `CotizacionDetalle`, `Seguimiento` | `InterfazRepositorioCotizacion`, `InterfazRepositorioSeguimiento` | `CrearCotizacionUseCase`, `EnviarCotizacionUseCase`, `AprobarCotizacionUseCase`, `RechazarCotizacionUseCase`, `PerderCotizacionUseCase`, `ConvertirCotizacionAReservaUseCase`, `ListarCotizacionesUseCase`, `ObtenerCotizacionUseCase`, `ReactivarCotizacionUseCase` | `CotizacionControlador` | `vistas/cotizaciones/`, `componentes/modulos/cotizaciones/` |
| **Reservas** | `Reserva`, `Pasajero`, `Pago`, `Alojamiento`, `ServicioReserva` | `InterfazRepositorioReserva`, `InterfazRepositorioPasajero`, `InterfazRepositorioPago`, `InterfazRepositorioAlojamiento`, `InterfazRepositorioServicioReserva` | `CrearReservaUseCase`, `ActualizarReservaUseCase`, `AnularReservaUseCase`, `RegistrarPagoUseCase`, `AnularPagoUseCase`, `GenerarPDFReservaUseCase`, `ListarReservasUseCase`, `ObtenerReservaUseCase`, `FinalizarReservaUseCase` | `ReservaControlador` | `vistas/reservas/`, `componentes/modulos/reservas/` |
| **Clientes** | `Cliente` | `InterfazRepositorioCliente` | `CrearClienteUseCase`, `BuscarClienteUseCase`, `ObtenerClienteUseCase` | `ClienteControlador` | `vistas/clientes/`, `componentes/modulos/clientes/` |
| **Leads** | `Lead` | `InterfazRepositorioLead` | `CrearLeadUseCase`, `ConvertirLeadEnClienteUseCase`, `ListarLeadsUseCase` | `LeadControlador` | `vistas/leads/` |
| **Seguridad/Auth** | `Usuario`, `Rol`, `Permiso` | `InterfazRepositorioUsuario`, `InterfazRepositorioRol` | `AutenticarUsuarioUseCase`, `CerrarSesionUseCase`, `ObtenerPerfilUseCase` | `AuthControlador` | `vistas/login/`, `stores/authStore.js`, `router/guardias.js` |
| **Global/Infra** | `SecuenciaCodigo`, `Auditoria` | `InterfazRepositorioSecuenciaCodigo`, `InterfazRepositorioAuditoria` | `GenerarSecuenciaCodigoUseCase`, `RegistrarAuditoriaUseCase` | — (llamado internamente) | — |
| **Reportes** | — (solo consultas) | — | `GenerarReporteVentasUseCase`, `GenerarReporteConversionUseCase`, `GenerarReporteRendimientoUseCase` | `ReporteControlador` | `vistas/reportes/`, `componentes/charts/` |
| **Dashboard** | — (solo consultas) | — | `ObtenerDashboardResumenUseCase` | `DashboardControlador` | `vistas/dashboard/`, `componentes/modulos/dashboard/` |
| **Admin/Config** | `PoliticaConfiguracion`, `Plantilla`, `MotivoPerdida`, `CanalOrigen` | `InterfazRepositorioPoliticaConfiguracion`, `InterfazRepositorioPlantilla`, `InterfazRepositorioMotivoPerdida` | `ServicioPoliticasDocumento` y gestión administrativa | `AdminControlador` | `src/views/Administracion/PoliticasDocumento.vue`, `UsuariosLista.vue` |

---

## 8. Eventos de Dominio vs Listeners

| Evento | Disparado por | Listener | Efecto |
|--------|---------------|----------|--------|
| `CotizacionCreada` | `CrearCotizacionUseCase` | `RegistrarAuditoriaListener` | INSERT en auditoría (R3) |
| `CotizacionEnviada` | `EnviarCotizacionUseCase` | `CrearSeguimientoListener` | Crea seguimiento "Programado" para 3 días después |
| `CotizacionVista` | `RegistrarVistoCotizacionUseCase` | `ActualizarSeguimientoListener` | Actualiza seguimiento a "Realizado" |
| `CotizacionAprobada` | `AprobarCotizacionUseCase` | `NotificarVendedorListener`, `RegistrarAuditoriaListener` | Notifica al vendedor, registra auditoría A05 |
| `CotizacionConvertida` | `ConvertirCotizacionAReservaUseCase` | `RegistrarAuditoriaListener` | Registra auditoría A06 |
| `CotizacionPerdida` | `PerderCotizacionUseCase` | `CerrarSeguimientosListener`, `RegistrarAuditoriaListener` | Cierra seguimientos activos, registra auditoría |
| `ReservaCreada` | `CrearReservaUseCase` | `RegistrarAuditoriaListener` | INSERT en auditoría |
| `ReservaConfirmada` | `RegistrarPagoUseCase` (automático) | `GenerarPDFListener`, `EnviarCorreoConfirmacionListener` | Genera PDF, envía email al cliente |
| `ReservaAnulada` | `AnularReservaUseCase` | `RegistrarAuditoriaListener` | Registra auditoría A09 |
| `PagoRegistrado` | `RegistrarPagoUseCase` | `VerificarSaldoListener`, `RegistrarAuditoriaListener` | Verifica si saldo = 0 → confirma reserva, registra auditoría A07 |
| `PagoAnulado` | `AnularPagoUseCase` | `RecalcularSaldoListener`, `RegistrarAuditoriaListener` | Recalcula saldo, revierte confirmación si saldo < 30%, registra auditoría A08 |
| `ClienteCreado` | `CrearClienteUseCase` | `RegistrarAuditoriaListener` | Registra auditoría |

---

## 9. ValueObjects del Dominio

| ValueObject | Propósito | ¿Dónde se usa? | Regla que implementa |
|-------------|-----------|----------------|----------------------|
| `Precio` | Trunca a 2 decimales sin redondear | Todos los campos monetarios | R2 |
| `CodigoCorrelativo` | Genera y valida formato `XXX-YYYY-NNNN` | Cotizaciones, Reservas, Clientes, Leads | RF-02 |
| `EstadoCotizacion` | ENUM con transiciones válidas | Cotizaciones | FSM Cotización |
| `EstadoReserva` | ENUM con transiciones válidas | Reservas | FSM Reserva |
| `Email` | Validación de formato email | Usuarios, Clientes | — |
| `DocumentoIdentidad` | Validación según tipo (DNI, RUC, CE, Pasaporte) | Clientes, Pasajeros | RF-03 |
| `Rol` | ENUM con niveles de acceso | Usuarios | RF-20, NF-08 |
| `Porcentaje` | Validación 0–100% | Descuentos, márgenes | DB-01/02/03 |
| `Telefono` | Validación formato telefónico | Clientes, Pasajeros | — |

---

## 10. Excepciones de Negocio

| Excepción | Código HTTP | Código Interno | Disparada por | Ejemplo |
|-----------|-------------|----------------|---------------|---------|
| `ExcepcionNegocio` | 409 | `REGLA_NEGOCIO` | Violación de regla de negocio R1–R4 | "Esta cotización ya fue convertida a la reserva ADV-2026-0047" |
| `ExcepcionTransicionEstadoInvalida` | 409 | `TRANSICION_INVALIDA` | FSM Cotización/Reserva | "No se puede pasar de Propuesta a Aprobado" |
| `ExcepcionNoEncontrado` | 404 | `NO_ENCONTRADO` | Entidad no existe en BD | "Cotización con ID 999 no encontrada" |
| `ExcepcionNoAutorizado` | 403 | `NO_AUTORIZADO` | Rol sin permiso | "Solo los Supervisores pueden aprobar cotizaciones" |
| `ExcepcionValidacion` | 422 | `VALIDACION` | Datos inválidos en FormRequest | "El campo email debe ser un correo válido" |
| `ExcepcionDescuentoExcedido` | 403 | `DESCUENTO_EXCEDIDO` | Límite de descuento superado | "El descuento (15%) excede el límite del vendedor (10%)" |
| `ExcepcionClienteInactivo` | 409 | `CLIENTE_INACTIVO` | Cliente con soft delete | "El cliente seleccionado no está activo" |
| `ExcepcionCotizacionYaConvertida` | 409 | `COTIZACION_YA_CONVERTIDA` | R1 | "Esta cotización ya generó la reserva ADV-2026-0047" |
| `ExcepcionReservaNoModificable` | 409 | `RESERVA_NO_MODIFICABLE` | Estado no permite modificación | "No se puede modificar una reserva Finalizada" |
| `ExcepcionSaldoInsuficiente` | 409 | `SALDO_INSUFICIENTE` | Adelanto mínimo no cubierto | "El adelanto (S/100.00) no cubre el mínimo requerido (S/375.00)" |
| `ExcepcionPagoAnulado` | 409 | `PAGO_ANULADO` | Operación sobre pago anulado | "No se puede modificar un pago anulado" |

---

## 11. Tareas Programadas (Cron)

| Tarea Artisan | Frecuencia | Descripción | Caso de Uso Asociado |
|---------------|-----------|-------------|----------------------|
| `seguimientos:vencer` | Cada hora | Marcar seguimientos con fecha pasada como "Vencido" | `VencerSeguimientosUseCase` |
| `cotizaciones:vencer` | Diario 00:00 | Marcar cotizaciones vencidas como "Perdido" automáticamente | `VencerCotizacionesUseCase` |
| `reservas:finalizar` | Diario 06:00 | Marcar reservas con fecha de viaje pasada como "Finalizada" | `FinalizarReservaUseCase` |
| `backup:diario` | Diario 03:00 | Backup de BD + archivos con retención 30 días | Script `backup-adventur.sh` |
| `auditoria:limpiar` | Mensual | Archivar particiones de auditoría antiguas (opcional) | — |

---

## 12. Pruebas por Capa

| Capa | Tipo de Prueba | Framework | Ejemplo de Archivo |
|------|---------------|-----------|-------------------|
| Dominio | Unitarias (Entidades, ValueObjects, Reglas) | Pest + PHPUnit | `tests/Unit/Dominio/PrecioTest.php`, `tests/Unit/Dominio/EstadoCotizacionTest.php` |
| Aplicación | Integración (Casos de Uso con repositorios mock) | Pest + Mockery | `tests/Unit/Aplicacion/CrearCotizacionUseCaseTest.php` |
| Infraestructura | Integración (Repositorios con BD real) | Pest + RefreshDatabase | `tests/Feature/Repositorios/RepositorioCotizacionEloquentTest.php` |
| HTTP | Feature (Controladores con autenticación) | Pest + Sanctum | `tests/Feature/Cotizacion/CrearCotizacionTest.php` |
| E2E | Extremo a extremo (Flujo completo) | Playwright / Laravel Dusk | `tests/Browser/FlujoReservaCompletaTest.php` |
| Frontend | Unitarias (Componentes Vue) | Vitest + Vue Test Utils | `frontend/src/__tests__/BadgeEstado.spec.ts` |
| Frontend | E2E (Playwright) | Playwright | `frontend/e2e/flujo-cotizacion.spec.ts` |

---

## 13. Resumen de Reglas de Bloqueo (Descuentos y Vencimientos)

| ID | Regla | Condición | Mensaje de Error | Archivo de Validación |
|----|-------|-----------|-------------------|-----------------------|
| DB-01 | Descuento excede límite del Vendedor | `%desc > 10%` AND rol=VENDEDOR AND sin aprobación | "Requiere aprobación de Supervisor" | `app/Core/Dominio/Reglas/ReglaDescuentoRol.php` |
| DB-02 | Descuento excede límite del Supervisor | `%desc > 15%` AND rol=SUPERVISOR AND sin aprobación | "Requiere aprobación de Administrador" | `app/Core/Dominio/Reglas/ReglaDescuentoRol.php` |
| DB-03 | Descuento excede límite absoluto | `%desc > 50%` | "El descuento no puede exceder el 50%" | `app/Core/Dominio/Reglas/ReglaDescuentoRol.php` |
| DB-04 | Cotización vencida → Aprobar | `valid_until < CURRENT_DATE` AND intento a "Aprobado" | "Cotización vencida. Debe renovarse." | `app/Core/Aplicacion/CasosUso/Cotizaciones/AprobarCotizacionUseCase.php` |
| DB-05 | Cotización vencida → Convertir | `valid_until < CURRENT_DATE` AND intento de conversión | "Cotización vencida. Debe renovarse." | `app/Core/Aplicacion/CasosUso/Cotizaciones/ConvertirCotizacionAReservaUseCase.php` |
| DB-06 | Cotización vencida → Enviar | `valid_until < CURRENT_DATE` AND intento de envío | "La cotización vencerá pronto. Renueve para extender validez." | `app/Core/Aplicacion/CasosUso/Cotizaciones/EnviarCotizacionUseCase.php` |
| DB-07 | Pago excede saldo pendiente | `monto_pago > saldo_pendiente` | "El pago excede el saldo pendiente (S/ X.XX)" | `app/Core/Aplicacion/CasosUso/Reservas/RegistrarPagoUseCase.php` |
| DB-08 | Seguimiento con fecha pasada | `scheduled_at <= CURRENT_TIMESTAMP` | "La fecha de seguimiento debe ser futura" | `app/Core/Dominio/Reglas/ReglaFechaFutura.php` |
| DB-09 | Eliminación física de registro protegido | Intento de DELETE físico | "No está permitido eliminar físicamente este registro." | `app/Infraestructura/Persistencia/Repositorios/` (todos implementan soft delete) |

---

## 14. Matriz de Dependencias entre Paquetes y Reglas

| Paquete Composer | Uso en el Sistema | Regla/Requisito que Implementa |
|-----------------|-------------------|-------------------------------|
| `laravel/sanctum` | Autenticación API via tokens Bearer | RF-01, NF-01 |
| `spatie/laravel-permission` | RBAC (roles y permisos) | RF-20, NF-08 |
| `barryvdh/laravel-dompdf` | Generación de PDF de 2 páginas | RF-13, RF-14 |
| `maatwebsite/laravel-excel` | Exportación de reportes a Excel | RF-16 |
| `nunomaduro/larastan` | Análisis estático PHPStan | CA-10 (calidad de código) |
| `laravel/horizon` | Monitoreo de colas Redis (Fase 2) | NF-02, NF-06 |
| `predis/predis` | Cliente Redis para colas (Fase 2) | NF-02, NF-06 |

---

## 15. Trazabilidad de Reglas de Visibilidad (RBAC)

| Regla de Visibilidad | Descripción | Implementación Backend | Implementación Frontend |
|---------------------|-------------|----------------------|------------------------|
| V-R01 | Vendedor solo ve registros propios (`asignado_a = su_id`) | Scope global `PropiosScope` en repositorios, filtro `WHERE asignado_a = ?` | `authStore.usuario.id` en queries TanStack Query, filtro automático |
| V-R02 | Supervisor ve registros de su equipo (`id_supervisor = su_id`) | Scope `EquipoScope` con JOIN a usuarios por `id_supervisor` | Filtro en API params `id_vendedor` |
| V-R03 | Operaciones solo ve reservas Confirmadas o Finalizadas | Scope `SoloReservasOperativasScope`: `WHERE estado IN ('Confirmada','Finalizada')` | Sidebar oculta enlaces a cotizaciones/leads para rol Operaciones |
| V-R04 | Contabilidad ve montos pero no datos personales | API Resource `ContabilidadResource` oculta campos personales | Componentes condicionales con `v-if="esContabilidad"` |
| V-R05 | Gerencia solo lectura en todos los módulos | Middleware `SoloLecturaMiddleware` retorna 403 en POST/PUT/DELETE | Botones de acción ocultos con `v-if="!esGerencia"` |
| V-R06 | Admin acceso total excepto DELETE físico | Middleware `AdminMiddleware`, repositorios siempre ejecutan soft delete | Componentes sin restricción para admin |
