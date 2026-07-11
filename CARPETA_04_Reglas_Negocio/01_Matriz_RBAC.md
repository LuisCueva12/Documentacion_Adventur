# Matriz de Control de Acceso por Roles (RBAC)

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Roles del Sistema

| Rol | Código Interno | Descripción |
|-----|---------------|-------------|
| **Administrador** | `ADMIN` | Control total del sistema. Gestión de usuarios, configuración, datos maestros |
| **Vendedor** | `VENDEDOR` | Gestión comercial directa: leads, cotizaciones, seguimientos, conversión a reservas |
| **Supervisor** | `SUPERVISOR` | Supervisión de equipo comercial. Aprueba descuentos y visualiza métricas del equipo |
| **Operaciones** | `OPERACIONES` | Gestión operativa de reservas confirmadas: movilidad, guías, conductores, notas |
| **Contabilidad** | `CONTABILIDAD` | Registro y verificación de pagos. Control de caja |
| **Gerencia** | `GERENCIA` | Acceso de SOLO LECTURA a reportes, KPIs, dashboards y datos consolidados |

---

## 2. Matriz de Permisos (Acciones por Entidad)

### Leyenda

| Símbolo | Significado |
|:-------:|-------------|
| **C** | Crear (Crear nuevos registros) |
| **R** | Leer (Consultar/visualizar registros) |
| **U** | Editar (Modificar registros existentes) |
| **D** | Anular (Eliminación lógica — soft delete) |
| **A** | Aprobar (Aprobar descuentos que exceden límite del vendedor) |
| **—** | Sin acceso (acción no permitida) |

### 2.1 Entidades Principales

| Entidad / Acción | ADMIN | VENDEDOR | SUPERVISOR | OPERACIONES | CONTABILIDAD | GERENCIA |
|-----------------|:-----:|:--------:|:----------:|:-----------:|:------------:|:--------:|
| **Usuarios** | C,R,U,D | — | R(equipo) | — | — | R |
| **Clientes** | C,R,U,D | C,R,U(propios) | C,R,U(equipo) | R | R | R |
| **Leads** | C,R,U,D | C,R,U(propios) | C,R,U(equipo) | — | — | R |
| **Cotizaciones** | C,R,U,D,A | C,R,U(propias) | C,R,U,A(equipo) | R(solo reserva) | R(montos) | R |
| **Cotización Detalle** | C,R,U,D | C,R,U(propias) | C,R,U(equipo) | — | — | R |
| **Plantillas** | C,R,U,D | R | C,R,U | R | R | R |
| **Rutas** | C,R,U,D | R | C,R,U | R | R | R |
| **Seguimientos** | C,R,U,D | C,R,U(propios) | R,U(equipo) | — | — | R |
| **Reservas** | C,R,U,D | C,R,U(propias) | C,R,U(equipo) | R,U(datos op.) | R | R |
| **Pasajeros** | C,R,U | C,R,U(propias) | R(equipo) | R,U | — | R |
| **Pagos** | C,R,U,D | R(propias) | R(equipo) | R | C,R,U,D | R |
| **Reportes** | R | R(propios) | R(equipo) | R | R | R |
| **Configuración** | C,R,U,D | — | R | — | — | R |
| **Auditoría** | R | — | R(equipo) | — | — | R |
| **Políticas** | C,R,U,D | R | R | R | R | R |

### 2.2 Restricciones de Visibilidad

| Restricción | Significado |
|:-----------:|-------------|
| **(propios)** | Solo registros donde `asignado_a = ID_del_usuario_actual` o `creado_por = ID_del_usuario_actual` |
| **(equipo)** | Registros de los vendedores cuyo `id_supervisor = ID_del_usuario_actual` |
| **(solo reserva)** | Solo ve cotizaciones que ya fueron convertidas a reserva (estado = RESERVA) |
| **(montos)** | Solo lectura de campos monetarios. No ve datos personales del cliente |
| **(datos op.)** | Solo edición de campos operativos: movilidad, guía, conductor, notas. NO puede editar montos ni estados financieros |

