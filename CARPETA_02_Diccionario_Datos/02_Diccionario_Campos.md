# Diccionario de Campos por Entidad

**Carpeta:** 02 — Diccionario de Datos Agnóstico  
**Versión:** 1.0 — Especificación Agnóstica

---

## Convención de Tipos Lógicos

| Tipo Lógico | Descripción | Ejemplo |
|-------------|-------------|---------|
| `ID` | Identificador único interno | 1, 2, 3... |
| `Texto(N)` | Cadena de texto de hasta N caracteres | "María Fernández" |
| `Texto(L)` | Cadena de texto de longitud libre (ilimitada) | "Observaciones..." |
| `Entero` | Número entero sin decimales | 4, 42, 100 |
| `Decimal(P,S)` | Número decimal con P dígitos totales y S decimales | Decimal(10,2) = 611.80 |
| `Fecha` | Fecha calendario (sin hora) | 2026-08-15 |
| `Hora` | Hora del día (HH:MM) | 08:00 |
| `FechaHora` | Fecha y hora combinadas | 2026-07-26 15:30:00 |
| `Booleano` | Valor verdadero/falso | TRUE, FALSE |
| `Lista(Enum)` | Conjunto fijo de valores permitidos | "DNI", "Pasaporte", "RUC" |

---

## E01: usuarios

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | nombre_completo | Texto(255) | SÍ | Mínimo 3 caracteres |
| 3 | correo_electronico | Texto(255) | SÍ | **ÚNICO.** Debe contener `@`. Formato: `usuario@dominio.tld` |
| 4 | contraseña | Texto(cifrado) | SÍ | Mínimo 8 caracteres. Almacenada mediante función hash unidireccional (nunca texto plano) |
| 5 | rol | Lista(Enum) | SÍ | Valores: `Administrador`, `Vendedor`, `Supervisor`, `Operaciones`, `Contabilidad`, `Gerencia` |
| 6 | telefono | Texto(20) | NO | Solo dígitos, guiones y signo `+` |
| 7 | activo | Booleano | SÍ | Default: `TRUE`. Si `FALSE`: no puede iniciar sesión |
| 8 | id_supervisor | ID | NO | Debe referenciar un `usuarios.id` existente con rol `Supervisor` |
| 9 | ultimo_acceso | FechaHora | NO | Se actualiza automáticamente al iniciar sesión |
| 10 | fecha_creacion | FechaHora | SÍ | Default: `CURRENT_TIMESTAMP` |
| 11 | fecha_actualizacion | FechaHora | SÍ | Se actualiza automáticamente en cada modificación |

**Unicidad compuesta:** `correo_electronico` — No pueden existir dos usuarios con el mismo email.

---

## E02: clientes

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(20) | SÍ | **ÚNICO.** Formato: `CLI-{AÑO}-{CORRELATIVO}` |
| 3 | tipo_documento | Lista(Enum) | NO | Valores: `DNI`, `Pasaporte`, `RUC`, `Carné Extranjería` |
| 4 | numero_documento | Texto(20) | Condicional | SÍ si `tipo_documento != NULL`. DNI: 8 dígitos. Pasaporte: 12 alfanumérico. RUC: 11 dígitos |
| 5 | nombre_completo | Texto(255) | SÍ | Mínimo 2 palabras (nombres + apellidos) |
| 6 | telefono | Texto(20) | SÍ | Mínimo 9 dígitos. Solo dígitos |
| 7 | correo_electronico | Texto(255) | NO | Si se proporciona: debe contener `@` y dominio válido |
| 8 | direccion | Texto(L) | NO | |
| 9 | ciudad | Texto(100) | NO | |
| 10 | nacionalidad | Texto(100) | NO | Default: `Peruana` |
| 11 | fecha_nacimiento | Fecha | NO | Si se proporciona: `fecha_nacimiento <= CURRENT_DATE` |
| 12 | contacto_emergencia | Texto(255) | NO | |
| 13 | telefono_emergencia | Texto(20) | NO | |
| 14 | notas_salud | Texto(L) | NO | Alergias, condiciones médicas preexistentes |
| 15 | convertido_de_lead | Booleano | SÍ | Default: `FALSE`. Se marca `TRUE` cuando se origina desde un lead |
| 16 | canal_origen | Texto(50) | NO | Referencia a `canales_origen.codigo` |
| 17 | creado_por | ID | SÍ | Referencia a `usuarios.id` |
| 18 | fecha_creacion | FechaHora | SÍ | Default: `CURRENT_TIMESTAMP` |
| 19 | fecha_actualizacion | FechaHora | SÍ | Se actualiza automáticamente |
| 20 | fecha_eliminacion | FechaHora | NO | Soft delete. `NULL` = activo. Timestamp = eliminado |

