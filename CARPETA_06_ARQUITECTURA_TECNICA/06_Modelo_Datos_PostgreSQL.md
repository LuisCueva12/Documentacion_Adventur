# Modelo de Datos PostgreSQL — Adventur Travel

**Carpeta:** 06 — Arquitectura Técnica  
**Versión:** 1.0 — DDL de Implementación  
**Motor:** PostgreSQL 16+  
**Encoding:** UTF8  
**Schema:** `public` (por defecto)

---

## 1. Introducción

Este documento define el modelo de datos físico del sistema Adventur Travel implementado sobre PostgreSQL 16+. Se utiliza el schema `public` por defecto, encoding `UTF8` y `LC_COLLATE` en `es_PE.UTF-8`. Todos los identificadores (nombres de tabla, columna, constraint) están en español. Las tablas se nombran en plural.

**Convenciones:**

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Tablas | Plural, snake_case | `cotizaciones`, `motivos_perdida` |
| Columnas | snake_case | `nombre_completo`, `fecha_creacion` |
| PK | `id SERIAL PRIMARY KEY` en toda tabla | |
| FK | `campo_id INTEGER REFERENCES tabla(id)` | |
| Timestamps | `fecha_creacion`, `fecha_actualizacion` | `DEFAULT CURRENT_TIMESTAMP` |
| Soft delete | `fecha_eliminacion TIMESTAMP DEFAULT NULL` | |
| Monetario | `NUMERIC(10,2)` | Truncado a 2 decimales (Regla R2) |
| Porcentaje | `NUMERIC(5,2)` | Ej: `15.00` = 15% |

---

## 2. Tipos Personalizados (ENUMs)

```sql
CREATE TYPE tipo_documento AS ENUM ('DNI', 'RUC', 'CE', 'Pasaporte');

CREATE TYPE estado_cotizacion AS ENUM (
    'Propuesta', 'Enviado', 'Visto', 'Aprobado', 'Convertido', 'Perdido'
);

CREATE TYPE estado_reserva AS ENUM (
    'Borrador', 'Pendiente', 'Confirmada', 'Finalizada', 'Anulada'
);

CREATE TYPE estado_lead AS ENUM ('Nuevo', 'Cotizado', 'Perdido', 'Convertido');

CREATE TYPE estado_pago AS ENUM ('Registrado', 'Anulado');

CREATE TYPE estado_seguimiento AS ENUM (
    'Programado', 'Realizado', 'Vencido', 'Reprogramado'
);

CREATE TYPE rol_usuario AS ENUM (
    'Administrador', 'Vendedor', 'Supervisor', 'Operaciones', 'Contabilidad', 'Gerencia'
);

CREATE TYPE tipo_habitacion AS ENUM (
    'Individual', 'Doble', 'Triple', 'Compartida', 'Suite'
);

CREATE TYPE tipo_cama AS ENUM (
    'Queen', 'King', 'Dos_camas_individuales', 'Litera', 'Sofa_cama'
);

CREATE TYPE plan_comidas AS ENUM (
    'Solo_desayuno', 'Media_pension', 'Pension_completa', 'Todo_incluido'
);

CREATE TYPE tipo_servicio AS ENUM (
    'Seguro_viajero', 'Tour_adicional', 'Traslado_extra', 'Cena_especial',
    'Equipo_adicional', 'Otro'
);

CREATE TYPE metodo_pago AS ENUM (
    'Efectivo', 'Yape', 'Plin', 'Transferencia', 'Tarjeta_credito',
    'Tarjeta_debito', 'Deposito'
);
```

---

## 3. Tablas del Sistema

### 3.1 usuarios

```sql
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre_completo VARCHAR(255) NOT NULL,
    correo_electronico VARCHAR(255) NOT NULL,
    contrasena VARCHAR(255) NOT NULL,
    rol rol_usuario NOT NULL,
    telefono VARCHAR(20) DEFAULT NULL,
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    id_supervisor INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    ultimo_acceso TIMESTAMP DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_usuarios_correo UNIQUE (correo_electronico)
);

COMMENT ON COLUMN usuarios.nombre_completo IS 'Nombre completo del usuario';
COMMENT ON COLUMN usuarios.correo_electronico IS 'Email único para inicio de sesión';
COMMENT ON COLUMN usuarios.contrasena IS 'Hash de contraseña (bcrypt)';
COMMENT ON COLUMN usuarios.rol IS 'Rol del usuario en el sistema';
COMMENT ON COLUMN usuarios.telefono IS 'Teléfono de contacto';
COMMENT ON COLUMN usuarios.activo IS 'TRUE = puede iniciar sesión, FALSE = bloqueado';
COMMENT ON COLUMN usuarios.id_supervisor IS 'Referencia al supervisor del usuario (autorreferencial)';
COMMENT ON COLUMN usuarios.ultimo_acceso IS 'Timestamp del último inicio de sesión exitoso';

CREATE INDEX idx_usuarios_rol ON usuarios(rol);
CREATE INDEX idx_usuarios_activo ON usuarios(activo);
CREATE INDEX idx_usuarios_supervisor ON usuarios(id_supervisor);
```

### 3.2 clientes

```sql
CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    tipo_documento tipo_documento DEFAULT NULL,
    numero_documento VARCHAR(20) DEFAULT NULL,
    nombre_completo VARCHAR(255) NOT NULL,
    telefono VARCHAR(20) NOT NULL,
    correo_electronico VARCHAR(255) DEFAULT NULL,
    direccion TEXT DEFAULT NULL,
    ciudad VARCHAR(100) DEFAULT NULL,
    nacionalidad VARCHAR(100) DEFAULT 'Peruana',
    fecha_nacimiento DATE DEFAULT NULL,
    contacto_emergencia VARCHAR(255) DEFAULT NULL,
    telefono_emergencia VARCHAR(20) DEFAULT NULL,
    notas_salud TEXT DEFAULT NULL,
    convertido_de_lead BOOLEAN NOT NULL DEFAULT FALSE,
    canal_origen VARCHAR(50) DEFAULT NULL,
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT uq_clientes_codigo UNIQUE (codigo),
    CONSTRAINT uq_clientes_documento UNIQUE (tipo_documento, numero_documento)
);

COMMENT ON COLUMN clientes.codigo IS 'Código único auto-generado: CLI-{AÑO}-{CORRELATIVO}';
COMMENT ON COLUMN clientes.tipo_documento IS 'DNI, RUC, CE o Pasaporte';
COMMENT ON COLUMN clientes.numero_documento IS 'Número según tipo de documento';
COMMENT ON COLUMN clientes.nombre_completo IS 'Nombres y apellidos del cliente';
COMMENT ON COLUMN clientes.telefono IS 'Teléfono principal de contacto';
COMMENT ON COLUMN clientes.correo_electronico IS 'Correo electrónico del cliente';
COMMENT ON COLUMN clientes.direccion IS 'Dirección de domicilio';
COMMENT ON COLUMN clientes.ciudad IS 'Ciudad de residencia';
COMMENT ON COLUMN clientes.nacionalidad IS 'Nacionalidad del cliente, default Peruana';
COMMENT ON COLUMN clientes.fecha_nacimiento IS 'Fecha de nacimiento para cálculos de edad';
COMMENT ON COLUMN clientes.contacto_emergencia IS 'Nombre del contacto de emergencia';
COMMENT ON COLUMN clientes.telefono_emergencia IS 'Teléfono del contacto de emergencia';
COMMENT ON COLUMN clientes.notas_salud IS 'Alergias, condiciones médicas preexistentes';
COMMENT ON COLUMN clientes.convertido_de_lead IS 'TRUE si el cliente se originó desde un lead';
COMMENT ON COLUMN clientes.canal_origen IS 'Código del canal de origen (C-01, C-02, etc.)';
COMMENT ON COLUMN clientes.creado_por IS 'Usuario que registró al cliente';
COMMENT ON COLUMN clientes.fecha_eliminacion IS 'Soft delete: NULL = activo, timestamp = eliminado';

CREATE INDEX idx_clientes_nombre ON clientes USING btree (lower(nombre_completo) varchar_pattern_ops);
CREATE INDEX idx_clientes_telefono ON clientes(telefono);
CREATE INDEX idx_clientes_fecha_eliminacion ON clientes(fecha_eliminacion);
CREATE INDEX idx_clientes_creado_por ON clientes(creado_por);
```

