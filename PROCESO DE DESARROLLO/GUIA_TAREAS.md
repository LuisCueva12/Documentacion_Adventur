# GUÍA DE DESARROLLO — Adventur Travel

> **Propósito:** seguimiento detallado y definido por módulo de lo realizado y lo
> pendiente, para tener claro el estado del sistema y qué falta refinar.
>
> Orden = flujo del usuario (desde el login). Cada tarea tiene estado:
> - ✅ **Completado** (verificado)
> - 🟡 **Base implementada** (existe endpoint/vista; pendiente pulir o probar a fondo)
> - ⬜ **Pendiente**
> - 🔴 **Bloqueante (P0)**
>
> Referencias: `RF-xx` = Requisitos Funcionales · `Doc 0x` = carpeta Docs ·
> `E06` = Especificación de Plantillas.

---

## 1. Autenticación, Login y Perfil  ·  RF-01, RF-20
**Endpoints:** `POST /auth/login`, `/auth/logout`, `/auth/me`, `PUT /auth/perfil`
**Vistas:** `Auth/Signin.vue`, `Perfil.vue`

- [x] Login email + contraseña (Sanctum, bcrypt, throttle `login`)
- [x] Logout y sesión persistente en store `auth`
- [x] `AuthControlador@me` y `actualizarPerfil`
- [x] Matriz RBAC sembrada: 6 roles × 15 entidades (`RolSeeder`)
- [x] Guards de ruta (`beforeEach`) + directivas por permiso en frontend
- [x] Vista de perfil de usuario (`/perfil`)
- [ ] 🔴 **P1** Visibilidad RBAC a nivel de datos en repositorios (hoy solo
      config JSON + filtrado UI; no se enforce `propios`/`equipo`/`todos` en BD)

## 2. Dashboard e Indicadores  ·  RF-15, RF-16
**Endpoint:** `GET /dashboard/resumen` (permiso `reportes,ver`)
**Vistas:** `Dashboard.vue`

- [x] Contadores operativos: clientes, cotizaciones, reservas, leads, ingresos, próximas salidas
- [x] KPIs gerenciales cableados `ServicioKpi` → `DashboardResource` → frontend:
      **TCV, TPC, TPV, TPP, SV** (verificado con datos reales)
- [x] Grid "Indicadores gerenciales · mes actual" en `Dashboard.vue`
- [ ] ⬜ **TRP** (Tiempo de Respuesta): requiere esquema `fecha_envio` + `id_lead`
      en cotizaciones y `fecha_consulta` en leads → hoy muestra `N/D`
- [ ] ⬜ Selector de período en el dashboard (hoy usa mes actual fijo)

## 3. Clientes  ·  RF-03
**Endpoints:** `GET /clientes/buscar`, `GET/POST/PUT /clientes/{id}`
**Vistas:** `ClientesLista`, `ClienteForm`, `ClienteDetalle`

- [x] Lista, búsqueda, alta y edición
- [x] Búsqueda por documento (índices GIN, debounce)
- [ ] 🟡 Pendiente de prueba funcional y pulido de validaciones/estados

## 4. Leads  ·  Arquitectura Hexagonal ✅
**Endpoints:** `GET/POST/PUT /leads/{id}`, `POST /leads/{id}/convertir`, `POST /leads/{id}/seguimiento`, `POST /leads/{id}/perder`
**Vistas:** `LeadsLista`, `LeadForm`, `LeadDetalle`

**Backend (hexagonal):**
- [x] Controller thin (82 líneas, 0 lógica de negocio)
- [x] 6 Use Cases en `CasosUso/Leads/`: Crear, Actualizar, Listar, Convertir, Seguimiento, Perder
- [x] 6 DTOs en `DTOs/Leads/`: CrearLeadDto, ActualizarLeadDto, ConvertirLeadDto, RegistrarSeguimientoLeadDto, MarcarPerdidoLeadDto, PaginaLeadsDto
- [x] 6 Peticiones con `aDto()`: CrearLeadPeticion, ActualizarLeadPeticion, ListarLeadsPeticion, ConvertirLeadPeticion, RegistrarSeguimientoLeadPeticion, MarcarPerdidoLeadPeticion
- [x] `InterfazRepositorioLead` → `RepositorioLeadEloquent` (crear, guardar, obtenerPorId, listarPaginado, eliminar)
- [x] FSM corregido: Nuevo→EnSeguimiento→Convertido|Perdido
- [x] Búsqueda `ILIKE` (case-insensitive) + debounce 300ms frontend
- [x] Migración unificada (sin parches)

