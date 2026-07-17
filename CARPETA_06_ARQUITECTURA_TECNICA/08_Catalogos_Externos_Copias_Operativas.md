# Catálogos Externos y Copias Operativas Editables

**Carpeta:** 06 — Arquitectura Técnica  
**Versión:** 1.0  
**Ámbito:** Paquetes, hoteles, habitaciones, movilidades y servicios

---

## 1. Decisión arquitectónica

Los sistemas externos son fuentes de referencia, no propietarios del contenido de una cotización o reserva ya creada.

Cuando un usuario selecciona un paquete, hotel, habitación o movilidad, Adventur crea una **copia operativa** de los campos necesarios. La copia se puede ajustar para el cliente sin modificar el registro externo ni depender de que la fuente siga disponible.

```text
Fuente externa → Adaptador → Catálogo local sincronizado → Copia operativa editable
                                                       ├─ Cotización / ítems
                                                       └─ Reserva / alojamientos / servicios
```

Esta regla evita dos errores:

1. Una sincronización posterior no cambia documentos históricos ni precios ya ofrecidos.
2. Una caída de la fuente externa no impide consultar, editar o renderizar una operación existente.

---

## 2. Responsabilidad de cada nivel

| Nivel | Responsabilidad | Editable en la operación |
|---|---|---:|
| Fuente externa | Publicar el catálogo original y el precio vigente | No |
| Adaptador | Traducir el formato externo al contrato del dominio | No |
| Catálogo local | Mantener una referencia sincronizada y tolerante a fallos | Sí, mediante sincronización o administración autorizada |
| Ítem de cotización | Conservar descripción, cantidad, precio y procedencia usados para el cliente | Sí, antes del envío |
| Alojamiento de reserva | Conservar hotel, habitación, cama, plan y fechas operativas | Sí, mientras la reserva admita cambios |
| Servicio de reserva | Conservar descripción, proveedor, precio, cantidad y fecha | Sí, mientras la reserva admita cambios |

---

## 3. Campos mínimos de trazabilidad

Toda copia importada debe conservar:

| Campo lógico | Uso |
|---|---|
| `tipo_origen` | `paquete`, `hotel`, `movilidad` o `manual` |
| `id_origen` | Identificador del registro en el catálogo local |
| `codigo_origen` | Identificador estable de la fuente (`wp_id`, código externo o código de habitación) |
| Campos descriptivos | Instantánea legible que se mostrará en formularios, detalle y PDF |
| Precio y cantidad | Valores aceptados para esa cotización o reserva, no referencias dinámicas |

Para alojamiento se conservan además `hotel_catalogo_id`, `hotel_codigo_externo` y `habitacion_codigo_externo`. Para movilidad se conserva el nombre asignado y sus claves de catálogo.

---

## 4. Reglas de edición

### Cotización

- Un dato importado es una sugerencia inicial; descripción, precio y cantidad son editables.
- Solo se pueden reemplazar ítems mientras la cotización esté en `nuevo` o `cotizado`.
- Al guardar cambios se recalculan subtotal, margen, descuento, total y precio por pasajero.
- Después del envío se bloquea la composición para conservar lo comunicado al cliente.

### Reserva

- Al convertir o seleccionar una cotización, sus ítems se copian como servicios operativos.
- Alojamientos, movilidad, guía, conductor y servicios son copias propias de la reserva.
- Una reserva `Finalizada` o `Anulada` no admite cambios operativos.
- Editar una copia nunca actualiza WooCommerce ni los sistemas externos de hoteles o movilidades.

---

## 5. Contrato de presentación

Las pantallas y documentos deben renderizar la copia operativa, no consultar la fuente externa en cada apertura.

La interfaz debe mostrar, cuando exista:

- nombre y descripción comprensibles;
- ciudad, categoría o duración del paquete;
- hotel, estrellas, habitación, cama, alimentación y capacidad;
- movilidad, tipo y capacidad;
- precio unitario, cantidad y subtotal;
- etiqueta discreta de procedencia, sin exponer detalles técnicos al cliente final.

El precio web puede mostrarse como referencia al momento de seleccionar, pero el valor persistido es el que gobierna la cotización y el PDF.

---

## 6. Criterios de aceptación

| ID | Criterio |
|---|---|
| CE-01 | Una cotización con varios servicios conserva todos sus ítems al guardarse |
| CE-02 | Un paquete, hotel o movilidad importado permite cambiar descripción, precio y cantidad antes del envío |
| CE-03 | Hotel y movilidad exponen su código externo para consultar el precio sugerido |
| CE-04 | La reserva recibe los servicios de la cotización y permite ajustarlos sin cambiar el catálogo |
| CE-05 | Una sincronización posterior no altera cotizaciones, reservas ni PDF existentes |
| CE-06 | El detalle de reserva muestra coherentemente alojamientos, servicios y procedencia disponible |
| CE-07 | Las operaciones siguen consultables si una API externa está caída |

