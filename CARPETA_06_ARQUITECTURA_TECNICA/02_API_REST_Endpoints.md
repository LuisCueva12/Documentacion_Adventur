# API REST — Especificación de Endpoints

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 02 — API REST Endpoints

---

## 1. Base URL

| Entorno | URL |
|---------|-----|
| Local | `http://adventur-api.test` |
| SiteGround | `https://api.adventur.com` | 
| VPS | `https://api.adventur.com` |

Todas las rutas usan **prefijo `/api`**.

---

## 2. Autenticación

### 2.1 Flujo de Autenticación

```
┌──────────┐         ┌─────────────┐         ┌──────────┐
│ Frontend │         │  Laravel    │         │   BD     │
│ (SPA)    │         │  (API)      │         │          │
└────┬─────┘         └──────┬──────┘         └────┬─────┘
     │                      │                     │
     │ POST /api/auth/login │                     │
     │ {email, password}    │                     │
     │─────────────────────►│                     │
     │                      │ Buscar usuario      │
     │                      │────────────────────►│
     │                      │◄────────────────────│
     │                      │                     │
     │                      │ Verificar password  │
     │                      │ Crear token         │
     │                      │                     │
     │◄─────────────────────│                     │
     │ {                    │                     │
     │   token: "1|xxx...", │                     │
     │   usuario: {         │                     │
     │     id, nombre,      │                     │
     │     email, rol       │                     │
     │   }                  │                     │
     │ }                    │                     │
     │                      │                     │
     │ [Todas las requests  │                     │
     │  siguientes llevan]  │                     │
     │ Authorization:       │                     │
     │   Bearer 1|xxx...    │                     │
     │─────────────────────►│                     │
```

### 2.2 Endpoints de Auth

#### `POST /api/auth/login`

**Body:**
```json
{
  "email": "juan@adventur.com",
  "password": "********"
}
```

**Response (200):**
```json
{
  "exitoso": true,
  "datos": {
    "token": "1|8xj3kL2s9f...",
    "tipo_token": "Bearer",
    "usuario": {
      "id": 1,
      "nombre": "Juan Pérez",
      "email": "juan@adventur.com",
      "rol": "Vendedor",
      "nivel_acceso": 20
    }
  },
  "mensaje": "Inicio de sesión exitoso"
}
```

**Response (401):**
```json
{
  "exitoso": false,
  "datos": null,
  "mensaje": "Credenciales inválidas",
  "errores": null
}
```

#### `POST /api/auth/logout`

**Headers:** `Authorization: Bearer {token}`

**Response (200):**
```json
{
  "exitoso": true,
  "datos": null,
  "mensaje": "Sesión cerrada correctamente"
}
```

#### `GET /api/auth/me`

**Headers:** `Authorization: Bearer {token}`

**Response (200):**
```json
{
  "exitoso": true,
  "datos": {
    "id": 1,
    "nombre": "Juan Pérez",
    "email": "juan@adventur.com",
    "rol": "Vendedor",
    "supervisor": {
      "id": 3,
      "nombre": "Carlos López"
    }
  }
}
```

---

## 3. Endpoints de Cotizaciones

### `GET /api/cotizaciones`

Lista paginada de cotizaciones.

**Query params:**
| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `pagina` | int | 1 | Número de página |
| `por_pagina` | int | 15 | Resultados por página |
| `estado` | string|null | null | Filtrar por estado |
| `id_vendedor` | int|null | null | Filtrar por vendedor |
| `id_cliente` | int|null | null | Filtrar por cliente |
| `fecha_desde` | date|null | null | Filtro fecha creación inicio |
| `fecha_hasta` | date|null | null | Filtro fecha creación fin |
| `busqueda` | string|null | null | Búsqueda por código o cliente |