### 3.3 leads

```sql
CREATE TABLE leads (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    nombre_completo VARCHAR(255) NOT NULL,
    telefono VARCHAR(20) NOT NULL,
    correo_electronico VARCHAR(255) DEFAULT NULL,
    canal_origen VARCHAR(50) DEFAULT NULL,
    ruta_interes TEXT DEFAULT NULL,
    fecha_viaje_estimada DATE DEFAULT NULL,
    num_pasajeros INTEGER NOT NULL DEFAULT 1,
    observacion_inicial TEXT DEFAULT NULL,
    id_cliente INTEGER REFERENCES clientes(id) ON DELETE SET NULL,
    asignado_a INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    estado estado_lead NOT NULL DEFAULT 'Nuevo',
    fecha_conversion TIMESTAMP DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT uq_leads_codigo UNIQUE (codigo),
    CONSTRAINT chk_leads_num_pasajeros CHECK (num_pasajeros >= 1)
);

COMMENT ON COLUMN leads.codigo IS 'Código único: LEAD-{AÑO}-{CORRELATIVO}';
COMMENT ON COLUMN leads.nombre_completo IS 'Nombre del prospecto';
COMMENT ON COLUMN leads.telefono IS 'Teléfono de contacto del lead';
COMMENT ON COLUMN leads.correo_electronico IS 'Correo del lead (opcional)';
COMMENT ON COLUMN leads.canal_origen IS 'Código del canal por donde llegó (C-01...C-07)';
COMMENT ON COLUMN leads.ruta_interes IS 'Destino o paquete turístico de interés';
COMMENT ON COLUMN leads.fecha_viaje_estimada IS 'Fecha aproximada en que desea viajar';
COMMENT ON COLUMN leads.num_pasajeros IS 'Número estimado de pasajeros';
COMMENT ON COLUMN leads.observacion_inicial IS 'Comentario inicial del lead';
COMMENT ON COLUMN leads.id_cliente IS 'Cliente existente vinculado al lead';
COMMENT ON COLUMN leads.asignado_a IS 'Vendedor responsable del lead';
COMMENT ON COLUMN leads.estado IS 'Nuevo, Cotizado, Perdido o Convertido';
COMMENT ON COLUMN leads.fecha_conversion IS 'Timestamp cuando se creó la primera cotización';

CREATE INDEX idx_leads_asignado ON leads(asignado_a);
CREATE INDEX idx_leads_estado ON leads(estado);
CREATE INDEX idx_leads_fecha_creacion ON leads(fecha_creacion);
CREATE INDEX idx_leads_fecha_eliminacion ON leads(fecha_eliminacion);
CREATE INDEX idx_leads_id_cliente ON leads(id_cliente);
```

### 3.4 cotizaciones

```sql
CREATE TABLE cotizaciones (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    id_lead INTEGER REFERENCES leads(id) ON DELETE SET NULL,
    id_cliente INTEGER REFERENCES clientes(id) ON DELETE SET NULL,
    id_plantilla INTEGER REFERENCES plantillas(id) ON DELETE SET NULL,
    id_ruta INTEGER REFERENCES rutas(id) ON DELETE SET NULL,
    num_pasajeros INTEGER NOT NULL,
    base_movilidad NUMERIC(10,2) NOT NULL,
    base_guia NUMERIC(10,2) NOT NULL,
    base_entradas NUMERIC(10,2) NOT NULL,
    base_alimentacion NUMERIC(10,2) NOT NULL,
    base_extras NUMERIC(10,2) NOT NULL,
    base_total NUMERIC(10,2) NOT NULL,
    porcentaje_margen NUMERIC(5,2) NOT NULL,
    monto_margen NUMERIC(10,2) NOT NULL,
    porcentaje_descuento NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    monto_descuento NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    descuento_aprobado_por INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    total NUMERIC(10,2) NOT NULL,
    precio_por_persona NUMERIC(10,2) NOT NULL,
    dias_validez INTEGER NOT NULL,
    valido_hasta DATE NOT NULL,
    estado estado_cotizacion NOT NULL DEFAULT 'Propuesta',
    fecha_envio TIMESTAMP DEFAULT NULL,
    enviado_via VARCHAR(50) DEFAULT NULL,
    visto_por_cliente BOOLEAN NOT NULL DEFAULT FALSE,
    id_motivo_perdida INTEGER REFERENCES motivos_perdida(id) ON DELETE SET NULL,
    observacion_perdida TEXT DEFAULT NULL,
    fecha_perdida TIMESTAMP DEFAULT NULL,
    id_reserva INTEGER REFERENCES reservas(id) ON DELETE SET NULL,
    asignado_a INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    notas_internas TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT uq_cotizaciones_codigo UNIQUE (codigo),
    CONSTRAINT chk_cotizaciones_num_pasajeros CHECK (num_pasajeros >= 1),
    CONSTRAINT chk_cotizaciones_base_movilidad CHECK (base_movilidad >= 0),
    CONSTRAINT chk_cotizaciones_base_guia CHECK (base_guia >= 0),
    CONSTRAINT chk_cotizaciones_base_entradas CHECK (base_entradas >= 0),
    CONSTRAINT chk_cotizaciones_base_alimentacion CHECK (base_alimentacion >= 0),
    CONSTRAINT chk_cotizaciones_base_extras CHECK (base_extras >= 0),
    CONSTRAINT chk_cotizaciones_porcentaje_margen CHECK (porcentaje_margen >= 0),
    CONSTRAINT chk_cotizaciones_porcentaje_descuento CHECK (porcentaje_descuento >= 0),
    CONSTRAINT chk_cotizaciones_dias_validez CHECK (dias_validez IN (7, 15, 30))
);

COMMENT ON COLUMN cotizaciones.codigo IS 'Código único: COT-{AÑO}-{CORRELATIVO}';
COMMENT ON COLUMN cotizaciones.id_lead IS 'Lead que originó la cotización (si aplica)';
COMMENT ON COLUMN cotizaciones.id_cliente IS 'Cliente destino de la cotización';
COMMENT ON COLUMN cotizaciones.id_plantilla IS 'Plantilla usada como modelo';
COMMENT ON COLUMN cotizaciones.id_ruta IS 'Ruta turística cotizada';
COMMENT ON COLUMN cotizaciones.num_pasajeros IS 'Número de pasajeros para la cotización';
COMMENT ON COLUMN cotizaciones.base_movilidad IS 'Costo base de movilidad/transporte';
COMMENT ON COLUMN cotizaciones.base_guia IS 'Costo base del guía turístico';
COMMENT ON COLUMN cotizaciones.base_entradas IS 'Costo base de entradas a atractivos';
COMMENT ON COLUMN cotizaciones.base_alimentacion IS 'Costo base de alimentación';
COMMENT ON COLUMN cotizaciones.base_extras IS 'Costo base de extras no categorizados';
COMMENT ON COLUMN cotizaciones.base_total IS 'Suma de todas las bases';
COMMENT ON COLUMN cotizaciones.porcentaje_margen IS 'Porcentaje de margen de ganancia';
COMMENT ON COLUMN cotizaciones.monto_margen IS 'Monto calculado del margen';
COMMENT ON COLUMN cotizaciones.porcentaje_descuento IS 'Porcentaje de descuento aplicado';
COMMENT ON COLUMN cotizaciones.monto_descuento IS 'Monto calculado del descuento';
COMMENT ON COLUMN cotizaciones.descuento_aprobado_por IS 'Usuario que aprobó el descuento (si requiere)';
COMMENT ON COLUMN cotizaciones.total IS 'Total final: base_total + monto_margen - monto_descuento';
COMMENT ON COLUMN cotizaciones.precio_por_persona IS 'Total dividido entre num_pasajeros';
COMMENT ON COLUMN cotizaciones.dias_validez IS 'Días de validez de la oferta: 7, 15 o 30';
COMMENT ON COLUMN cotizaciones.valido_hasta IS 'Fecha de vencimiento de la cotización';
COMMENT ON COLUMN cotizaciones.estado IS 'Estado en la máquina de estados de cotización';
COMMENT ON COLUMN cotizaciones.fecha_envio IS 'Timestamp del envío al cliente';
COMMENT ON COLUMN cotizaciones.enviado_via IS 'Medio de envío: WhatsApp, Email, Manual';
COMMENT ON COLUMN cotizaciones.visto_por_cliente IS 'Indica si el cliente visualizó la cotización';
COMMENT ON COLUMN cotizaciones.id_motivo_perdida IS 'Motivo si la cotización se perdió';
COMMENT ON COLUMN cotizaciones.observacion_perdida IS 'Observación obligatoria si se pierde';
COMMENT ON COLUMN cotizaciones.fecha_perdida IS 'Timestamp cuando se marcó como perdido';
COMMENT ON COLUMN cotizaciones.id_reserva IS 'Reserva generada a partir de esta cotización (0 o 1)';
COMMENT ON COLUMN cotizaciones.asignado_a IS 'Vendedor responsable';
COMMENT ON COLUMN cotizaciones.creado_por IS 'Usuario que creó la cotización';
COMMENT ON COLUMN cotizaciones.notas_internas IS 'Notas de uso interno del vendedor';

CREATE INDEX idx_cotizaciones_asignado ON cotizaciones(asignado_a);
CREATE INDEX idx_cotizaciones_estado ON cotizaciones(estado);
CREATE INDEX idx_cotizaciones_fecha_creacion ON cotizaciones(fecha_creacion);
CREATE INDEX idx_cotizaciones_id_cliente ON cotizaciones(id_cliente);
CREATE INDEX idx_cotizaciones_id_lead ON cotizaciones(id_lead);
CREATE INDEX idx_cotizaciones_fecha_eliminacion ON cotizaciones(fecha_eliminacion);
CREATE INDEX idx_cotizaciones_estado_fecha ON cotizaciones(estado, fecha_creacion);
CREATE INDEX idx_cotizaciones_valido_hasta ON cotizaciones(valido_hasta);
```

