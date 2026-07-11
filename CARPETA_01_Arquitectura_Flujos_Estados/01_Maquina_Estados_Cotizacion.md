# Máquina de Estados de Cotización

**Carpeta:** 01 — Arquitectura de Flujos y Estados  
**Versión:** 1.0 — Especificación Agnóstica (Cero Tecnología)

---

## 1. Diagrama de Estados

```
                      ┌─────────────┐
                      │   NUEVO     │
                      │ (Estado 0)  │
                      └──────┬──────┘
                             │
                    Condición: Vendedor completa
                    formulario y persiste borrador
                             │
                             ▼
                      ┌─────────────┐
              ┌───────│  COTIZADO   │◄──────┐
              │       │ (Estado 1)  │       │
              │       └──────┬──────┘       │
              │              │              │
              │    Condición: Enviar por     │
              │    WhatsApp/Email/PDF        │
              │              │              │
              │              ▼              │
              │       ┌─────────────┐       │
              │       │  ENVIADO    │       │
              │       │ (Estado 2)  │       │
              │       └──────┬──────┘       │
              │              │              │
              │    Condición: Programar      │
              │    fecha futura de contacto  │
              │              │              │
              │              ▼              │
              │       ┌─────────────┐       │
              │       │EN SEGUIMIENTO│───────┘
              │       │ (Estado 3)  │ (reprogramar)
              │       └──────┬──────┘
              │              │
              │     ┌────────┴────────┐
              │     │                │
              │     ▼                ▼
              │  ┌──────────┐  ┌──────────┐
              │  │ APROBADO │  │ PERDIDO  │
              │  │(Estado 4)│  │(Estado 6)│
              │  └─────┬────┘  └─────┬────┘
              │        │             │
              │        ▼             │
              │  ┌──────────┐       │
              │  │ RESERVA  │       │
              │  │(Estado 5)│       │
              │  └──────────┘       │
              │                     │
              └─────────────────────┘ (reactivación)
```

---

## 2. Tabla de Transiciones de Estado

| ID | Desde | Hacia | Disparador | Acción de Validación | Rol Permitido |
|-----|-------|-------|-----------|----------------------|---------------|
| T01 | NUEVO | COTIZADO | Vendedor completa formulario y presiona "Guardar Cotización" | Todos los campos obligatorios completos. Cálculo matemático ejecutado. Template existe y está activo. | Vendedor, Admin, Supervisor |
| T02 | NUEVO | PERDIDO | Vendedor marca como "Perdido sin cotizar" | `motivo_perdida != NULL` AND `longitud(observacion) >= 10` | Vendedor, Admin, Supervisor |
| T03 | COTIZADO | ENVIADO | Sistema ejecuta envío (WhatsApp/Email/Descarga) | PDF generado exitosamente. Destinatario no vacío. Se registra `fecha_hora_envio`. | Vendedor, Admin, Supervisor |
| T04 | ENVIADO | EN SEGUIMIENTO | Vendedor programa fecha de contacto | `scheduled_at > CURRENT_TIMESTAMP` (fecha futura obligatoria) | Vendedor, Admin, Supervisor |
| T05 | EN SEGUIMIENTO | EN SEGUIMIENTO | Vendedor reprograma seguimiento | `nueva_scheduled_at > CURRENT_TIMESTAMP` AND `nueva != anterior`. El seguimiento previo pasa a estado "REPROGRAMADO". | Vendedor, Admin, Supervisor |
| T06 | EN SEGUIMIENTO | APROBADO | Cliente acepta cotización formalmente | `quote.valid_until >= CURRENT_DATE` (no vencida). Se registra `fecha_aprobacion`. | Vendedor, Admin, Supervisor |
| T07 | EN SEGUIMIENTO | PERDIDO | Vendedor confirma pérdida | `motivo_perdida != NULL` AND `longitud(observacion) >= 10` | Vendedor, Admin, Supervisor |
| T08 | APROBADO | RESERVA | Vendedor ejecuta "Convertir en Reserva" | Existe cliente vinculado. Se ejecuta flujo de 5 pasos completo. | Vendedor, Admin, Supervisor |
| T09 | PERDIDO | COTIZADO | Admin/Supervisor reactiva cotización | Solo rol Admin o Supervisor. Registra razón de reactivación. Genera nuevo `valid_until`. | Admin, Supervisor |

---

## 3. Definición Formal de Estados

