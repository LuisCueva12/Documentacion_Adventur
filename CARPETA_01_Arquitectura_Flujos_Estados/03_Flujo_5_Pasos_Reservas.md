# Flujo Operativo Guiado de 5 Pasos para Creación de Reservas

**Carpeta:** 01 — Arquitectura de Flujos y Estados  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Diagrama del Flujo

```
PASO 1 ──── VALIDACIÓN ────► PASO 2 ──── VALIDACIÓN ────► PASO 3 ──── VALIDACIÓN ────► PASO 4 ──── VALIDACIÓN ────► PASO 5
Titular        │           Viaje          │          Pasajeros       │         Pago          │        Confirmación
               │                         │                         │                       │
               ▼                         ▼                         ▼                       ▼
         Bloqueante:              Bloqueante:               Bloqueante:             Bloqueante:
         client.id != NULL        travel_date >              1 pasajero              adelanto >=
                                  CURRENT_DATE              marcado como             total * 30%
                                                            titular
```

Cada paso es una etapa lógica independiente. NO se puede avanzar al siguiente paso sin validar el actual. NO se puede retroceder sin reiniciar campos específicos.

---

## 2. PASO 1: TITULAR

**Propósito:** Identificar al responsable principal de la reserva.

### Campos del Formulario

| Campo | Tipo Lógico | Obligatorio | Validación |
|-------|-------------|-------------|------------|
| Cliente | Entidad (ID) | SÍ | Debe existir en entidad `clientes`. Si no existe, debe crearse antes de avanzar. La búsqueda se realiza por: `numero_documento` (prioridad 1), `telefono` (prioridad 2), `correo_electronico` (prioridad 3) |
| ¿El titular también viaja? | Booleano | SÍ | Si es `TRUE`, se auto-crea un registro en `pasajeros` con `es_titular = TRUE` al llegar al Paso 3 |

### Algoritmo de Búsqueda de Cliente

```
FUNCIÓN BuscarCliente(criterio):
    // Prioridad 1: Búsqueda por número de documento
    SI criterio.tipo_documento != NULL Y criterio.numero_documento != NULL:
        cliente = BUSCAR(clientes WHERE tipo_documento = criterio.tipo_documento
                          AND numero_documento = criterio.numero_documento
                          AND fecha_eliminacion IS NULL)
        SI cliente ENCONTRADO:
            RETORNAR cliente
        FIN SI
    FIN SI
    
    // Prioridad 2: Búsqueda por teléfono
    SI criterio.telefono != NULL:
        cliente = BUSCAR(clientes WHERE telefono = criterio.telefono
                          AND fecha_eliminacion IS NULL)
        SI cliente ENCONTRADO:
            RETORNAR cliente
        FIN SI
    FIN SI
    
    // Prioridad 3: Búsqueda por correo electrónico
    SI criterio.correo != NULL:
        cliente = BUSCAR(clientes WHERE correo_electronico = criterio.correo
                          AND fecha_eliminacion IS NULL)
        SI cliente ENCONTRADO:
            RETORNAR cliente
        FIN SI
    FIN SI
    
    // No encontrado → debe crear nuevo cliente
    RETORNAR NULL (requiere creación)
FIN FUNCIÓN
```

### Validación Bloqueante del Paso 1

```
P1_VALIDO ↔ (client.id IS NOT NULL)
             AND (client.nombre_completo != "")
             AND (client.telefono IS NOT NULL)
```

---

## 3. PASO 2: VIAJE

**Propósito:** Definir los parámetros operativos del viaje.

### Campos del Formulario

| Campo | Tipo Lógico | Obligatorio | Validación |
|-------|-------------|-------------|------------|
| Fecha de viaje | Fecha | SÍ | `travel_date > CURRENT_DATE` (debe ser futura). Si se selecciona una fecha pasada: ERROR |
| Hora de salida | Hora (HH:MM) | NO | Formato 24 horas |
| Lugar de recojo | Texto (255) | NO | |
| Fecha de retorno | Fecha | NO | Si se especifica: `return_date >= travel_date`. Si es anterior: ERROR |
| Ruta/Paquete | Entidad (ID) | SÍ | Debe existir en entidad `rutas` y `ruta.activa = TRUE` |
| Movilidad asignada | Texto (255) | NO | |
| Guía asignado | Texto (255) | NO | |
| Conductor asignado | Texto (255) | NO | |
| **Alojamiento requerido** | Booleano | SÍ | Default: `FALSE`. Si `TRUE`, se habilitan campos de alojamiento. Aplica para rutas multi-día |
| Hotel / Alojamiento | Texto (255) | Condicional | SÍ si `alojamiento_requerido = TRUE` |
| Tipo de habitación | Lista (Enum) | Condicional | SÍ si `alojamiento_requerido = TRUE`. Valores: `Individual`, `Doble`, `Triple`, `Compartida`, `Suite` |
| Tipo de cama | Lista (Enum) | NO | Valores: `Queen`, `King`, `Dos camas individuales`, `Litera` |
| Plan de comidas | Lista (Enum) | NO | Valores: `Solo desayuno`, `Media pensión`, `Pensión completa`, `Todo incluido` |
| Fecha check-in | Fecha | Condicional | SÍ si `alojamiento_requerido = TRUE`. `check_in >= travel_date` |
| Fecha check-out | Fecha | Condicional | SÍ si `alojamiento_requerido = TRUE`. `check_out > check_in` |