---

## 3. Reglas de Descubrimiento y Visibilidad

| ID | Regla | Descripción |
|----|-------|-------------|
| V-R01 | **Visibilidad de Vendedor** | Un Vendedor SOLO ve leads, cotizaciones y reservas donde `asignado_a = su_id` o `creado_por = su_id`. No ve registros de otros vendedores |
| V-R02 | **Visibilidad de Supervisor** | Un Supervisor ve TODOS los registros de los vendedores cuyo `id_supervisor = su_id`. También ve sus propios registros si también actúa como vendedor |
| V-R03 | **Visibilidad de Operaciones** | Operaciones ve SOLO reservas en estado `Confirmada` o `Finalizada`. No ve cotizaciones en estados previos. No ve leads |
| V-R04 | **Visibilidad de Contabilidad** | Contabilidad ve montos de reservas confirmadas y registros de pagos. No ve datos personales de clientes ni detalles de cotizaciones |
| V-R05 | **Visibilidad de Gerencia** | Gerencia tiene ACCESO DE SOLO LECTURA (Read-Only) a todos los módulos y todos los registros. No puede crear, editar ni anular nada |
| V-R06 | **Visibilidad de Admin** | Administrador tiene ACCESO TOTAL (CRUD) a todos los módulos, EXCEPTO eliminar físicamente registros (solo soft delete permitido) |

---

## 4. Límites de Descuento por Rol

| Rol | % Máximo sin Aprobar | Requiere Aprobación de | Límite Absoluto |
|-----|:---------------------:|:----------------------:|:----------------:|
| Vendedor | 10% del Subtotal con Margen (SM) | Supervisor | 50% |
| Supervisor | 15% del SM | Administrador | 50% |
| Administrador | 50% del SM | — (autónomo) | 50% |
| > 50% | **BLOQUEADO** para todos los roles | Decisión fuera del sistema | No permitido |

### Pseudocódigo de Validación

```
FUNCIÓN ValidarDescuento(porcentaje_descuento, rol_usuario, descuento_aprobado_por):
    SI porcentaje_descuento > 50:
        BLOQUEAR("El descuento no puede exceder el 50% del subtotal con margen")
    FIN SI
    
    SEGÚN rol_usuario:
        CASO "Vendedor":
            SI porcentaje_descuento > 10:
                SI descuento_aprobado_por ES NULL:
                    BLOQUEAR("Descuento del " + porcentaje_descuento + "% requiere aprobación de Supervisor")
                FIN SI
            FIN SI
        
        CASO "Supervisor":
            SI porcentaje_descuento > 15:
                SI descuento_aprobado_por ES NULL:
                    BLOQUEAR("Descuento del " + porcentaje_descuento + "% requiere aprobación de Administrador")
                FIN SI
            FIN SI
        
        CASO "Administrador":
            // Puede aprobar hasta 50%, sin restricción adicional
            // Si el descuento es > 15%, él mismo puede auto-aprobar
        
        CASO OTRO:
            BLOQUEAR("El rol " + rol_usuario + " no tiene permiso para aplicar descuentos")
    FIN SEGÚN
    
    RETORNAR APROBADO
FIN FUNCIÓN
```

---

## 5. Acciones Bloqueadas por Rol (Prohibiciones Explícitas)

| Rol | Acciones Prohibidas |
|-----|---------------------|
| **Vendedor** | Crear usuarios, modificar plantillas, modificar rutas, anular reservas de otros, eliminar cualquier registro, ver montos agregados del equipo, aprobar descuentos |
| **Supervisor** | Eliminar registros, modificar configuración del sistema, crear usuarios, ver datos de vendedores de otro supervisor |
| **Operaciones** | Crear cotizaciones, modificar montos, registrar pagos, ver leads, anular reservas |
| **Contabilidad** | Crear cotizaciones, editar datos de clientes, ver datos personales, crear usuarios |
| **Gerencia** | CUALQUIER acción de escritura (crear, editar, anular, aprobar) |
| **Administrador** | Eliminar físicamente registros (solo soft delete) |