### 3.5 cotizacion_detalle

```sql
CREATE TABLE cotizacion_detalle (
    id SERIAL PRIMARY KEY,
    id_cotizacion INTEGER NOT NULL REFERENCES cotizaciones(id) ON DELETE CASCADE,
    descripcion VARCHAR(500) NOT NULL,
    tipo VARCHAR(50) NOT NULL,
    precio_unitario NUMERIC(10,2) NOT NULL,
    cantidad INTEGER NOT NULL,
    subtotal NUMERIC(10,2) NOT NULL,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT chk_detalle_precio CHECK (precio_unitario >= 0),
    CONSTRAINT chk_detalle_cantidad CHECK (cantidad >= 1),
    CONSTRAINT chk_detalle_subtotal CHECK (subtotal >= 0)
);

COMMENT ON COLUMN cotizacion_detalle.id_cotizacion IS 'Cotización padre (se elimina en cascada)';
COMMENT ON COLUMN cotizacion_detalle.descripcion IS 'Descripción del ítem o servicio';
COMMENT ON COLUMN cotizacion_detalle.tipo IS 'Servicio, Extra o Descuento';
COMMENT ON COLUMN cotizacion_detalle.precio_unitario IS 'Precio por unidad';
COMMENT ON COLUMN cotizacion_detalle.cantidad IS 'Cantidad de unidades';
COMMENT ON COLUMN cotizacion_detalle.subtotal IS 'precio_unitario * cantidad';

CREATE INDEX idx_detalle_cotizacion ON cotizacion_detalle(id_cotizacion);
CREATE INDEX idx_detalle_fecha_eliminacion ON cotizacion_detalle(fecha_eliminacion);
```

### 3.6 plantillas

```sql
CREATE TABLE plantillas (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    id_ruta INTEGER REFERENCES rutas(id) ON DELETE SET NULL,
    descripcion TEXT DEFAULT NULL,
    default_movilidad NUMERIC(10,2) NOT NULL,
    default_guia NUMERIC(10,2) NOT NULL,
    default_entradas NUMERIC(10,2) NOT NULL,
    default_alimentacion NUMERIC(10,2) NOT NULL,
    default_extras NUMERIC(10,2) NOT NULL,
    default_margen NUMERIC(5,2) NOT NULL,
    dias_validez INTEGER NOT NULL,
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_plantillas_nombre UNIQUE (nombre),
    CONSTRAINT chk_plantillas_default_movilidad CHECK (default_movilidad >= 0),
    CONSTRAINT chk_plantillas_default_guia CHECK (default_guia >= 0),
    CONSTRAINT chk_plantillas_default_entradas CHECK (default_entradas >= 0),
    CONSTRAINT chk_plantillas_default_alimentacion CHECK (default_alimentacion >= 0),
    CONSTRAINT chk_plantillas_default_extras CHECK (default_extras >= 0),
    CONSTRAINT chk_plantillas_default_margen CHECK (default_margen >= 0),
    CONSTRAINT chk_plantillas_dias_validez CHECK (dias_validez IN (7, 15, 30))
);

COMMENT ON COLUMN plantillas.nombre IS 'Nombre único de la plantilla';
COMMENT ON COLUMN plantillas.id_ruta IS 'Ruta asociada a la plantilla';
COMMENT ON COLUMN plantillas.descripcion IS 'Descripción de la plantilla';
COMMENT ON COLUMN plantillas.default_movilidad IS 'Valor por defecto para movilidad';
COMMENT ON COLUMN plantillas.default_guia IS 'Valor por defecto para guía';
COMMENT ON COLUMN plantillas.default_entradas IS 'Valor por defecto para entradas';
COMMENT ON COLUMN plantillas.default_alimentacion IS 'Valor por defecto para alimentación';
COMMENT ON COLUMN plantillas.default_extras IS 'Valor por defecto para extras';
COMMENT ON COLUMN plantillas.default_margen IS 'Porcentaje de margen por defecto';
COMMENT ON COLUMN plantillas.dias_validez IS 'Días de validez por defecto: 7, 15 o 30';
COMMENT ON COLUMN plantillas.activa IS 'FALSE = oculta en selectores de cotización';

CREATE INDEX idx_plantillas_activa ON plantillas(activa);
CREATE INDEX idx_plantillas_id_ruta ON plantillas(id_ruta);
```

### 3.7 rutas

```sql
CREATE TABLE rutas (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT DEFAULT NULL,
    duracion_horas INTEGER DEFAULT NULL,
    duracion_dias INTEGER DEFAULT NULL,
    incluye TEXT DEFAULT NULL,
    no_incluye TEXT DEFAULT NULL,
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_rutas_nombre UNIQUE (nombre),
    CONSTRAINT chk_rutas_duracion_horas CHECK (duracion_horas IS NULL OR duracion_horas > 0),
    CONSTRAINT chk_rutas_duracion_dias CHECK (duracion_dias IS NULL OR duracion_dias > 0)
);

COMMENT ON COLUMN rutas.nombre IS 'Nombre único de la ruta o paquete turístico';
COMMENT ON COLUMN rutas.descripcion IS 'Descripción detallada del destino';
COMMENT ON COLUMN rutas.duracion_horas IS 'Duración en horas (para tours de 1 día)';
COMMENT ON COLUMN rutas.duracion_dias IS 'Duración en días (para paquetes multi-día)';
COMMENT ON COLUMN rutas.incluye IS 'Lista de servicios incluidos';
COMMENT ON COLUMN rutas.no_incluye IS 'Lista de servicios no incluidos';
COMMENT ON COLUMN rutas.activa IS 'FALSE = ruta descontinuada';

CREATE INDEX idx_rutas_activa ON rutas(activa);
```