### Validación Bloqueante del Paso 2

```
P2_VALIDO ↔ (travel_date > CURRENT_DATE)
             AND (route.id IS NOT NULL)
             AND (route.activa = TRUE)
             AND (return_date IS NULL OR return_date >= travel_date)
             AND (
                    (alojamiento_requerido = FALSE)
                    OR (alojamiento_requerido = TRUE
                        AND hotel IS NOT NULL
                        AND tipo_habitacion IS NOT NULL
                        AND check_in IS NOT NULL
                        AND check_out IS NOT NULL
                        AND check_in >= travel_date
                        AND check_out > check_in)
                 )
```

---

## 4. PASO 3: PASAJEROS

**Propósito:** Registrar todos los viajantes. Mínimo 1 pasajero (el titular si marcó que viaja en Paso 1).

### Campos del Formulario (por cada pasajero)

| Campo | Tipo Lógico | Obligatorio | Validación |
|-------|-------------|-------------|------------|
| Nombre completo | Texto (255) | SÍ | Debe contener al menos 2 palabras separadas por espacio (nombre + apellido). Si tiene menos: ERROR |
| Tipo de documento | Lista (Enum) | SÍ | Valores: `DNI`, `Pasaporte`, `RUC`, `Carné de Extranjería` |
| Número de documento | Texto (20) | SÍ (excepto menores sin DNI) | Según tipo: DNI = exactamente 8 dígitos. Pasaporte = 12 caracteres alfanuméricos. RUC = 11 dígitos |
| Fecha de nacimiento | Fecha | NO | Si se proporciona: `birth_date <= CURRENT_DATE`. Si es futura: ERROR |
| Edad | Entero | NO (calculado) | Solo lectura. Si `birth_date` presente: `edad = (CURRENT_DATE - birth_date).years` |
| Teléfono | Texto (20) | NO | |
| Correo electrónico | Texto (255) | NO | Si se proporciona, debe contener `@` y un dominio válido |
| Contacto de emergencia | Texto (255) | SÍ | |
| Teléfono de emergencia | Texto (20) | SÍ | |
| Notas de salud | Texto (libre) | NO | Alergias, condiciones médicas, restricciones alimenticias |
| ¿Requiere seguro? | Booleano | SÍ | Default: `FALSE` |
| ¿Es titular? | Booleano | SÍ | Exactamente 1 pasajero por reserva debe tener `TRUE`. El sistema lo asigna automáticamente si en Paso 1 se marcó "El titular viaja" |

### Reglas de Validación de Edad y Documento

| Rango de Edad | ¿Documento Obligatorio? | Observaciones |
|---------------|------------------------|---------------|
| `edad >= 18` | SÍ | DNI o Pasaporte obligatorio |
| `edad < 18` | SÍ, excepto menores de 7 días sin DNI | Puede registrar "PENDIENTE" como documento |
| `edad = NULL` (no proporcionada) | SÍ | Se asume adulto |

### Validación Bloqueante del Paso 3

```
P3_VALIDO ↔ (CONTAR(pasajeros) >= 1)
             AND (EXISTE_EXACTAMENTE_1(pasajeros WHERE es_titular = TRUE))
             AND (PARA_CADA pasajero: CONTAR_PALABRAS(pasajero.nombre_completo) >= 2)
             AND (PARA_CADA pasajero:
                    SI pasajero.tipo_documento = "DNI":
                        VALIDAR_FORMATO(pasajero.numero_documento, "^\d{8}$")
                    SI pasajero.tipo_documento = "Pasaporte":
                        VALIDAR_FORMATO(pasajero.numero_documento, "^[A-Za-z0-9]{12}$")
                    SI pasajero.tipo_documento = "RUC":
                        VALIDAR_FORMATO(pasajero.numero_documento, "^\d{11}$")
                 )
             AND (PARA_CADA pasajero: pasajero.contacto_emergencia != "")
             AND (PARA_CADA pasajero: pasajero.telefono_emergencia != "")
```

---

## 5. PASO 4: PAGO

**Propósito:** Registrar el adelanto o pago completo.

### Campos del Formulario

| Campo | Tipo Lógico | Obligatorio | Validación |
|-------|-------------|-------------|------------|
| Monto | Decimal(10,2) | SÍ | `monto > 0` AND `monto <= reserva.monto_total`. Si excede: ERROR |
| Método de pago | Lista (Enum) | SÍ | Valores: `Efectivo`, `Transferencia Bancaria`, `Yape`, `Plin`, `Tarjeta Débito/Crédito`, `Depósito` |
| Referencia / N° Operación | Texto (255) | NO | |
| Fecha y hora del pago | FechaHora | SÍ | `paid_at <= CURRENT_TIMESTAMP`. Si es futura: ERROR |
| ¿Es adelanto? | Booleano | SÍ (calculado) | `TRUE` si `monto < reserva.monto_total`. `FALSE` si `monto = reserva.monto_total` |
| Comprobante (archivo) | Binario | NO | Formatos aceptados: PDF, JPG, PNG. Tamaño máximo: 5 MB |