**Frontend:**
- [x] Lista con filtros: búsqueda (debounce), estado, canal de origen
- [x] LeadDetalle: seguimiento, motivo pérdida, botones acción, conversión con nombre formal
- [x] Tipos TypeScript actualizados

- [ ] ⬜ Ligar lead → cotización (`id_lead`) y registrar `fecha_consulta`
      (requerido para TRP)
- [ ] ⬜ Filtro por vendedor asignado (requiere catálogo `/usuarios?vendedor=1`)
- [ ] ⬜ Historial de seguimientos (tabla `seguimientos_lead`)
- [ ] 🟡 Pendiente de pulido y verificación del flujo lead→cotización

## 5. Plantillas (Perfiles Comerciales)  ·  E06  ✅ COMPLETADO + COMMIT
**Endpoints:** `GET/POST/PUT/DELETE /catalogos/plantillas`
**Vistas:** `PlantillasLista`, `PlantillaForm`

- [x] 7 campos por defecto en migración oficial
      (`default_movilidad/guia/entradas/alimentacion/extras/margen`, `dias_validez`)
- [x] Dominio + VOs `Precio`/`Porcentaje`, repo, `ServicioPlantillas`, `PlantillaResource`, rutas CRUD
- [x] Frontend: lista, formulario (precios/validez), precarga en Cotizador
- [x] Commits: backend `98e1d93` · frontend `a1acb72`

## 6. Cotizaciones (Cotizador + FSM)  ·  RF-02, RF-08
**Endpoints:** `GET/POST/PUT/DELETE /cotizaciones/{id}`,
`/enviar`, `/seguimiento`, `/aprobar`, `/perder`, `/rechazar`, `/convertir`,
`/reactivar`, `GET /cotizaciones/{id}/pdf`
**Vistas:** `CotizacionesLista`, `Cotizador`, `CotizacionDetalle`

- [x] Cotizador (wizard) con servicios desde plantilla
- [x] Códigos correlativos únicos por año (`GenerarSecuenciaCodigoUseCase`, RF-02)
- [x] FSM 7 estados en dominio (Nuevo→Cotizado→Enviado→En Seguimiento→Aprobado→Reserva→Perdido)
- [x] Acciones: enviar, programar seguimiento, aprobar, perder, rechazar, reactivar
- [x] PDF de cotización (`DocumentoControlador@cotizacion`)
- [ ] 🔴 **P0** `convertir` genera código `RES-` (debe ser `ADV-`) y crea una
      reserva vacía → rompe regla R1 (cotización ≠ reserva). Corregir en `CotizacionControlador@convertir`
- [ ] 🔴 **P0** Casing mezclado de estados (cotizaciones minúsculas vs reservas
      TitleCase) causa bugs silenciosos → unificar criterio
- [ ] ⬜ Acción "enviar" debe registrar `fecha_envio` (necesario para TCV/TRP)
- [ ] 🟡 Pendiente de prueba funcional de todo el flujo FSM y del PDF

## 7. Reservas  ·  RF-04 … RF-14, RF-18
**Endpoints:** `GET/POST/PUT/DELETE /reservas/{id}`, `/anular`,
`POST /reservas/{id}/pagos`, `POST /pagos/{id}/anular`, `GET /reservas/{id}/pdf`
**Vistas:** `ReservasLista`, `ReservaNueva`, `ReservaDetalle`

- [x] Alta en Borrador, pasajeros (mín 1), edad automática, contacto emergencia (RF-04/05/06/07)
- [x] Anulación lógica (RF-18)
- [x] Pagos, saldo en tiempo real, PDF confirmación 2 páginas (DomPdf) (RF-11/13/14)
- [ ] 🔴 **P0** Corregir conversión desde cotización (ver módulo 6 P0)
- [ ] 🔴 Unificar casing de estados (reservas ya TitleCase; alinear con cotizaciones)
- [ ] 🟡 Pendiente de prueba funcional de alta, pagos y PDF

## 8. Pagos  ·  RF-11, RF-12
**Endpoints:** `POST /reservas/{id}/pagos`, `POST /pagos/{id}/anular`
**Vistas:** dentro de `ReservaDetalle`

- [x] Registro de pagos parciales, saldo en tiempo real
- [x] Anulación de pago
- [ ] 🟡 Pendiente de pulido (comprobantes ≤5MB PDF/JPG/PNG, validaciones monto)