**Response (200):**
```json
{
  "exitoso": true,
  "datos": [
    {
      "id": 1,
      "codigo": "COT-2026-0124",
      "estado": "Enviado",
      "cliente": {
        "id": 5,
        "nombre": "María Fernández",
        "documento": "DNI 12345678"
      },
      "vendedor": {
        "id": 1,
        "nombre": "Juan Pérez"
      },
      "total": "1250.00",
      "fecha_emision": "2026-06-15",
      "fecha_vencimiento": "2026-07-15",
      "dias_restantes": 4
    }
  ],
  "meta": {
    "pagina_actual": 1,
    "por_pagina": 15,
    "total": 234,
    "ultima_pagina": 16
  }
}
```

### `POST /api/cotizaciones`

Crea una nueva cotización en estado Borrador.

**Body:**
```json
{
  "id_cliente": 5,
  "id_vendedor": 1,
  "fecha_vencimiento": "2026-07-15",
  "items": [
    {
      "descripcion": "Tour Machu Picchu 2D/1N",
      "cantidad": 2,
      "precio_unitario": "450.00"
    },
    {
      "descripcion": "Seguro viajero",
      "cantidad": 2,
      "precio_unitario": "25.50"
    }
  ],
  "id_plantilla": null,
  "id_ruta": 3
}
```

**Response (201):**
```json
{
  "exitoso": true,
  "datos": {
    "id": 125,
    "codigo": "COT-2026-0125",
    "estado": "Borrador",
    "subtotal": "951.00",
    "descuento": "0.00",
    "total": "951.00",
    "items": [ ... ],
    "cliente": { ... },
    "fecha_creacion": "2026-07-11T14:30:00Z"
  },
  "mensaje": "Cotización creada correctamente"
}
```

### `GET /api/cotizaciones/{id}`

Obtiene detalle completo de una cotización.

**Response (200):**
```json
{
  "exitoso": true,
  "datos": {
    "id": 125,
    "codigo": "COT-2026-0125",
    "estado": "Borrador",
    "estados_permitidos": ["Enviado"],
    "cliente": {
      "id": 5,
      "nombre": "María Fernández",
      "documento": "DNI 12345678",
      "email": "maria@email.com",
      "telefono": "999888777"
    },
    "vendedor": { "id": 1, "nombre": "Juan Pérez" },
    "items": [
      {
        "descripcion": "Tour Machu Picchu 2D/1N",
        "cantidad": 2,
        "precio_unitario": "450.00",
        "subtotal": "900.00"
      }
    ],
    "subtotal": "951.00",
    "descuento": "0.00",
    "total": "951.00",
    "seguimientos": [ ... ],
    "historial_estados": [ ... ],
    "fecha_emision": "2026-07-11",
    "fecha_vencimiento": "2026-07-15"
  }
}
```

### `PUT /api/cotizaciones/{id}`

Actualiza cotización (solo en estado Borrador).

**Body:** (mismos campos que POST, todos opcionales en actualización)

**Response (200):** Cotización actualizada

### `DELETE /api/cotizaciones/{id}`

Anula (soft delete) una cotización.

**Response (200):**
```json
{
  "exitoso": true,
  "datos": null,
  "mensaje": "Cotización anulada correctamente"
}
```

### `POST /api/cotizaciones/{id}/enviar`

Cambia estado a Enviado. Registra seguimiento automático.

**Response (200):**
```json
{
  "exitoso": true,
  "datos": { "estado": "Enviado", "fecha_envio": "2026-07-11T15:00:00Z" },
  "mensaje": "Cotización enviada al cliente"
}
```

### `POST /api/cotizaciones/{id}/aprobar`

**Body:**
```json
{
  "porcentaje_descuento": "5.00",
  "id_supervisor_aprueba": 3
}
```

**Response (200):**
```json
{
  "exitoso": true,
  "datos": {
    "estado": "Aprobado",
    "descuento_aplicado": "47.55",
    "total_final": "903.45"
  },
  "mensaje": "Cotización aprobada"
}
```

**Response (403) - Descuento excede límite del rol:**
```json
{
  "exitoso": false,
  "datos": null,
  "mensaje": "El descuento solicitado (15.00%) excede el límite del vendedor (10.00%)",
  "errores": null
}
```