**Unicidad compuesta:** `(codigo)`, `(tipo_documento, numero_documento)` — solo cuando ambos están presentes y no son NULL.

---

## E03: leads

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(20) | SÍ | **ÚNICO.** Formato: `LEAD-{AÑO}-{CORRELATIVO}` |
| 3 | nombre_completo | Texto(255) | SÍ | Mínimo 3 caracteres |
| 4 | telefono | Texto(20) | SÍ | Mínimo 9 dígitos |
| 5 | correo_electronico | Texto(255) | NO | |
| 6 | canal_origen | Texto(50) | NO | Referencia a `canales_origen.codigo` |
| 7 | ruta_interes | Texto(L) | NO | Destino o paquete de interés |
| 8 | fecha_viaje_estimada | Fecha | NO | |
| 9 | num_pasajeros | Entero | SÍ | Default: `1`. Rango: `>= 1` |
| 10 | observacion_inicial | Texto(L) | NO | |
| 11 | id_cliente | ID | NO | Referencia a `clientes.id`. Se vincula al identificar el lead como cliente existente |
| 12 | asignado_a | ID | SÍ | Referencia a `usuarios.id` (vendedor responsable) |
| 13 | estado | Lista(Enum) | SÍ | Valores: `Nuevo`, `Cotizado`, `Perdido` |
| 14 | fecha_conversion | FechaHora | NO | Se registra cuando se crea la primera cotización |
| 15 | fecha_creacion | FechaHora | SÍ | Default: `CURRENT_TIMESTAMP` |
| 16 | fecha_actualizacion | FechaHora | SÍ | Se actualiza automáticamente |
| 17 | fecha_eliminacion | FechaHora | NO | Soft delete |

---