### 3.8 seguimientos

```sql
CREATE TABLE seguimientos (
    id SERIAL PRIMARY KEY,
    id_cotizacion INTEGER NOT NULL REFERENCES cotizaciones(id) ON DELETE CASCADE,
    fecha_programada TIMESTAMP NOT NULL,
    fecha_realizada TIMESTAMP DEFAULT NULL,
    medio_contacto VARCHAR(50) DEFAULT NULL,
    resultado TEXT DEFAULT NULL,
    proxima_accion TEXT DEFAULT NULL,
    id_reprogramacion INTEGER REFERENCES seguimientos(id) ON DELETE SET NULL,
    estado estado_seguimiento NOT NULL DEFAULT 'Programado',
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT chk_seguimientos_fecha_programada CHECK (fecha_programada > CURRENT_TIMESTAMP)
);

COMMENT ON COLUMN seguimientos.id_cotizacion IS 'Cotización asociada al seguimiento';
COMMENT ON COLUMN seguimientos.fecha_programada IS 'Fecha y hora programada para el contacto';
COMMENT ON COLUMN seguimientos.fecha_realizada IS 'Timestamp cuando se realizó el contacto';
COMMENT ON COLUMN seguimientos.medio_contacto IS 'Llamada, WhatsApp, Email o Presencial';
COMMENT ON COLUMN seguimientos.resultado IS 'Descripción del resultado del contacto';
COMMENT ON COLUMN seguimientos.proxima_accion IS 'Próxima acción acordada con el cliente';
COMMENT ON COLUMN seguimientos.id_reprogramacion IS 'Seguimiento anterior que fue reprogramado a este';
COMMENT ON COLUMN seguimientos.estado IS 'Programado, Realizado, Vencido o Reprogramado';
COMMENT ON COLUMN seguimientos.creado_por IS 'Usuario que programó el seguimiento';

CREATE INDEX idx_seguimientos_cotizacion ON seguimientos(id_cotizacion);
CREATE INDEX idx_seguimientos_estado ON seguimientos(estado);
CREATE INDEX idx_seguimientos_fecha_programada ON seguimientos(fecha_programada);
CREATE INDEX idx_seguimientos_creado_por ON seguimientos(creado_por);
```

### 3.9 motivos_perdida

```sql
CREATE TABLE motivos_perdida (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(10) NOT NULL,
    nombre VARCHAR(255) NOT NULL,
    requiere_observacion BOOLEAN NOT NULL DEFAULT TRUE,
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    orden INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT uq_motivos_perdida_codigo UNIQUE (codigo),
    CONSTRAINT uq_motivos_perdida_nombre UNIQUE (nombre)
);

COMMENT ON COLUMN motivos_perdida.codigo IS 'Código del motivo: P-01, P-02...';
COMMENT ON COLUMN motivos_perdida.nombre IS 'Nombre descriptivo del motivo';
COMMENT ON COLUMN motivos_perdida.requiere_observacion IS 'TRUE = obliga a escribir observación';
COMMENT ON COLUMN motivos_perdida.activo IS 'FALSE = motivo desactivado';
COMMENT ON COLUMN motivos_perdida.orden IS 'Orden de aparición en selectores';

CREATE UNIQUE INDEX uq_motivos_perdida_orden ON motivos_perdida(orden);
```

### 3.10 reservas

```sql
CREATE TABLE reservas (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    id_cotizacion INTEGER NOT NULL REFERENCES cotizaciones(id) ON DELETE SET NULL,
    id_cliente INTEGER NOT NULL REFERENCES clientes(id) ON DELETE SET NULL,
    fecha_viaje DATE NOT NULL,
    hora_salida TIME DEFAULT NULL,
    lugar_recojo VARCHAR(255) DEFAULT NULL,
    fecha_retorno DATE DEFAULT NULL,
    num_pasajeros INTEGER NOT NULL,
    monto_total NUMERIC(10,2) NOT NULL,
    total_pagado NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    saldo NUMERIC(10,2) NOT NULL,
    estado estado_reserva NOT NULL DEFAULT 'Borrador',
    movilidad_asignada VARCHAR(255) DEFAULT NULL,
    guia_asignado VARCHAR(255) DEFAULT NULL,
    conductor_asignado VARCHAR(255) DEFAULT NULL,
    notas_operativas TEXT DEFAULT NULL,
    motivo_anulacion TEXT DEFAULT NULL,
    fecha_anulacion TIMESTAMP DEFAULT NULL,
    anulado_por INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT uq_reservas_codigo UNIQUE (codigo),
    CONSTRAINT uq_reservas_cotizacion UNIQUE (id_cotizacion),
    CONSTRAINT chk_reservas_num_pasajeros CHECK (num_pasajeros >= 1),
    CONSTRAINT chk_reservas_monto_total CHECK (monto_total >= 0),
    CONSTRAINT chk_reservas_total_pagado CHECK (total_pagado >= 0),
    CONSTRAINT chk_reservas_saldo CHECK (saldo >= 0),
    CONSTRAINT chk_reservas_fecha_viaje CHECK (fecha_viaje > CURRENT_DATE),
    CONSTRAINT chk_reservas_fecha_retorno CHECK (fecha_retorno IS NULL OR fecha_retorno >= fecha_viaje)
);

COMMENT ON COLUMN reservas.codigo IS 'Código único: ADV-{AÑO}-{CORRELATIVO}';
COMMENT ON COLUMN reservas.id_cotizacion IS 'Cotización que originó la reserva (relación 1:1)';
COMMENT ON COLUMN reservas.id_cliente IS 'Cliente titular de la reserva';
COMMENT ON COLUMN reservas.fecha_viaje IS 'Fecha de inicio del viaje';
COMMENT ON COLUMN reservas.hora_salida IS 'Hora de salida del recojo';
COMMENT ON COLUMN reservas.lugar_recojo IS 'Dirección o punto de recojo';
COMMENT ON COLUMN reservas.fecha_retorno IS 'Fecha de retorno (para viajes multi-día)';
COMMENT ON COLUMN reservas.num_pasajeros IS 'Número total de pasajeros';
COMMENT ON COLUMN reservas.monto_total IS 'Monto total de la reserva (desde cotización)';
COMMENT ON COLUMN reservas.total_pagado IS 'Suma de pagos vigentes recibidos';
COMMENT ON COLUMN reservas.saldo IS 'monto_total - total_pagado';
COMMENT ON COLUMN reservas.estado IS 'Borrador, Pendiente, Confirmada, Finalizada, Anulada';
COMMENT ON COLUMN reservas.movilidad_asignada IS 'Descripción del vehículo asignado';
COMMENT ON COLUMN reservas.guia_asignado IS 'Nombre del guía asignado';
COMMENT ON COLUMN reservas.conductor_asignado IS 'Nombre del conductor asignado';
COMMENT ON COLUMN reservas.notas_operativas IS 'Instrucciones para el equipo de operaciones';
COMMENT ON COLUMN reservas.motivo_anulacion IS 'Motivo obligatorio si la reserva se anula';
COMMENT ON COLUMN reservas.fecha_anulacion IS 'Timestamp de anulación';
COMMENT ON COLUMN reservas.anulado_por IS 'Usuario que anuló la reserva';
COMMENT ON COLUMN reservas.creado_por IS 'Usuario que creó la reserva';

CREATE INDEX idx_reservas_cliente ON reservas(id_cliente);
CREATE INDEX idx_reservas_estado ON reservas(estado);
CREATE INDEX idx_reservas_fecha_viaje ON reservas(fecha_viaje);
CREATE INDEX idx_reservas_fecha_creacion ON reservas(fecha_creacion);
CREATE INDEX idx_reservas_creado_por ON reservas(creado_por);
CREATE INDEX idx_reservas_fecha_eliminacion ON reservas(fecha_eliminacion);
CREATE INDEX idx_reservas_estado_fecha ON reservas(estado, fecha_viaje);
```