### Cálculos Automáticos Post-Registro

```
total_pagado = SUM(pagos.monto
                   WHERE pagos.id_reserva = reserva.id
                   AND pagos.fecha_anulacion IS NULL)

saldo = reserva.monto_total - total_pagado
```

### Validación Bloqueante del Paso 4

```
P4_VALIDO ↔ (monto > 0)
             AND (monto <= reserva.monto_total)
             AND (paid_at <= CURRENT_TIMESTAMP)
             AND (metodo_pago IN {"Efectivo", "Transferencia Bancaria", "Yape",
                                  "Plin", "Tarjeta Débito/Crédito", "Depósito"})
             AND ((total_pagado + monto) <= reserva.monto_total)  // No exceder el total
```

---

## 6. PASO 5: CONFIRMACIÓN

**Propósito:** Mostrar resumen final, generar documento y cerrar el proceso.

### Resumen que se Muestra al Usuario

```
╔══════════════════════════════════════════════════════════════════╗
║                    RESUMEN DE RESERVA                           ║
╠══════════════════════════════════════════════════════════════════╣
║ TITULAR:        [María Fernández]                               ║
║ VIAJE:          [Cumbemayo+Otuzco | 15/08/2026 | 08:00 hrs]    ║
║ PASAJEROS:      [2 personas]                                    ║
║ TOTAL:          [S/ 611.80]                                     ║
║ ADELANTO:       [S/ 200.00 - Yape]                              ║
║ SALDO:          [S/ 411.80]                                     ║
║ ESTADO:         [CONFIRMADA / PENDIENTE DE PAGO]                 ║
║                                                                  ║
║ [VOLVER A PASO 4]                     [CONFIRMAR RESERVA]        ║
╚══════════════════════════════════════════════════════════════════╝
```

### Validación Bloqueante del Paso 5

```
P5_VALIDO ↔ P1_VALIDO AND P2_VALIDO AND P3_VALIDO AND P4_VALIDO
```

### Acciones Atómicas al Confirmar

Cuando el usuario presiona "CONFIRMAR RESERVA", el sistema ejecuta en orden:

1. **Generar código único de reserva:** `ADV-{AÑO}-{CORRELATIVO}` usando el generador de secuencias
2. **Persistir la reserva** con todos los datos de los 5 pasos
3. **Establecer estado:**
   - `"CONFIRMADA"` si `total_pagado >= monto_total * 0.30`
   - `"PENDIENTE"` si `total_pagado < monto_total * 0.30`
   - `"BORRADOR"` si no se registró pago
4. **Registrar el pago** en la entidad `pagos` (si existe monto > 0)
5. **Actualizar la cotización origen:**
   - `cotizacion.estado → "RESERVA"`
   - `cotizacion.id_reserva → nueva_reserva.id`
6. **Registrar en auditoría:** acción = `"Conversión Cotización → Reserva"`
7. **Generar PDF de confirmación** (2 páginas)
8. **Retornar** la reserva creada con su código y datos

### Pseudocódigo del Flujo Completo

```
FUNCIÓN EjecutarFlujo5Pasos(datos):
    // Los datos se validan y acumulan paso a paso
    // Esta función se llama al final para crear todo
    
    // Paso 1: Validar y extraer
    cliente = datos.paso1.cliente
    titular_viaja = datos.paso1.titular_viaja
    
    // Paso 2: Validar y extraer
    travel_date = datos.paso2.fecha_viaje
    departure_time = datos.paso2.hora_salida
    pickup = datos.paso2.lugar_recojo
    return_date = datos.paso2.fecha_retorno
    route = datos.paso2.ruta
    mobility = datos.paso2.movilidad
    guide = datos.paso2.guia
    
    // Paso 3: Validar y extraer
    pasajeros = datos.paso3.pasajeros
    
    // Paso 4: Validar y extraer
    monto_pago = datos.paso4.monto
    metodo_pago = datos.paso4.metodo_pago
    referencia_pago = datos.paso4.referencia
    fecha_pago = datos.paso4.fecha_pago
    comprobante = datos.paso4.comprobante
    
    // Construir resultado
    resultado = {
        exitoso: TRUE,
        id_cliente: cliente.id,
        fecha_viaje: travel_date,
        hora_salida: departure_time,
        lugar_recojo: pickup,
        num_pasajeros: CONTAR(pasajeros),
        pasajeros: pasajeros,
        monto_total: cotizacion_asociada.total,
        total_pagado: monto_pago,
        saldo: cotizacion_asociada.total - monto_pago,
        monto_pago: monto_pago,
        metodo_pago: metodo_pago,
        referencia_pago: referencia_pago,
        fecha_pago: fecha_pago,
        comprobante: comprobante
    }
    
    RETORNAR resultado
FIN FUNCIÓN
```