## E04: cotizaciones

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(20) | SÍ | **ÚNICO.** Formato: `COT-{AÑO}-{CORRELATIVO}` |
| 3 | id_lead | ID | NO | Referencia a `leads.id` |
| 4 | id_cliente | ID | NO | Referencia a `clientes.id` |
| 5 | id_plantilla | ID | NO | Referencia a `plantillas.id` |
| 6 | num_pasajeros | Entero | SÍ | `>= 1` |
| 7 | base_movilidad | Decimal(10,2) | SÍ | `>= 0.00` |
| 8 | base_guia | Decimal(10,2) | SÍ | `>= 0.00` |
| 9 | base_entradas | Decimal(10,2) | SÍ | `>= 0.00` |
| 10 | base_alimentacion | Decimal(10,2) | SÍ | `>= 0.00` |
| 11 | base_extras | Decimal(10,2) | SÍ | `>= 0.00` |
| 12 | base_total | Decimal(10,2) | SÍ (calculado) | `= base_movilidad + base_guia + base_entradas + base_alimentacion + base_extras` |
| 13 | porcentaje_margen | Decimal(5,2) | SÍ | `>= 0.00`. Ej: 15.00 = 15% |
| 14 | monto_margen | Decimal(10,2) | SÍ (calculado) | `= base_total * (porcentaje_margen / 100)` |
| 15 | porcentaje_descuento | Decimal(5,2) | SÍ | `>= 0.00`. Default: 0.00 |
| 16 | monto_descuento | Decimal(10,2) | SÍ (calculado) | `= (base_total + monto_margen) * (porcentaje_descuento / 100)` |
| 17 | descuento_aprobado_por | ID | NO | Referencia a `usuarios.id`. NULL si no requiere aprobación |
| 18 | total | Decimal(10,2) | SÍ (calculado) | `= base_total + monto_margen - monto_descuento` |
| 19 | precio_por_persona | Decimal(10,2) | SÍ (calculado) | `= total / num_pasajeros` |
| 20 | dias_validez | Entero | SÍ | Valores permitidos: `7`, `15`, `30` |
| 21 | valido_hasta | Fecha | SÍ (calculado) | `= CURRENT_DATE + dias_validez` |
| 22 | estado | Lista(Enum) | SÍ | Valores: `Nuevo`, `Cotizado`, `Enviado`, `En seguimiento`, `Aprobado`, `Reserva`, `Perdido` |
| 23 | fecha_envio | FechaHora | NO | Se registra al enviar la cotización |
| 24 | enviado_via | Lista(Enum) | NO | Valores: `WhatsApp`, `Email`, `Manual` |
| 25 | visto_por_cliente | Booleano | SÍ | Default: `FALSE` |
| 26 | id_motivo_perdida | ID | NO | Referencia a `motivos_perdida.id`. Obligatorio si estado = `Perdido` |
| 27 | observacion_perdida | Texto(L) | NO | Obligatorio si estado = `Perdido`. Mínimo 10 caracteres |
| 28 | fecha_perdida | FechaHora | NO | Se registra al marcar como perdido |
| 29 | id_reserva | ID | NO | Referencia a `reservas.id`. Se vincula al convertir |
| 30 | asignado_a | ID | SÍ | Referencia a `usuarios.id` (vendedor responsable) |
| 31 | creado_por | ID | SÍ | Referencia a `usuarios.id` |
| 32 | notas_internas | Texto(L) | NO | |
| 33 | fecha_creacion | FechaHora | SÍ | |
| 34 | fecha_actualizacion | FechaHora | SÍ | |
| 35 | fecha_eliminacion | FechaHora | NO | Soft delete |

---

## E05: cotizacion_detalle

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_cotizacion | ID | SÍ | Referencia a `cotizaciones.id`. Borrado en cascada |
| 3 | descripcion | Texto(500) | SÍ | |
| 4 | tipo | Lista(Enum) | SÍ | Valores: `Servicio`, `Extra`, `Descuento` |
| 5 | precio_unitario | Decimal(10,2) | SÍ | `>= 0.00` |
| 6 | cantidad | Entero | SÍ | `>= 1` |
| 7 | subtotal | Decimal(10,2) | SÍ (calculado) | `= precio_unitario * cantidad` |

---

## E06: plantillas

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | nombre | Texto(255) | SÍ | **ÚNICO.** Mínimo 3 caracteres. Ej: "Cajamarca Express", "Cumbemayo + Otuzco" |
| 3 | id_ruta | ID | NO | Referencia a `rutas.id` |
| 4 | descripcion | Texto(L) | NO | |
| 5 | default_movilidad | Decimal(10,2) | SÍ | `>= 0.00` |
| 6 | default_guia | Decimal(10,2) | SÍ | `>= 0.00` |
| 7 | default_entradas | Decimal(10,2) | SÍ | `>= 0.00` |
| 8 | default_alimentacion | Decimal(10,2) | SÍ | `>= 0.00` |
| 9 | default_extras | Decimal(10,2) | SÍ | `>= 0.00` |
| 10 | default_margen | Decimal(5,2) | SÍ | `>= 0.00` |
| 11 | dias_validez | Entero | SÍ | Valores: `7`, `15`, `30` |
| 12 | activa | Booleano | SÍ | Default: `TRUE`. Si `FALSE`: no aparece en selector de cotizaciones |
| 13 | creado_por | ID | SÍ | Referencia a `usuarios.id` |
| 14 | fecha_creacion | FechaHora | SÍ | |
| 15 | fecha_actualizacion | FechaHora | SÍ | |