### 3.11 pasajeros

```sql
CREATE TABLE pasajeros (
    id SERIAL PRIMARY KEY,
    id_reserva INTEGER NOT NULL REFERENCES reservas(id) ON DELETE CASCADE,
    nombre_completo VARCHAR(255) NOT NULL,
    tipo_documento tipo_documento DEFAULT NULL,
    numero_documento VARCHAR(20) DEFAULT NULL,
    fecha_nacimiento DATE DEFAULT NULL,
    edad INTEGER DEFAULT NULL,
    telefono VARCHAR(20) DEFAULT NULL,
    correo_electronico VARCHAR(255) DEFAULT NULL,
    contacto_emergencia VARCHAR(255) NOT NULL,
    telefono_emergencia VARCHAR(20) NOT NULL,
    notas_salud TEXT DEFAULT NULL,
    requiere_seguro BOOLEAN NOT NULL DEFAULT FALSE,
    es_titular BOOLEAN NOT NULL DEFAULT FALSE,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL
);

COMMENT ON COLUMN pasajeros.id_reserva IS 'Reserva a la que pertenece el pasajero';
COMMENT ON COLUMN pasajeros.nombre_completo IS 'Nombre completo del pasajero';
COMMENT ON COLUMN pasajeros.tipo_documento IS 'Tipo de documento de identidad';
COMMENT ON COLUMN pasajeros.numero_documento IS 'Número de documento';
COMMENT ON COLUMN pasajeros.fecha_nacimiento IS 'Fecha de nacimiento';
COMMENT ON COLUMN pasajeros.edad IS 'Edad calculada al momento del registro';
COMMENT ON COLUMN pasajeros.telefono IS 'Teléfono del pasajero';
COMMENT ON COLUMN pasajeros.correo_electronico IS 'Correo del pasajero';
COMMENT ON COLUMN pasajeros.contacto_emergencia IS 'Nombre del contacto de emergencia';
COMMENT ON COLUMN pasajeros.telefono_emergencia IS 'Teléfono del contacto de emergencia';
COMMENT ON COLUMN pasajeros.notas_salud IS 'Alergias o condiciones médicas';
COMMENT ON COLUMN pasajeros.requiere_seguro IS 'TRUE si necesita seguro de viajero';
COMMENT ON COLUMN pasajeros.es_titular IS 'Exactamente 1 por reserva debe ser TRUE';

CREATE INDEX idx_pasajeros_reserva ON pasajeros(id_reserva);
CREATE INDEX idx_pasajeros_fecha_eliminacion ON pasajeros(fecha_eliminacion);
```

### 3.12 pagos

```sql
CREATE TABLE pagos (
    id SERIAL PRIMARY KEY,
    id_reserva INTEGER NOT NULL REFERENCES reservas(id) ON DELETE CASCADE,
    monto NUMERIC(10,2) NOT NULL,
    metodo_pago metodo_pago NOT NULL,
    referencia VARCHAR(255) DEFAULT NULL,
    fecha_pago TIMESTAMP NOT NULL,
    recibido_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    ruta_comprobante VARCHAR(500) DEFAULT NULL,
    notas TEXT DEFAULT NULL,
    es_adelanto BOOLEAN NOT NULL DEFAULT FALSE,
    fecha_anulacion TIMESTAMP DEFAULT NULL,
    anulado_por INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    motivo_anulacion TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_pagos_monto CHECK (monto > 0),
    CONSTRAINT chk_pagos_fecha_pago CHECK (fecha_pago <= CURRENT_TIMESTAMP)
);

COMMENT ON COLUMN pagos.id_reserva IS 'Reserva a la que se aplica el pago';
COMMENT ON COLUMN pagos.monto IS 'Monto del pago (debe ser > 0)';
COMMENT ON COLUMN pagos.metodo_pago IS 'Efectivo, Yape, Plin, Transferencia, Tarjeta crédito, Tarjeta débito, Depósito';
COMMENT ON COLUMN pagos.referencia IS 'Número de operación o voucher';
COMMENT ON COLUMN pagos.fecha_pago IS 'Fecha y hora en que se realizó el pago';
COMMENT ON COLUMN pagos.recibido_por IS 'Usuario que registró el pago';
COMMENT ON COLUMN pagos.ruta_comprobante IS 'Ruta al archivo del comprobante (PDF/JPG/PNG)';
COMMENT ON COLUMN pagos.notas IS 'Observaciones del pago';
COMMENT ON COLUMN pagos.es_adelanto IS 'TRUE si es un pago parcial';
COMMENT ON COLUMN pagos.fecha_anulacion IS 'Timestamp de anulación del pago';
COMMENT ON COLUMN pagos.anulado_por IS 'Usuario que anuló el pago';
COMMENT ON COLUMN pagos.motivo_anulacion IS 'Motivo obligatorio de la anulación';

CREATE INDEX idx_pagos_reserva ON pagos(id_reserva);
CREATE INDEX idx_pagos_metodo ON pagos(metodo_pago);
CREATE INDEX idx_pagos_fecha ON pagos(fecha_pago);
CREATE INDEX idx_pagos_recibido_por ON pagos(recibido_por);
```

### 3.13 politicas_configuracion

```sql
CREATE TABLE politicas_configuracion (
    id SERIAL PRIMARY KEY,
    titulo VARCHAR(255) NOT NULL,
    contenido TEXT NOT NULL,
    seccion VARCHAR(50) NOT NULL,
    orden INTEGER NOT NULL DEFAULT 0,
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    creado_por INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE SET NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_politicas_seccion CHECK (
        seccion IN ('Cancelacion', 'Pago', 'Responsabilidad', 'General')
    )
);

COMMENT ON COLUMN politicas_configuracion.titulo IS 'Título de la política';
COMMENT ON COLUMN politicas_configuracion.contenido IS 'Texto completo de la política';
COMMENT ON COLUMN politicas_configuracion.seccion IS 'Cancelacion, Pago, Responsabilidad o General';
COMMENT ON COLUMN politicas_configuracion.orden IS 'Orden de aparición en el PDF';
COMMENT ON COLUMN politicas_configuracion.activa IS 'TRUE = visible en PDF y sistema';
COMMENT ON COLUMN politicas_configuracion.creado_por IS 'Usuario que creó/modificó la política';

CREATE INDEX idx_politicas_seccion ON politicas_configuracion(seccion);
CREATE INDEX idx_politicas_activa ON politicas_configuracion(activa);
```

### 3.14 auditoria

```sql
CREATE TABLE auditoria (
    id SERIAL PRIMARY KEY,
    id_usuario INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    accion VARCHAR(50) NOT NULL,
    tipo_entidad VARCHAR(100) NOT NULL,
    id_entidad INTEGER NOT NULL,
    valores_anteriores TEXT DEFAULT NULL,
    valores_nuevos TEXT DEFAULT NULL,
    direccion_ip VARCHAR(45) DEFAULT NULL,
    agente_usuario TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON COLUMN auditoria.id_usuario IS 'Usuario que ejecutó la acción (NULL = sistema)';
COMMENT ON COLUMN auditoria.accion IS 'Creación, Modificación, Envío, Aprobación, Anulación, Pago, Conversión, Reactivación';
COMMENT ON COLUMN auditoria.tipo_entidad IS 'Nombre de la entidad afectada';
COMMENT ON COLUMN auditoria.id_entidad IS 'ID del registro afectado';
COMMENT ON COLUMN auditoria.valores_anteriores IS 'Estado anterior: campo=valor|campo=valor...';
COMMENT ON COLUMN auditoria.valores_nuevos IS 'Estado nuevo: campo=valor|campo=valor...';
COMMENT ON COLUMN auditoria.direccion_ip IS 'Dirección IP desde donde se ejecutó la acción';
COMMENT ON COLUMN auditoria.agente_usuario IS 'User-Agent del navegador/cliente';
COMMENT ON COLUMN auditoria.fecha_creacion IS 'Timestamp inmutable de creación';

CREATE INDEX idx_auditoria_entidad ON auditoria(tipo_entidad, id_entidad);
CREATE INDEX idx_auditoria_usuario ON auditoria(id_usuario);
CREATE INDEX idx_auditoria_accion ON auditoria(accion);
CREATE INDEX idx_auditoria_fecha ON auditoria(fecha_creacion);
```