### `POST /api/cotizaciones/{id}/rechazar`

**Body:**
```json
{
  "motivo": "Cliente encontró mejor precio en otra agencia"
}
```

**Response (200):**
```json
{
  "exitoso": true,
  "datos": { "estado": "Rechazado" },
  "mensaje": "Cotización marcada como Rechazada"
}
```

### `POST /api/cotizaciones/{id}/perder`

**Body:**
```json
{
  "id_motivo_perdida": 3,
  "observacion": "Cliente no respondió después de 3 seguimientos"
}
```

**Response (200):**
```json
{
  "exitoso": true,
  "datos": { "estado": "Perdido" },
  "mensaje": "Cotización marcada como Perdida"
}
```

### `POST /api/cotizaciones/{id}/convertir`

Convierte cotización aprobada en reserva.

**Response (201):**
```json
{
  "exitoso": true,
  "datos": {
    "id_reserva": 47,
    "codigo_reserva": "ADV-2026-0047",
    "estado": "Borrador"
  },
  "mensaje": "Cotización convertida a reserva exitosamente"
}
```

---

## 4. Endpoints de Reservas

### `GET /api/reservas`

Lista paginada de reservas.

**Query params:**
| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `pagina` | int | 1 | |
| `por_pagina` | int | 15 | |
| `estado` | string|null | null | Borrador, Pendiente, Confirmada, Finalizada, Anulada |
| `id_vendedor` | int|null | null | |
| `fecha_desde` | date|null | null | |
| `fecha_hasta` | date|null | null | |
| `busqueda` | string|null | null | Código ADV, cliente, etc. |

### `POST /api/reservas`

Crea una reserva completa (flujo 5 pasos en una sola request).

**Body:**
```json
{
  "id_cliente": 5,
  "id_cotizacion": 125,
  "id_vendedor": 1,
  "fecha_viaje": "2026-08-15",
  "origen": "Lima",
  "destino": "Cusco",
  "movilidad_asignada": "Bus turístico",
  "guia_asignado": "Pedro Huamán",
  "conductor_asignado": "José García",
  "pasajeros": [
    {
      "nombre": "María Fernández",
      "documento": "DNI 12345678",
      "fecha_nacimiento": "1990-05-20",
      "es_titular": true,
      "contacto_emergencia": "Carlos Fernández",
      "telefono_emergencia": "999111222",
      "notas_salud": null
    },
    {
      "nombre": "Carlos Fernández",
      "documento": "DNI 87654321",
      "fecha_nacimiento": "1988-11-10",
      "es_titular": false,
      "contacto_emergencia": "María Fernández",
      "telefono_emergencia": "999333444",
      "notas_salud": "Alérgico a mariscos"
    }
  ],
  "alojamientos": [
    {
      "hotel_nombre": "Hotel Cusco Plaza",
      "noche_numero": 1,
      "tipo_habitacion": "Doble",
      "tipo_cama": "Queen",
      "plan_comidas": "Desayuno incluido",
      "check_in": "2026-08-15",
      "check_out": "2026-08-16"
    }
  ],
  "servicios_adicionales": [
    {
      "descripcion": "Tour Valle Sagrado",
      "tipo_servicio": "Tour adicional",
      "precio_unitario": "80.00",
      "cantidad": 2,
      "fecha_servicio": "2026-08-16"
    }
  ],
  "pagos": [
    {
      "monto": "500.00",
      "metodo_pago": "Yape",
      "referencia": "PAGO-YAPE-001",
      "fecha_pago": "2026-07-11"
    }
  ],
  "monto_total": "1250.00"
}
```

**Response (201):**
```json
{
  "exitoso": true,
  "datos": {
    "id": 47,
    "codigo": "ADV-2026-0047",
    "estado": "Pendiente",
    "saldo_pendiente": "750.00"
  },
  "mensaje": "Reserva creada correctamente"
}
```

### `GET /api/reservas/{id}`

Detalle completo de reserva (incluye pasajeros, pagos, alojamientos, servicios).

### `PUT /api/reservas/{id}`