---

## E07: rutas

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | nombre | Texto(255) | SÍ | **ÚNICO.** |
| 3 | descripcion | Texto(L) | NO | |
| 4 | duracion_horas | Entero | NO | `> 0` si se especifica |
| 5 | duracion_dias | Entero | NO | `> 0` si se especifica |
| 6 | incluye | Texto(L) | NO | Lista de servicios incluidos |
| 7 | no_incluye | Texto(L) | NO | Lista de lo no incluido |
| 8 | activa | Booleano | SÍ | Default: `TRUE` |

---

## E08: seguimientos

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_cotizacion | ID | SÍ | Referencia a `cotizaciones.id`. Borrado en cascada |
| 3 | fecha_programada | FechaHora | SÍ | `> CURRENT_TIMESTAMP` (fecha futura) |
| 4 | fecha_realizada | FechaHora | NO | Se registra al completar el seguimiento |
| 5 | medio_contacto | Texto(50) | NO | Valores: `Llamada`, `WhatsApp`, `Email`, `Presencial` |
| 6 | resultado | Texto(L) | NO | Descripción del resultado del contacto |
| 7 | proxima_accion | Texto(L) | NO | |
| 8 | id_reprogramacion | ID | NO | Referencia a `seguimientos.id`. Se vincula al reprogramar |
| 9 | estado | Lista(Enum) | SÍ | Valores: `Programado`, `Realizado`, `Reprogramado`, `Vencido` |
| 10 | creado_por | ID | SÍ | Referencia a `usuarios.id` |

**Regla de vencimiento automático:** Un seguimiento en estado `Programado` pasa a `Vencido` cuando `CURRENT_TIMESTAMP > fecha_programada + 24 horas`.

---

## E09: motivos_perdida

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(10) | SÍ | **ÚNICO.** Formato: `L-{NN}` |
| 3 | nombre | Texto(255) | SÍ | **ÚNICO.** |
| 4 | requiere_observacion | Booleano | SÍ | Default: `TRUE`. Si `TRUE`, el sistema obliga a escribir observación de pérdida |
| 5 | activo | Booleano | SÍ | Default: `TRUE` |
| 6 | orden | Entero | SÍ | Default: `0`. Controla el orden de aparición en selectores |

---

## E10: reservas

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(20) | SÍ | **ÚNICO.** Formato: `ADV-{AÑO}-{CORRELATIVO}` |
| 3 | id_cotizacion | ID | SÍ | Referencia a `cotizaciones.id`. Relación 1:1 (una cotización → máximo una reserva) |
| 4 | id_cliente | ID | SÍ | Referencia a `clientes.id` |
| 5 | fecha_viaje | Fecha | SÍ | `> CURRENT_DATE` al momento de creación |
| 6 | hora_salida | Hora | NO | |
| 7 | lugar_recojo | Texto(255) | NO | |
| 8 | fecha_retorno | Fecha | NO | `>= fecha_viaje` |
| 9 | num_pasajeros | Entero | SÍ | `>= 1` |
| 10 | monto_total | Decimal(10,2) | SÍ | `>= 0.00`. Proviene de la cotización origen |
| 11 | total_pagado | Decimal(10,2) | SÍ (calculado) | `= SUM(pagos.monto WHERE pagos.id_reserva = id AND pagos.fecha_anulacion IS NULL)` |
| 12 | saldo | Decimal(10,2) | SÍ (calculado) | `= monto_total - total_pagado`. Siempre `>= 0.00` |
| 13 | estado | Lista(Enum) | SÍ | Valores: `Borrador`, `Pendiente`, `Confirmada`, `Finalizada`, `Anulada` |
| 14 | movilidad_asignada | Texto(255) | NO | |
| 15 | guia_asignado | Texto(255) | NO | |
| 16 | conductor_asignado | Texto(255) | NO | |
| 17 | notas_operativas | Texto(L) | NO | |
| 18 | motivo_anulacion | Texto(L) | Condicional | SÍ si estado = `Anulada` |
| 19 | fecha_anulacion | FechaHora | NO | |
| 20 | anulado_por | ID | NO | Referencia a `usuarios.id` |
| 21 | creado_por | ID | SÍ | Referencia a `usuarios.id` |
| 22 | fecha_creacion | FechaHora | SÍ | |
| 23 | fecha_actualizacion | FechaHora | SÍ | |
| 24 | fecha_eliminacion | FechaHora | NO | Soft delete |

