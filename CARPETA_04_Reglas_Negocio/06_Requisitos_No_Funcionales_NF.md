# Requisitos No Funcionales del Sistema (NF-01 a NF-08)

**Carpeta:** 04 — Reglas de Negocio, Auditoría y Seguridad  
**Versión:** 1.0 — Especificación Agnóstica  
**Fuente:** Documento 2 — Proyecto Sistema de Reservas (Capítulo 9)

---

| ID | Requisito | Categoría | Criterio de Aceptación |
|----|-----------|-----------|------------------------|
| NF-01 | **Seguridad de acceso:** Las contraseñas deben almacenarse cifradas mediante función hash unidireccional. El sistema debe implementar control de sesión con tiempo de expiración por inactividad (30 minutos). Todas las comunicaciones deben ser cifradas (HTTPS) | Seguridad | Las contraseñas nunca se almacenan en texto plano. La sesión expira automáticamente tras 30 minutos sin actividad. Toda comunicación viaja cifrada |
| NF-02 | **Disponibilidad:** El sistema debe estar disponible 24 horas al día, 7 días a la semana, con una ventana de mantenimiento programado no mayor a 2 horas semanales (preferentemente domingo 2:00-4:00 AM) | Disponibilidad | El sistema responde a peticiones HTTP 24/7. La ventana de mantenimiento no excede 2 horas semanales |
| NF-03 | **Rendimiento:** Las páginas del sistema deben cargar en menos de 3 segundos en una conexión de 10 Mbps. Las operaciones de cálculo y persistencia no deben superar 1 segundo | Rendimiento | Tiempo de carga página < 3s. Tiempo de persistencia < 1s. Verificado con herramientas de monitoreo |
| NF-04 | **Usabilidad:** El proceso de creación de reservas debe seguir un flujo guiado de 5 pasos, permitiendo al usuario completar una reserva en menos de 5 minutos. Máximo 10 clics para crear una cotización desde un lead | UX | Flujo de 5 pasos implementado. Tiempo promedio de creación de reserva < 5 minutos. Cotización desde lead en ≤ 10 clics |
| NF-05 | **Compatibilidad:** El sistema debe funcionar correctamente en los navegadores Chrome, Firefox, Edge y Safari en sus últimas 2 versiones. Debe ser responsive (adaptable a desktop, tablet y celular) | Compatibilidad | Pruebas en 4 navegadores × 3 tamaños de pantalla = 12 combinaciones mínimas. Sin errores de renderizado |
| NF-06 | **Escalabilidad:** La arquitectura debe soportar el crecimiento a múltiples agencias (multi-tenancy liviano) sin requerir reescritura. La base de datos debe manejar al menos 100,000 cotizaciones y 50,000 reservas sin degradación | Escalabilidad | Estructura de datos preparada para agregar `id_agencia` como filtro. Prueba de carga con 100k registros sin superar 5s de respuesta |
| NF-07 | **Backup y recuperación:** El sistema debe realizar backup automático diario de la base de datos y los archivos (PDFs, comprobantes) con retención mínima de 30 días. Debe existir un procedimiento documentado de restauración | Backup | Backup diario automático ejecutado y verificado. Restauración probada en entorno de staging. Retención mínima 30 días |
| NF-08 | **Privacidad y auditoría:** Los datos personales de clientes (DNI, dirección, teléfono, datos de salud) solo deben ser accesibles por roles autorizados (Vendedor, Admin). Toda operación sensible debe quedar registrada en la bitácora de auditoría inmutable | Privacidad | Roles sin permiso no ven datos personales. Auditoría registra quién, cuándo y qué cambió. Los registros de auditoría son inmutables |

---

## Reglas de Validación de Cumplimiento NF

| NF ID | ¿Cómo se verifica? | Frecuencia |
|-------|--------------------|:----------:|
| NF-01 | Revisión de código + pruebas de penetración | Pre-producción y cada release |
| NF-02 | Monitoreo de uptime (herramienta externa) | Continuo |
| NF-03 | Pruebas de rendimiento con datos reales | Cada release |
| NF-04 | Pruebas de usuario (UAT) cronometradas | Pre-producción |
| NF-05 | Pruebas en matriz de navegadores y dispositivos | Cada release |
| NF-06 | Pruebas de carga con datos masivos | Pre-producción |
| NF-07 | Prueba de backup y restore en staging | Mensual |
| NF-08 | Revisión de permisos + verificación de auditoría | Cada release |