Actualiza reserva (solo en Borrador o Pendiente).

### `DELETE /api/reservas/{id}`

Anula reserva (soft delete - estado Anulada).

### `POST /api/reservas/{id}/pagos`

Registra un pago adicional.

**Body:**
```json
{
  "monto": "300.00",
  "metodo_pago": "Transferencia",
  "referencia": "TRA-2026-0891",
  "fecha_pago": "2026-07-20",
  "comprobante": "data:image/jpeg;base64,..."
}
```

**Response (200):**
```json
{
  "exitoso": true,
  "datos": {
    "id_pago": 15,
    "total_pagado": "800.00",
    "saldo_pendiente": "450.00",
    "reserva_confirmada": false
  },
  "mensaje": "Pago registrado correctamente"
}
```

### `GET /api/reservas/{id}/pdf`

Genera y descarga el PDF de confirmación de 2 páginas.

**Response:** Binary PDF file (Content-Type: application/pdf)

---

## 5. Endpoints de Clientes

### `GET /api/clientes`

Lista paginada de clientes.

### `POST /api/clientes`

**Body:**
```json
{
  "tipo_documento": "DNI",
  "numero_documento": "12345678",
  "nombre": "María Fernández",
  "email": "maria@email.com",
  "telefono": "999888777",
  "direccion": "Av. Principal 123, Lima"
}
```

### `GET /api/clientes/{id}`

### `GET /api/clientes/buscar`

**Query params:** `q=12345678` (busca por DNI, email o teléfono)

---

## 6. Endpoints de Leads

### `GET /api/leads`
### `POST /api/leads`
### `GET /api/leads/{id}`
### `PUT /api/leads/{id}`
### `POST /api/leads/{id}/convertir`

---

## 7. Endpoints de Reportes

### `GET /api/reportes/ventas`

**Query params:** `fecha_desde`, `fecha_hasta`, `id_vendedor`

**Response:**
```json
{
  "exitoso": true,
  "datos": {
    "total_ventas": "45890.00",
    "total_reservas": 34,
    "promedio_por_reserva": "1350.00",
    "por_asesor": [
      { "vendedor": "Juan Pérez", "total": "15230.00", "reservas": 12 },
      { "vendedor": "Ana López", "total": "18900.00", "reservas": 14 }
    ]
  }
}
```

### `GET /api/reportes/conversion`

**Query params:** `fecha_desde`, `fecha_hasta`

```json
{
  "datos": {
    "total_cotizaciones": 150,
    "total_reservas": 34,
    "tasa_conversion": "22.67%",
    "por_motivo_perdida": [
      { "motivo": "Precio alto", "cantidad": 45, "porcentaje": "30.00%" },
      { "motivo": "No respondió", "cantidad": 30, "porcentaje": "20.00%" }
    ]
  }
}
```

### `GET /api/reportes/rendimiento`

**Query params:** `fecha_desde`, `fecha_hasta`

```json
{
  "datos": {
    "mejor_vendedor": { "nombre": "Ana López", "conversion": "35.00%" },
    "rutas_mas_vendidas": [
      { "ruta": "Machu Picchu", "veces": 15 },
      { "ruta": "Huacachina", "veces": 8 }
    ]
  }
}
```

---

## 8. Endpoints de Dashboard

### `GET /api/dashboard/resumen`

Resumen ejecutivo para la pantalla principal.

```json
{
  "datos": {
    "cotizaciones_hoy": 5,
    "reservas_pendientes": 12,
    "proximas_salidas": [
      {
        "codigo": "ADV-2026-0047",
        "cliente": "María Fernández",
        "fecha_viaje": "2026-07-15",
        "destino": "Cusco",
        "saldo_pendiente": "450.00"
      }
    ],
    "alertas": [
      "3 cotizaciones por vencer hoy",
      "1 seguimiento vencido"
    ],
    "grafico_ventas_mensuales": {
      "labels": ["Ene", "Feb", "Mar", "Abr", "May", "Jun"],
      "datos": [1200, 3400, 5600, 7800, 4500, 8900]
    }
  }
}
```