---

## E11: pasajeros

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_reserva | ID | SÍ | Referencia a `reservas.id`. Borrado en cascada |
| 3 | nombre_completo | Texto(255) | SÍ | Mínimo 2 palabras |
| 4 | tipo_documento | Lista(Enum) | NO | Valores: `DNI`, `Pasaporte`, `Carné Extranjería` |
| 5 | numero_documento | Texto(20) | Condicional | SÍ si tipo_documento != NULL |
| 6 | fecha_nacimiento | Fecha | NO | |
| 7 | edad | Entero | NO (calculado) | Si `fecha_nacimiento` presente: `edad = (CURRENT_DATE - fecha_nacimiento).years` |
| 8 | telefono | Texto(20) | NO | |
| 9 | correo_electronico | Texto(255) | NO | |
| 10 | contacto_emergencia | Texto(255) | SÍ | |
| 11 | telefono_emergencia | Texto(20) | SÍ | |
| 12 | notas_salud | Texto(L) | NO | |
| 13 | requiere_seguro | Booleano | SÍ | Default: `FALSE` |
| 14 | es_titular | Booleano | SÍ | Exactamente 1 por reserva debe ser `TRUE` |

---

## E12: pagos

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_reserva | ID | SÍ | Referencia a `reservas.id`. Borrado en cascada |
| 3 | monto | Decimal(10,2) | SÍ | `> 0.00` |
| 4 | metodo_pago | Lista(Enum) | SÍ | Valores: `Efectivo`, `Transferencia`, `Yape`, `Plin`, `Tarjeta`, `Depósito` |
| 5 | referencia | Texto(255) | NO | Número de operación bancaria o voucher |
| 6 | fecha_pago | FechaHora | SÍ | `<= CURRENT_TIMESTAMP` |
| 7 | recibido_por | ID | SÍ | Referencia a `usuarios.id` |
| 8 | ruta_comprobante | Texto(500) | NO | Ruta al archivo digital del comprobante (PDF/JPG/PNG) |
| 9 | notas | Texto(L) | NO | |
| 10 | es_adelanto | Booleano | SÍ | `TRUE` si `monto < reserva.monto_total`. `FALSE` si `monto = reserva.monto_total` |
| 11 | fecha_anulacion | FechaHora | NO | |
| 12 | anulado_por | ID | NO | Referencia a `usuarios.id` |
| 13 | motivo_anulacion | Texto(L) | Condicional | SÍ si se anula |

---

## E13: politicas

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | titulo | Texto(255) | SÍ | Ej: "Política de Cancelación", "Política de Pagos" |
| 3 | contenido | Texto(L) | SÍ | |
| 4 | seccion | Lista(Enum) | SÍ | Valores: `Cancelación`, `Pago`, `Responsabilidad`, `General` |
| 5 | orden | Entero | SÍ | Default: `0`. Controla orden de aparición en PDF |
| 6 | activa | Booleano | SÍ | Default: `TRUE` |
| 7 | creado_por | ID | SÍ | Referencia a `usuarios.id` |

---