### 3.15 canales_origen

```sql
CREATE TABLE canales_origen (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(10) NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    CONSTRAINT uq_canales_origen_codigo UNIQUE (codigo),
    CONSTRAINT uq_canales_origen_nombre UNIQUE (nombre)
);

COMMENT ON COLUMN canales_origen.codigo IS 'Código del canal: C-01, C-02...';
COMMENT ON COLUMN canales_origen.nombre IS 'Nombre descriptivo del canal';
COMMENT ON COLUMN canales_origen.activo IS 'TRUE = disponible en selectores';

CREATE INDEX idx_canales_activo ON canales_origen(activo);
```

### 3.16 secuencias_codigos

```sql
CREATE TABLE secuencias_codigos (
    id SERIAL PRIMARY KEY,
    prefijo VARCHAR(10) NOT NULL,
    anio INTEGER NOT NULL,
    ultimo_numero INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT uq_secuencias_prefijo_anio UNIQUE (prefijo, anio),
    CONSTRAINT chk_secuencias_ultimo_numero CHECK (ultimo_numero >= 0)
);

COMMENT ON COLUMN secuencias_codigos.prefijo IS 'Prefijo: COT, ADV, CLI, LEAD';
COMMENT ON COLUMN secuencias_codigos.anio IS 'Año de la secuencia';
COMMENT ON COLUMN secuencias_codigos.ultimo_numero IS 'Último número correlativo usado';

CREATE INDEX idx_secuencias_prefijo ON secuencias_codigos(prefijo, anio);
```

### 3.17 alojamientos

```sql
CREATE TABLE alojamientos (
    id SERIAL PRIMARY KEY,
    id_reserva INTEGER NOT NULL REFERENCES reservas(id) ON DELETE CASCADE,
    noche_numero INTEGER NOT NULL,
    hotel_nombre VARCHAR(255) NOT NULL,
    tipo_habitacion tipo_habitacion NOT NULL,
    tipo_cama tipo_cama DEFAULT NULL,
    plan_comidas plan_comidas DEFAULT NULL,
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    notas_alojamiento TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT chk_alojamientos_noche_numero CHECK (noche_numero >= 1),
    CONSTRAINT chk_alojamientos_check_out CHECK (check_out > check_in)
);

COMMENT ON COLUMN alojamientos.id_reserva IS 'Reserva a la que pertenece el alojamiento';
COMMENT ON COLUMN alojamientos.noche_numero IS 'Número de noche (1, 2, 3...)';
COMMENT ON COLUMN alojamientos.hotel_nombre IS 'Nombre del hotel o alojamiento';
COMMENT ON COLUMN alojamientos.tipo_habitacion IS 'Individual, Doble, Triple, Compartida, Suite';
COMMENT ON COLUMN alojamientos.tipo_cama IS 'Queen, King, Dos camas individuales, Litera, Sofá cama';
COMMENT ON COLUMN alojamientos.plan_comidas IS 'Solo desayuno, Media pensión, Pensión completa, Todo incluido';
COMMENT ON COLUMN alojamientos.check_in IS 'Fecha de ingreso';
COMMENT ON COLUMN alojamientos.check_out IS 'Fecha de salida (debe ser mayor a check_in)';
COMMENT ON COLUMN alojamientos.notas_alojamiento IS 'Preferencias o solicitudes especiales';

CREATE INDEX idx_alojamientos_reserva ON alojamientos(id_reserva);
CREATE INDEX idx_alojamientos_fecha_eliminacion ON alojamientos(fecha_eliminacion);
```

### 3.18 servicios_reserva

```sql
CREATE TABLE servicios_reserva (
    id SERIAL PRIMARY KEY,
    id_reserva INTEGER NOT NULL REFERENCES reservas(id) ON DELETE CASCADE,
    descripcion VARCHAR(500) NOT NULL,
    tipo_servicio tipo_servicio NOT NULL,
    proveedor VARCHAR(255) DEFAULT NULL,
    precio_unitario NUMERIC(10,2) NOT NULL,
    cantidad INTEGER NOT NULL,
    subtotal NUMERIC(10,2) NOT NULL,
    fecha_servicio DATE DEFAULT NULL,
    notas TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT chk_servicios_precio CHECK (precio_unitario >= 0),
    CONSTRAINT chk_servicios_cantidad CHECK (cantidad >= 1)
);

COMMENT ON COLUMN servicios_reserva.id_reserva IS 'Reserva a la que pertenece el servicio';
COMMENT ON COLUMN servicios_reserva.descripcion IS 'Descripción del servicio adicional';
COMMENT ON COLUMN servicios_reserva.tipo_servicio IS 'Seguro viajero, Tour adicional, Traslado extra, Cena especial, Equipo adicional, Otro';
COMMENT ON COLUMN servicios_reserva.proveedor IS 'Nombre del proveedor del servicio';
COMMENT ON COLUMN servicios_reserva.precio_unitario IS 'Precio por unidad del servicio';
COMMENT ON COLUMN servicios_reserva.cantidad IS 'Cantidad de unidades contratadas';
COMMENT ON COLUMN servicios_reserva.subtotal IS 'precio_unitario * cantidad';
COMMENT ON COLUMN servicios_reserva.fecha_servicio IS 'Fecha en que se presta el servicio';
COMMENT ON COLUMN servicios_reserva.notas IS 'Observaciones del servicio';

CREATE INDEX idx_servicios_reserva ON servicios_reserva(id_reserva);
CREATE INDEX idx_servicios_fecha_eliminacion ON servicios_reserva(fecha_eliminacion);
```

### 3.19 roles

```sql
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    descripcion TEXT DEFAULT NULL,
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    CONSTRAINT uq_roles_codigo UNIQUE (codigo),
    CONSTRAINT uq_roles_nombre UNIQUE (nombre)
);

COMMENT ON COLUMN roles.codigo IS 'Código interno del rol: ADMIN, VENDEDOR, SUPERVISOR...';
COMMENT ON COLUMN roles.nombre IS 'Nombre legible del rol';
COMMENT ON COLUMN roles.descripcion IS 'Descripción de las responsabilidades del rol';
COMMENT ON COLUMN roles.activo IS 'TRUE = rol disponible para asignar';
```

### 3.20 permisos

```sql
CREATE TABLE permisos (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(50) NOT NULL,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT DEFAULT NULL,
    CONSTRAINT uq_permisos_codigo UNIQUE (codigo)
);

COMMENT ON COLUMN permisos.codigo IS 'Código del permiso: cotizaciones.crear, reservas.anular...';
COMMENT ON COLUMN permisos.nombre IS 'Nombre descriptivo del permiso';
COMMENT ON COLUMN permisos.descripcion IS 'Descripción de lo que otorga el permiso';
```

### 3.21 rol_permiso (tabla pivote)

```sql
CREATE TABLE rol_permiso (
    id SERIAL PRIMARY KEY,
    id_rol INTEGER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    id_permiso INTEGER NOT NULL REFERENCES permisos(id) ON DELETE CASCADE,
    CONSTRAINT uq_rol_permiso UNIQUE (id_rol, id_permiso)
);

COMMENT ON COLUMN rol_permiso.id_rol IS 'Referencia al rol';
COMMENT ON COLUMN rol_permiso.id_permiso IS 'Referencia al permiso';

CREATE INDEX idx_rol_permiso_rol ON rol_permiso(id_rol);
CREATE INDEX idx_rol_permiso_permiso ON rol_permiso(id_permiso);
```

### 3.22 agencias (multi-tenancy futuro)