---

## 9. Endpoints de Administración

### `GET /api/admin/usuarios`
### `POST /api/admin/usuarios`
### `PUT /api/admin/usuarios/{id}/rol`
### `GET /api/admin/configuracion/politicas`

Devuelve, en orden de presentación, las cuatro políticas que alimentan la segunda página del PDF: cancelación, pagos, responsabilidad y términos generales. Requiere permiso `politicas.ver`.

### `PUT /api/admin/configuracion/politicas`

Actualiza una o varias políticas. Requiere permiso `politicas.editar`; cada cambio incrementa la versión y genera un registro de auditoría.

```json
{
  "politicas": [
    {
      "tipo": "politica_cancelacion",
      "contenido": "Texto vigente que aparecerá en el documento."
    }
  ]
}
```

Los tipos permitidos son `politica_cancelacion`, `politica_pagos`, `politica_responsabilidad` y `terminos_servicio`. Configuraciones numéricas como IGV, vigencia o porcentaje de adelanto no se renderizan como políticas.

### `GET /api/admin/catalogos/motivos-perdida`
### `GET /api/admin/catalogos/canales-origen`
### `GET /api/admin/catalogos/plantillas`
### `GET /api/admin/catalogos/rutas`

---

## 10. Endpoints de Auditoría

### `GET /api/auditoria`

Lista el historial de cambios (solo lectura, R3).

**Query params:** `tipo_entidad`, `id_entidad`, `pagina`, `por_pagina`

```json
{
  "datos": [
    {
      "fecha": "2026-07-11T14:30:00Z",
      "usuario": "Juan Pérez",
      "accion": "Creación",
      "entidad": "Cotización",
      "id_entidad": 125,
      "detalle": "Creada cotización COT-2026-0125 por $951.00"
    }
  ]
}
```

---

## 11. Matriz de Permisos por Endpoint

| Endpoint | Admin | Vendedor | Supervisor | Operaciones | Contabilidad | Gerencia |
|----------|:-----:|:--------:|:----------:|:-----------:|:------------:|:--------:|
| `/api/auth/*` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `/api/cotizaciones` GET | ✅ | Solo propias | ✅ | ❌ | ❌ | ✅ |
| `/api/cotizaciones` POST | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `/api/cotizaciones/{id}/aprobar` | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| `/api/cotizaciones/{id}/convertir` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `/api/reservas` GET | ✅ | Solo propias | ✅ | ✅ | ✅ | ✅ |
| `/api/reservas` POST | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| `/api/reservas/{id}/pdf` | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| `/api/pagos` | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| `/api/reportes/*` | ✅ | Solo propio | ✅ | ✅ | ✅ | ✅ |
| `/api/admin/*` | ✅ | ❌ | ❌ | ❌ | ❌ | Solo lectura |
| `/api/auditoria` | ✅ | ❌ | Solo equipo | ❌ | ❌ | ✅ |

---

## 12. Códigos de Error de Negocio

| Código | HTTP | Significado |
|--------|:----:|-------------|
| `REGLA_NEGOCIO` | 409 | Violación de regla de negocio (R1-R4, RF) |
| `TRANSICION_INVALIDA` | 409 | Estado no permitido desde estado actual |
| `NO_ENCONTRADO` | 404 | Entidad no existe |
| `NO_AUTORIZADO` | 403 | Rol no tiene permiso |
| `VALIDACION` | 422 | Datos inválidos |
| `DESCUENTO_EXCEDIDO` | 403 | Límite de descuento superado |
| `CLIENTE_INACTIVO` | 409 | Cliente no puede operar |
| `COTIZACION_YA_CONVERTIDA` | 409 | Cotización ya generó una reserva (R1) |
| `RESERVA_NO_MODIFICABLE` | 409 | Estado de reserva no permite modificación |
| `SALDO_INSUFICIENTE` | 409 | Adelanto mínimo no cubierto |
| `PAGO_ANULADO` | 409 | No se puede modificar un pago anulado |