| Estado | Código Interno | Descripción | ¿Terminal? |
|--------|---------------|-------------|------------|
| NUEVO | `nuevo` | Lead registrado, aún no se ha generado cotización formal | No |
| COTIZADO | `cotizado` | Cotización calculada y persistida, aún no enviada al cliente | No |
| ENVIADO | `enviado` | Cotización enviada al cliente por algún medio | No |
| EN SEGUIMIENTO | `en_seguimiento` | Seguimiento programado, esperando respuesta del cliente | No |
| APROBADO | `aprobado` | Cliente aceptó la cotización verbalmente o por escrito | No |
| RESERVA | `reserva` | Cotización convertida en reserva formal con pago | Sí (terminal positivo) |
| PERDIDO | `perdido` | Cliente no concretó, con motivo registrado | Sí (terminal negativo, reactivable solo por Admin/Supervisor) |

---

## 4. Estados Absorbentes (Reglas Inmutables)

### R-E01: Estado RESERVA
Una vez que una Cotización alcanza el estado `RESERVA`, NO puede:
- Cambiar a ningún otro estado.
- Ser modificada en montos, márgenes o descuentos.
- Ser reutilizada para generar otra reserva.

La entidad `reserva` vinculada es la que gestiona cambios operativos posteriores.

### R-E02: Estado PERDIDO
Una vez que una Cotización alcanza el estado `PERDIDO`:
- NO puede cambiar a ningún estado EXCEPTO mediante reactivación explícita.
- La reactivación solo la puede ejecutar un usuario con rol `Admin` o `Supervisor`.
- La reactivación requiere registrar obligatoriamente: `razon_reactivacion`, y genera un nuevo `valid_until`.
- La reactivación genera un registro de auditoría (evento A10).

---

## 5. Visibilidad de Estados por Rol

| Rol | NUEVO | COTIZADO | ENVIADO | EN SEGUIMIENTO | APROBADO | RESERVA | PERDIDO |
|-----|-------|----------|---------|----------------|----------|---------|---------|
| **Vendedor** | Propias | Propias | Propias | Propias | Propias | Propias | Propias |
| **Supervisor** | Equipo | Equipo | Equipo | Equipo | Equipo | Equipo | Equipo |
| **Admin** | Todas | Todas | Todas | Todas | Todas | Todas | Todas |
| **Operaciones** | ❌ | ❌ | ❌ | ❌ | ❌ | Confirmadas/Finalizadas | ❌ |
| **Contabilidad** | ❌ | ❌ | ❌ | ❌ | ❌ | Solo montos | ❌ |
| **Gerencia** | Todas (RO) | Todas (RO) | Todas (RO) | Todas (RO) | Todas (RO) | Todas (RO) | Todas (RO) |

**RO = Solo Lectura (Read-Only). Ninguna acción de escritura permitida.**

---

## 6. Pseudocódigo del Motor de Transiciones

```
FUNCIÓN TransicionarEstado(cotizacion_id, estado_destino, datos_transicion):
    cotizacion = OBTENER(cotizaciones WHERE id = cotizacion_id)
    estado_origen = cotizacion.estado
    
    // Validar que la transición sea válida
    SI NO_EXISTE_TRANSICION(estado_origen, estado_destino):
        ERROR("Transición de {estado_origen} a {estado_destino} no está permitida")
    FIN SI
    
    // Validar permisos del rol
    SI NO_TIENE_PERMISO(usuario_actual.rol, estado_origen, estado_destino):
        ERROR("El rol {usuario_actual.rol} no tiene permiso para esta transición")
    FIN SI
    
    // Validar condiciones específicas de la transición
    SEGÚN (estado_origen + "→" + estado_destino):
        CASO "ENVIADO→EN_SEGUIMIENTO":
            SI datos_transicion.scheduled_at <= CURRENT_TIMESTAMP:
                ERROR("La fecha de seguimiento debe ser futura")
            FIN SI
        
        CASO "EN_SEGUIMIENTO→PERDIDO":
            SI datos_transicion.motivo_perdida IS NULL:
                ERROR("Debe seleccionar un motivo de pérdida")
            FIN SI
            SI LONGITUD(datos_transicion.observacion) < 10:
                ERROR("La observación debe tener al menos 10 caracteres")
            FIN SI
        
        CASO "APROBADO→RESERVA":
            SI cotizacion.valid_until < CURRENT_DATE:
                ERROR("La cotización está vencida. Debe renovarse antes de convertir.")
            FIN SI
            // La validación del flujo de 5 pasos se maneja en el proceso de reserva
    FIN SEGÚN
    
    // Ejecutar transición
    cotizacion.estado = estado_destino
    cotizacion.fecha_actualizacion = CURRENT_TIMESTAMP
    
    // Registrar auditoría
    REGISTRAR_AUDITORIA(
        accion = "Modificación de estado",
        tipo_entidad = "Cotización",
        id_entidad = cotizacion.id,
        valores_anteriores = "estado=" + estado_origen,
        valores_nuevos = "estado=" + estado_destino
    )
    
    RETORNAR cotizacion
FIN FUNCIÓN
```