```sql
CREATE TABLE agencias (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL,
    nombre VARCHAR(255) NOT NULL,
    ruc VARCHAR(11) NOT NULL,
    direccion TEXT DEFAULT NULL,
    telefono VARCHAR(20) DEFAULT NULL,
    correo_electronico VARCHAR(255) DEFAULT NULL,
    logo_url VARCHAR(500) DEFAULT NULL,
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_eliminacion TIMESTAMP DEFAULT NULL,
    CONSTRAINT uq_agencias_codigo UNIQUE (codigo),
    CONSTRAINT uq_agencias_ruc UNIQUE (ruc)
);

COMMENT ON COLUMN agencias.codigo IS 'Código único de la agencia';
COMMENT ON COLUMN agencias.nombre IS 'Nombre comercial de la agencia';
COMMENT ON COLUMN agencias.ruc IS 'RUC de la agencia (11 dígitos)';
COMMENT ON COLUMN agencias.direccion IS 'Dirección fiscal de la agencia';
COMMENT ON COLUMN agencias.telefono IS 'Teléfono de contacto';
COMMENT ON COLUMN agencias.correo_electronico IS 'Correo corporativo';
COMMENT ON COLUMN agencias.logo_url IS 'URL del logo para documentos PDF';
COMMENT ON COLUMN agencias.activa IS 'TRUE = agencia operativa';

CREATE INDEX idx_agencias_activa ON agencias(activa);
```

---

## 4. Seeders — Datos Iniciales

### 4.1 Roles

```sql
INSERT INTO roles (codigo, nombre, descripcion) VALUES
    ('ADMIN', 'Administrador', 'Control total del sistema. Gestión de usuarios, configuración y datos maestros.'),
    ('VENDEDOR', 'Vendedor', 'Gestión comercial directa: leads, cotizaciones, seguimientos, conversión a reservas.'),
    ('SUPERVISOR', 'Supervisor', 'Supervisión de equipo comercial. Aprueba descuentos y visualiza métricas del equipo.'),
    ('OPERACIONES', 'Operaciones', 'Gestión operativa de reservas confirmadas: movilidad, guías, conductores.'),
    ('CONTABILIDAD', 'Contabilidad', 'Registro y verificación de pagos. Control de caja.'),
    ('GERENCIA', 'Gerencia', 'Acceso de solo lectura a reportes, KPIs y dashboards.');
```

### 4.2 Canales de Origen

```sql
INSERT INTO canales_origen (codigo, nombre, activo) VALUES
    ('C-01', 'WhatsApp', TRUE),
    ('C-02', 'Facebook', TRUE),
    ('C-03', 'Instagram', TRUE),
    ('C-04', 'Referido', TRUE),
    ('C-05', 'Web', TRUE),
    ('C-06', 'Teléfono', TRUE),
    ('C-07', 'Presencial', TRUE);
```

### 4.3 Motivos de Pérdida

```sql
INSERT INTO motivos_perdida (codigo, nombre, requiere_observacion, activo, orden) VALUES
    ('P-01', 'Precio alto', TRUE, TRUE, 1),
    ('P-02', 'No respondió', TRUE, TRUE, 2),
    ('P-03', 'Encontró otra agencia', TRUE, TRUE, 3),
    ('P-04', 'Cambió de planes', TRUE, TRUE, 4),
    ('P-05', 'Problemas económicos', TRUE, TRUE, 5),
    ('P-06', 'No le interesó', TRUE, TRUE, 6),
    ('P-07', 'Otro', TRUE, TRUE, 7);
```

### 4.4 Políticas de Configuración

```sql
INSERT INTO politicas_configuracion (titulo, contenido, seccion, orden, activa) VALUES
    (
        'Política de Cancelación',
        'El cliente podrá cancelar su reserva hasta 48 horas antes de la fecha del viaje sin penalidad. '
        'Cancelaciones dentro de las 48 horas previas al viaje tendrán una penalidad del 50% del monto total. '
        'Cancelaciones el mismo día del viaje no tienen derecho a reembolso. '
        'Todo proceso de cancelación debe ser comunicado por escrito vía WhatsApp o correo electrónico.',
        'Cancelacion',
        1,
        TRUE
    ),
    (
        'Política de Pagos',
        'Para confirmar una reserva se requiere un adelanto mínimo del 50% del monto total. '
        'El saldo restante debe ser cancelado hasta 24 horas antes de la fecha del viaje. '
        'Los pagos pueden realizarse en efectivo, Yape, Plin, transferencia bancaria o tarjeta. '
        'Todo pago debe ser registrado en el sistema y se entregará un comprobante.',
        'Pago',
        2,
        TRUE
    ),
    (
        'Política de Responsabilidad',
        'Adventur Travel no se hace responsable por objetos personales perdidos durante el viaje. '
        'El cliente es responsable de contar con la documentación necesaria para el viaje (DNI, pasaporte, etc.). '
        'Adventur Travel se reserva el derecho de modificar el itinerario por causas de fuerza mayor '
        '(clima, desastres naturales, cierre de vías, etc.), comprometiéndose a ofrecer alternativas equivalentes. '
        'El cliente declara estar en condiciones físicas adecuadas para realizar las actividades del paquete contratado.',
        'Responsabilidad',
        3,
        TRUE
    ),
    (
        'Políticas Generales',
        'El horario de recojo será confirmado vía WhatsApp un día antes del viaje. '
        'Se recomienda llegar 10 minutos antes del horario acordado. '
        'Los menores de edad deben viajar acompañados de un adulto responsable. '
        'Las tarifas no incluyen gastos personales, propinas ni actividades no especificadas en el itinerario. '
        'Cualquier modificación a la reserva debe ser solicitada con al menos 24 horas de anticipación.',
        'General',
        4,
        TRUE
    );
```

### 4.5 Usuario Administrador Inicial

```sql
-- Contraseña por defecto: admin123 (debe cambiarse en primer inicio de sesión)
INSERT INTO usuarios (nombre_completo, correo_electronico, contrasena, rol, activo)
VALUES (
    'Administrador del Sistema',
    'admin@adventurtravel.pe',
    '$2y$12$LJ3m4ys3Lk0TSwHnbfOMiO7Stx7s.VJUl3JMfm8QIXBxUl.7q.6Sy',
    'Administrador',
    TRUE
);
```

---

## 5. Índices Adicionales Recomendados

### 5.1 Índices para Búsqueda de Clientes

```sql
CREATE INDEX idx_clientes_busqueda_texto ON clientes
    USING gin (to_tsvector('spanish', nombre_completo || ' ' || COALESCE(numero_documento, '')));

CREATE INDEX idx_clientes_codigo_trgm ON clientes
    USING gin (codigo gin_trgm_ops);
```

### 5.2 Índices para Reportes Gerenciales

```sql
-- Reporte de cotizaciones por mes y vendedor
CREATE INDEX idx_cotizaciones_reporte_mensual ON cotizaciones
    (EXTRACT(YEAR FROM fecha_creacion), EXTRACT(MONTH FROM fecha_creacion), asignado_a, estado);

-- Reporte de reservas por mes
CREATE INDEX idx_reservas_reporte_mensual ON reservas
    (EXTRACT(YEAR FROM fecha_creacion), EXTRACT(MONTH FROM fecha_creacion), estado);

-- Reporte de pagos por mes y método
CREATE INDEX idx_pagos_reporte_mensual ON pagos
    (EXTRACT(YEAR FROM fecha_pago), EXTRACT(MONTH FROM fecha_pago), metodo_pago);

-- KPIs de conversión
CREATE INDEX idx_leads_conversion ON leads
    (asignado_a, estado, fecha_creacion);
```

### 5.3 Índices para Evitar Duplicados en Reportes

```sql
CREATE UNIQUE INDEX uq_cotizacion_detalle_unique ON cotizacion_detalle
    (id_cotizacion, descripcion, tipo) WHERE fecha_eliminacion IS NULL;

CREATE UNIQUE INDEX uq_pasajero_titular ON pasajeros
    (id_reserva) WHERE es_titular = TRUE;
```

