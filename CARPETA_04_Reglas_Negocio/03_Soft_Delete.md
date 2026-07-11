# Regla de Eliminación Lógica (Soft Delete)

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Regla Inmutable (R4): Prohibición de Eliminación Física

> Ningún registro financiero, de reserva, de cotización, de pago, de cliente o de usuario
> puede ser ELIMINADO FÍSICAMENTE del sistema.
>
> Solo se permite la **ELIMINACIÓN LÓGICA** (Soft Delete), que consiste en marcar el
> registro como "eliminado" mediante un campo indicador, sin borrarlo del almacenamiento.

---

## 2. Entidades Protegidas por Soft Delete

| Entidad | Campo Indicador | Valor "Activo" | Valor "Eliminado" |
|---------|----------------|:--------------:|:-----------------:|
| `usuarios` | `activo` | `TRUE` | `FALSE` |
| `clientes` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `leads` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `cotizaciones` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `cotizacion_detalle` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `plantillas` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `rutas` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `reservas` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `pasajeros` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `pagos` | `fecha_eliminacion` | `NULL` | `Timestamp` |
| `politicas` | `fecha_eliminacion` | `NULL` | `Timestamp` |

---

## 3. Comportamiento del Sistema

### 3.1 Regla de Filtro Universal

TODAS las consultas del sistema deben filtrar automáticamente los registros eliminados:

```
CONSULTA_NORMAL = SELECT * FROM entidad WHERE fecha_eliminacion IS NULL
```

El usuario ve SOLO los registros activos. Los eliminados están ocultos por defecto.

### 3.2 Excepción: Consulta de Eliminados

El rol **Administrador** puede consultar registros eliminados mediante una opción explícita "Ver eliminados" o "Papelera".

### 3.3 Efecto en Cascada (Soft Delete Lógico)

| Acción | Efecto |
|--------|--------|
| Se elimina lógicamente una `cotización` | Sus `seguimientos` y `cotizacion_detalle` también se ocultan (misma fecha_eliminacion) |
| Se elimina lógicamente una `reserva` | Sus `pasajeros` y `pagos` también se ocultan |
| Se elimina lógicamente un `cliente` | No puede asociarse a nuevas cotizaciones ni reservas. Las existentes permanecen visibles |
| Se elimina lógicamente un `usuario` | No puede iniciar sesión. Sus registros creados permanecen visibles (con su nombre) |

### 3.4 Reactivación (Undo Delete)

Cualquier registro eliminado lógicamente puede ser **restaurado** por un Administrador:

```
FUNCIÓN RestaurarEntidad(tipo_entidad, id_entidad):
    entidad = OBTENER_REGISTRO_INCLUSO_ELIMINADO(tipo_entidad, id_entidad)
    
    SI entidad.fecha_eliminacion IS NULL:
        ERROR("El registro no está eliminado")
    FIN SI
    
    entidad.fecha_eliminacion = NULL
    
    REGISTRAR_AUDITORIA(
        accion = "Reactivación",
        tipo_entidad = tipo_entidad,
        id_entidad = id_entidad,
        valores_anteriores = "fecha_eliminacion=" + entidad.fecha_eliminacion,
        valores_nuevos = "fecha_eliminacion=NULL"
    )
    
    RETORNAR OPERACIÓN_EXITOSA
FIN FUNCIÓN
```

---

## 4. Justificación de Negocio

### 4.1 Cumplimiento Fiscal (Legislación Peruana)

Los registros financieros y contables deben conservarse por un mínimo de **5 años** según la legislación peruana (Código Tributario). La eliminación física de un pago, reserva o cotización podría constituir una infracción fiscal.

### 4.2 Trazabilidad Comercial

Un cliente puede reclamar sobre un servicio recibido hace 2 años. Si el registro fue eliminado físicamente, no hay forma de verificar lo ocurrido. Con soft delete, el Administrador puede restaurar y consultar el histórico.

### 4.3 Reversibilidad de Errores

Un vendedor puede eliminar accidentalmente una cotización importante. Con soft delete, la acción es reversible sin necesidad de restaurar backups completos.

### 4.4 Protección contra Fraude

La eliminación física de registros financieros podría ser utilizada para ocultar transacciones. El soft delete, combinado con auditoría, impide esta práctica.

---

## 5. Pseudocódigo del Soft Delete

```
FUNCIÓN EliminarLogicamente(tipo_entidad, id_entidad, id_usuario):
    entidad = OBTENER(tipo_entidad WHERE id = id_entidad)
    
    SI entidad.fecha_eliminacion != NULL:
        ERROR("El registro ya está eliminado")
    FIN SI
    
    // Marcar como eliminado
    entidad.fecha_eliminacion = CURRENT_TIMESTAMP
    
    // Registrar en auditoría
    REGISTRAR_AUDITORIA(
        accion = "Anulación",
        tipo_entidad = tipo_entidad,
        id_entidad = id_entidad,
        id_usuario = id_usuario,
        valores_anteriores = "fecha_eliminacion=NULL",
        valores_nuevos = "fecha_eliminacion=" + CURRENT_TIMESTAMP
    )
    
    // Soft delete en cascada si aplica
    SEGÚN tipo_entidad:
        CASO "Cotización":
            PARA_CADA seguimiento EN seguimientos WHERE id_cotizacion = id_entidad:
                seguimiento.fecha_eliminacion = CURRENT_TIMESTAMP
            FIN PARA_CADA
            PARA_CADA detalle EN cotizacion_detalle WHERE id_cotizacion = id_entidad:
                detalle.fecha_eliminacion = CURRENT_TIMESTAMP
            FIN PARA_CADA
        
        CASO "Reserva":
            PARA_CADA pasajero EN pasajeros WHERE id_reserva = id_entidad:
                pasajero.fecha_eliminacion = CURRENT_TIMESTAMP
            FIN PARA_CADA
            PARA_CADA pago EN pagos WHERE id_reserva = id_entidad:
                pago.fecha_eliminacion = CURRENT_TIMESTAMP
            FIN PARA_CADA
    FIN SEGÚN
    
    RETORNAR OPERACIÓN_EXITOSA
FIN FUNCIÓN
```