## 9. Catálogos  ·  (transversal a RF-08/09/10)
**Endpoints:** `/catalogos/motivos-perdida`, `/canales-origen`, `/plantillas`,
`/paquetes`, `/hoteles`, `/movilidades`, `/precio-sugerido`, `/sincronizar/{tipo}`
**Vistas:** `Catalogos.vue`

- [x] Motivos de pérdida (catálogo para TPP)
- [x] Canales de origen
- [x] Paquetes/Rutas (CRUD + sincronizar)
- [x] Hoteles/Alojamientos (CRUD)
- [x] Movilidades/Transporte (CRUD)
- [x] Precio sugerido
- [ ] 🟡 Pendiente de pulido de UX de gestión y de la sincronización externa

## 10. Reportes e Indicadores  ·  RF-16
**Endpoints:** `/reportes/ventas`, `/conversion`, `/rendimiento`, `/export`
**Vistas:** `ReportesLista`

- [x] `ServicioKpi`: `ventas()`, `conversion()`, `rendimiento()`
- [x] `ReporteControlador`: ventas, conversión, rendimiento
- [x] Endpoint de exportación (`/reportes/{tipo}/export`)
- [ ] ⬜ Exportación Excel/PDF real (`maatwebsite/laravel-excel`, DomPdf)
- [ ] ⬜ Gráficos frontend (ApexCharts ya en dependencias)

## 11. Auditoría  ·  RF-17
**Endpoints:** `GET /auditoria`
**Vistas:** `AuditoriaLista`

- [x] Tabla + listeners de eventos de dominio
- [ ] 🟡 Pendiente de verificar qué acciones se registran y pulido de la vista

## 12. Administración (Usuarios y Roles)  ·  RF-20
**Endpoints:** `/admin/usuarios`, `/admin/usuarios/{id}/rol`, `/admin/roles`
**Vistas:** `UsuariosLista`

- [x] Lista de usuarios, alta, edición, cambio de rol
- [x] Lista de roles (desde `RolSeeder`)
- [ ] 🟡 Pendiente de pulido y de verificar permisos por acción en la UI

## 13. Políticas del documento  ·  RF-19
**Endpoints:** `GET/PUT /admin/configuracion/politicas`
**Vistas:** `PoliticasDocumento`

- [x] Listar y actualizar políticas del PDF (edición versionada)
- [ ] 🟡 Pendiente de verificar trazabilidad/auditoría de cambios

## 14. Consulta de documento (SUNAT/Reniec)  ·  RF-03
**Endpoints:** `GET /consulta/documento` (permiso `clientes,ver`, throttle)
**Vistas:** usado en `ClienteForm`/búsqueda

- [x] Endpoint de consulta externa
- [ ] 🟡 Pendiente de verificar integración real con la fuente externa

## 15. RBAC / Permisos (transversal)  ·  RF-20, Doc 04
- [x] Matriz 6 roles × 15 entidades sembrada (`RolSeeder`)
- [x] Middleware `permiso:modulo,accion` en todas las rutas API
- [x] Guards de ruta y `v-if` en frontend
- [ ] 🔴 **P1** Enforcement de visibilidad a nivel de datos en repositorios
      (`propios`/`equipo`/`todos`, `solo_reserva`, `montos`, etc.)

---

## DEUDA TÉCNICA PRIORITARIA
1. 🔴 **P0** `convertir`: código `RES-` → `ADV-` y no crear reserva vacía (R1)
2. 🔴 **P0** Unificar casing de estados cotización/reserva
3. 🔴 **P1** Visibilidad RBAC a nivel de datos (repositorios)
4. ⬜ **P1** TRP: agregar `fecha_envio` + `id_lead` (cotizaciones) y `fecha_consulta` (leads)

## TRAZABILIDAD REQUISITO → CÓDIGO
- KPIs: PDF fuente (sec. 9.1) → `Doc 03_KPI_Gerenciales` → RF-16 → `ServicioKpi`
- Dashboard: RF-15 → `DashboardControlador@resumen`
- RBAC: `Doc 04_Matriz_RBAC` → `RolSeeder` → `VerificarPermisoUseCase`
- Plantillas: `E06` → `ServicioPlantillas` / `PlantillaResource`
- Reservas/Pagos/PDF: RF-04…14, RF-18 → `ReservaControlador` / `PagoControlador` / `DocumentoControlador`