## E14: auditoria

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_usuario | ID | NO | Referencia a `usuarios.id`. NULL si acción del sistema (automática) |
| 3 | accion | Lista(Enum) | SÍ | Valores: `Creación`, `Modificación`, `Envío`, `Aprobación`, `Anulación`, `Pago`, `Conversión`, `Reactivación` |
| 4 | tipo_entidad | Texto(100) | SÍ | Nombre de la entidad afectada: `Cotización`, `Reserva`, `Pago`, etc. |
| 5 | id_entidad | ID | SÍ | ID del registro afectado |
| 6 | valores_anteriores | Texto(L) | NO | Pares `campo=valor` separados por `\|`. Ej: `estado=enviado\|total=500.00` |
| 7 | valores_nuevos | Texto(L) | NO | Pares `campo=valor` separados por `\|` |
| 8 | direccion_ip | Texto(45) | NO | IPv4 o IPv6 |
| 9 | agente_usuario | Texto(L) | NO | Cadena User-Agent del navegador/cliente |
| 10 | fecha_creacion | FechaHora | SÍ | Default: `CURRENT_TIMESTAMP`. **INMUTABLE: no se actualiza nunca** |

---

## E15: canales_origen

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | codigo | Texto(10) | SÍ | **ÚNICO.** Formato: `C-{NN}` |
| 3 | nombre | Texto(100) | SÍ | **ÚNICO.** |
| 4 | activo | Booleano | SÍ | Default: `TRUE` |

---

## E16: secuencias_codigos

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | prefijo | Texto(10) | SÍ | Valores: `COT`, `ADV`, `CLI`, `LEAD` |
| 3 | año | Entero | SÍ | Ej: 2026 |
| 4 | ultimo_numero | Entero | SÍ | `>= 0`. Default: 0 |

**Unicidad compuesta:** `(prefijo, año)` — No puede haber dos secuencias para el mismo prefijo y año.

---

## E17: alojamientos

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_reserva | ID | SÍ | Referencia a `reservas.id`. Borrado en cascada |
| 3 | noche_numero | Entero | SÍ | `>= 1`. Ej: 1 = primera noche, 2 = segunda noche |
| 4 | hotel_nombre | Texto(255) | SÍ | |
| 5 | tipo_habitacion | Lista(Enum) | SÍ | Valores: `Individual`, `Doble`, `Triple`, `Compartida`, `Suite` |
| 6 | tipo_cama | Lista(Enum) | NO | Valores: `Queen`, `King`, `Dos camas individuales`, `Litera`, `Sofá cama` |
| 7 | plan_comidas | Lista(Enum) | NO | Valores: `Solo desayuno`, `Media pensión`, `Pensión completa`, `Todo incluido` |
| 8 | check_in | Fecha | SÍ | `check_in >= reserva.fecha_viaje` |
| 9 | check_out | Fecha | SÍ | `check_out > check_in` |
| 10 | notas_alojamiento | Texto(L) | NO | Preferencias de habitación, solicitudes especiales |
| 11 | fecha_creacion | FechaHora | SÍ | |
| 12 | fecha_actualizacion | FechaHora | SÍ | |
| 13 | fecha_eliminacion | FechaHora | NO | Soft delete |

---

## E18: servicios_reserva

| # | Campo | Tipo Lógico | Obligatorio | Validación / Límite |
|---|-------|-------------|-------------|---------------------|
| 1 | id | ID | SÍ | Único, autogenerado |
| 2 | id_reserva | ID | SÍ | Referencia a `reservas.id`. Borrado en cascada |
| 3 | descripcion | Texto(500) | SÍ | |
| 4 | tipo_servicio | Lista(Enum) | SÍ | Valores: `Seguro viajero`, `Tour adicional`, `Traslado extra`, `Cena especial`, `Equipo adicional`, `Otro` |
| 5 | proveedor | Texto(255) | NO | Nombre del proveedor del servicio |
| 6 | precio_unitario | Decimal(10,2) | SÍ | `>= 0.00` |
| 7 | cantidad | Entero | SÍ | `>= 1` |
| 8 | subtotal | Decimal(10,2) | SÍ (calculado) | `= precio_unitario * cantidad` |
| 9 | fecha_servicio | Fecha | NO | Fecha en que se presta el servicio |
| 10 | notas | Texto(L) | NO | |
| 11 | fecha_creacion | FechaHora | SÍ | |
| 12 | fecha_actualizacion | FechaHora | SÍ | |
| 13 | fecha_eliminacion | FechaHora | NO | Soft delete |