---

## 6. Particionamiento (Fase 2)

### 6.1 Auditoría por Año

```sql
-- Crear tabla particionada (reemplazar definición original)
CREATE TABLE auditoria (
    id SERIAL,
    id_usuario INTEGER REFERENCES usuarios(id) ON DELETE SET NULL,
    accion VARCHAR(50) NOT NULL,
    tipo_entidad VARCHAR(100) NOT NULL,
    id_entidad INTEGER NOT NULL,
    valores_anteriores TEXT DEFAULT NULL,
    valores_nuevos TEXT DEFAULT NULL,
    direccion_ip VARCHAR(45) DEFAULT NULL,
    agente_usuario TEXT DEFAULT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, fecha_creacion)
) PARTITION BY RANGE (fecha_creacion);

-- Crear particiones por año
CREATE TABLE auditoria_2025 PARTITION OF auditoria
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE auditoria_2026 PARTITION OF auditoria
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE TABLE auditoria_2027 PARTITION OF auditoria
    FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');

CREATE TABLE auditoria_default PARTITION OF auditoria DEFAULT;

-- Índices locales en cada partición (se crean automáticamente sobre la tabla padre
-- y se heredan a las particiones con PostgreSQL 16+)
CREATE INDEX idx_auditoria_entidad_particionado ON auditoria(tipo_entidad, id_entidad);
CREATE INDEX idx_auditoria_fecha_particionado ON auditoria(fecha_creacion);
```

### 6.2 Cotizaciones por Año (Opcional)

```sql
CREATE TABLE cotizaciones (
    id SERIAL,
    codigo VARCHAR(20) NOT NULL,
    -- ... mismos campos ...
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, fecha_creacion)
) PARTITION BY RANGE (fecha_creacion);

CREATE TABLE cotizaciones_2025 PARTITION OF cotizaciones
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE cotizaciones_2026 PARTITION OF cotizaciones
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE TABLE cotizaciones_default PARTITION OF cotizaciones DEFAULT;
```

---

## 7. Row Level Security (RLS) — Soft Delete (Opcional / Avanzado)

### 7.1 Política de Filtro por Defecto

```sql
-- Habilitar RLS en tablas con soft delete
ALTER TABLE clientes ENABLE ROW LEVEL SECURITY;
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE cotizaciones ENABLE ROW LEVEL SECURITY;
ALTER TABLE cotizacion_detalle ENABLE ROW LEVEL SECURITY;
ALTER TABLE reservas ENABLE ROW LEVEL SECURITY;
ALTER TABLE pasajeros ENABLE ROW LEVEL SECURITY;
ALTER TABLE pagos ENABLE ROW LEVEL SECURITY;
ALTER TABLE alojamientos ENABLE ROW LEVEL SECURITY;
ALTER TABLE servicios_reserva ENABLE ROW LEVEL SECURITY;

-- Política: ocultar registros con fecha_eliminacion NOT NULL
CREATE POLICY rls_soft_delete_clientes ON clientes
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_leads ON leads
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_cotizaciones ON cotizaciones
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_cotizacion_detalle ON cotizacion_detalle
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_reservas ON reservas
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_pasajeros ON pasajeros
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_pagos ON pagos
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_alojamientos ON alojamientos
    USING (fecha_eliminacion IS NULL);

CREATE POLICY rls_soft_delete_servicios_reserva ON servicios_reserva
    USING (fecha_eliminacion IS NULL);
```

### 7.2 Política para Administrador (Ver Todo)

```sql
-- El administrador puede ver todos los registros, incluso eliminados
-- (se asigna a un rol específico de BD o se maneja a nivel aplicación)
CREATE POLICY rls_admin_ver_todo_clientes ON clientes
    FOR SELECT
    USING (current_user = 'admin_adventur' OR fecha_eliminacion IS NULL);
```

### 7.3 Nota sobre RLS en Producción

> El uso de RLS para soft delete es **opcional**. En la mayoría de casos, el filtro se maneja a nivel de aplicación (Laravel Query Scopes). RLS se recomienda solo cuando se requiere garantizar a nivel de base de datos que ningún query (incluso desde consola SQL) pueda leer registros eliminados sin autorización explícita. Para el MVP se recomienda manejar el filtro en la capa de aplicación.

---

## 8. Resumen de Convenciones

| Aspecto | Convención |
|---------|-----------|
| IDs | `SERIAL PRIMARY KEY` |
| FK | `campo INTEGER REFERENCES tabla(id) ON DELETE {CASCADE/SET NULL}` |
| Timestamps creación | `fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| Timestamps actualización | `fecha_actualizacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` |
| Soft delete | `fecha_eliminacion TIMESTAMP DEFAULT NULL` |
| Booleanos | `BOOLEAN NOT NULL DEFAULT FALSE/TRUE` |
| Monetario | `NUMERIC(10,2)` — siempre TRUNC(2) |
| Porcentajes | `NUMERIC(5,2)` |
| Códigos únicos | `VARCHAR(20) UNIQUE NOT NULL` |
| Texto largo | `TEXT` |
| CHECK constraints | Prefijo `chk_` + nombre_tabla + campo |
| UNIQUE constraints | Prefijo `uq_` + nombre_tabla + campo(s) |
| Índices | Prefijo `idx_` + nombre_tabla + campo(s) |
| Comentarios | `COMMENT ON COLUMN` en español |
| ENUMs | Tipo personalizado PostgreSQL con `CREATE TYPE` |

---

## 9. Mapa de Dependencias (FK Resumen)

| Tabla | FK | Referencia | ON DELETE |
|-------|----|-----------|-----------|
| usuarios | id_supervisor | usuarios(id) | SET NULL |
| clientes | creado_por | usuarios(id) | SET NULL |
| leads | asignado_a | usuarios(id) | SET NULL |
| leads | id_cliente | clientes(id) | SET NULL |
| cotizaciones | id_lead | leads(id) | SET NULL |
| cotizaciones | id_cliente | clientes(id) | SET NULL |
| cotizaciones | id_plantilla | plantillas(id) | SET NULL |
| cotizaciones | id_ruta | rutas(id) | SET NULL |
| cotizaciones | descuento_aprobado_por | usuarios(id) | SET NULL |
| cotizaciones | id_motivo_perdida | motivos_perdida(id) | SET NULL |
| cotizaciones | id_reserva | reservas(id) | SET NULL |
| cotizaciones | asignado_a | usuarios(id) | SET NULL |
| cotizaciones | creado_por | usuarios(id) | SET NULL |
| cotizacion_detalle | id_cotizacion | cotizaciones(id) | CASCADE |
| plantillas | id_ruta | rutas(id) | SET NULL |
| plantillas | creado_por | usuarios(id) | SET NULL |
| seguimientos | id_cotizacion | cotizaciones(id) | CASCADE |
| seguimientos | id_reprogramacion | seguimientos(id) | SET NULL |
| seguimientos | creado_por | usuarios(id) | SET NULL |
| reservas | id_cotizacion | cotizaciones(id) | SET NULL |
| reservas | id_cliente | clientes(id) | SET NULL |
| reservas | anulado_por | usuarios(id) | SET NULL |
| reservas | creado_por | usuarios(id) | SET NULL |
| pasajeros | id_reserva | reservas(id) | CASCADE |
| pagos | id_reserva | reservas(id) | CASCADE |
| pagos | recibido_por | usuarios(id) | SET NULL |
| pagos | anulado_por | usuarios(id) | SET NULL |
| politicas_configuracion | creado_por | usuarios(id) | SET NULL |
| auditoria | id_usuario | usuarios(id) | SET NULL |
| alojamientos | id_reserva | reservas(id) | CASCADE |
| servicios_reserva | id_reserva | reservas(id) | CASCADE |
| rol_permiso | id_rol | roles(id) | CASCADE |
| rol_permiso | id_permiso | permisos(id) | CASCADE |
