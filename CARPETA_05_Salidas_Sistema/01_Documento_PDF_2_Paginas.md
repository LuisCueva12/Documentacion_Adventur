# Documento PDF Automático de 2 Páginas

**Carpeta:** 05 — Salidas del Sistema  
**Versión:** 1.0 — Especificación Agnóstica

---

## 1. Estructura de Datos de Entrada al Motor de PDF

El sistema debe inyectar la siguiente estructura de datos en el motor de generación de PDF. Esta estructura es la misma tanto para Cotización como para Confirmación de Reserva, variando solo algunos campos.

```
ESTRUCTURA_DATOS_PDF {
    // ═══════════════════════════════════════
    // CABECERA DEL DOCUMENTO
    // ═══════════════════════════════════════
    tipo_documento:            "COTIZACIÓN" | "CONFIRMACIÓN DE RESERVA"
    codigo:                    Texto(20)          // COT-2026-0124 o ADV-2026-0047
    fecha_emision:             Fecha              // 26/07/2026
    valido_hasta:              Fecha | NULL       // NULL si es confirmación de reserva
    vendedor_nombre:           Texto(255)         // "Juan Pérez"
    vendedor_telefono:         Texto(20)          // "+51 987 654 321"

    // ═══════════════════════════════════════
    // DATOS DE LA EMPRESA (fijos)
    // ═══════════════════════════════════════
    empresa_logo:              Binario            // Imagen del logo de Adventur Travel
    empresa_razon_social:      "HORIZONTE ANDINO COMPANY E.I.R.L."
    empresa_ruc:               Texto(11)          // "20XXXXXXXXX"
    empresa_direccion:         Texto(255)         // Jr. [Dirección Fiscal] - Cajamarca
    empresa_telefono:          Texto(20)          // "+51 76 XXXXXX"
    empresa_email:             "info@adventurtravel.pe"
    empresa_web:               "www.adventurtravel.pe"

    // ═══════════════════════════════════════
    // DATOS DEL CLIENTE
    // ═══════════════════════════════════════
    cliente_nombre:            Texto(255)         // "María Fernández"
    cliente_documento:         Texto(50)          // "DNI 87654321"
    cliente_celular:           Texto(20)          // "+51 987 654 321"
    cliente_email:             Texto(255)         // "maria@email.com"

    // ═══════════════════════════════════════
    // DATOS DEL VIAJE
    // ═══════════════════════════════════════
    ruta_nombre:               Texto(255)         // "Cumbemayo + Otuzco - Tour Full Day"
    fecha_viaje:               Fecha              // 15/08/2026
    hora_salida:               Texto(5)           // "08:00"
    lugar_recojo:              Texto(255)         // "Hotel Costa del Sol - Cajamarca"
    fecha_retorno:             Fecha | NULL       // 16/08/2026
    num_pasajeros:             Entero             // 2

    // ═══════════════════════════════════════
    // DESGLOSE FINANCIERO
    // ═══════════════════════════════════════
    items: Lista de {
        concepto:              Texto(255)         // "Movilidad", "Guía", "Entradas", ...
        cantidad:              Entero             // 1, 2, ...
        precio_unitario:       Decimal(10,2)      // 200.00
        subtotal:              Decimal(10,2)      // 200.00
    }
    subtotal_base:             Decimal(10,2)      // 560.00
    margen_porcentaje:         Decimal(5,2)       // 15.00%
    margen_monto:              Decimal(10,2)      // 84.00
    descuento_porcentaje:      Decimal(5,2)       // 5.00%
    descuento_monto:           Decimal(10,2)      // 32.20
    total:                     Decimal(10,2)      // 611.80
    precio_por_persona:        Decimal(10,2)      // 305.90

    // ═══════════════════════════════════════
    // LISTA DE PASAJEROS (solo CONFIRMACIÓN)
    // ═══════════════════════════════════════
    pasajeros: Lista de {
        nombre:                Texto(255)         // "María Fernández"
        documento:             Texto(50)          // "DNI 87654321"
        edad:                  Entero             // 36
        contacto_emergencia:   Texto(255)         // "Pedro Fernández"
        telefono_emergencia:   Texto(20)          // "+51 976 543 210"
        requiere_seguro:       Booleano           // FALSE
    }

    // ═══════════════════════════════════════
    // INFORMACIÓN DE PAGOS (solo CONFIRMACIÓN)
    // ═══════════════════════════════════════
    total_pagado:              Decimal(10,2)      // 200.00
    saldo_pendiente:           Decimal(10,2)      // 411.80
    estado_pago:               "CONFIRMADA" | "PENDIENTE DE PAGO"
    pagos: Lista de {
        monto:                 Decimal(10,2)      // 200.00
        metodo:                Texto(50)          // "Yape"
        fecha:                 Fecha              // 30/07/2026
        referencia:            Texto(255)         // "YAPE-123ABC"
    }

    // ═══════════════════════════════════════
    // DATOS OPERATIVOS (solo CONFIRMACIÓN)
    // ═══════════════════════════════════════
    movilidad_asignada:        Texto(255)         // "Toyota Hiace - AB-1234"
    guia_asignado:             Texto(255)         // "Guillermo Torres"
    conductor_asignado:        Texto(255)         // "Carlos Ramírez"
    notas_operativas:          Texto(L)           // "Cliente solicita silla de ruedas"

    // ═══════════════════════════════════════
    // POLÍTICAS (Página 2)
    // ═══════════════════════════════════════
    politicas: Lista de {
        seccion:               Texto(50)          // "Cancelación", "Pago", "Responsabilidad", "General"
        titulo:                Texto(255)         // "Política de Cancelación"
        contenido:             Texto(L)           // Texto completo de la política
    }

    // ═══════════════════════════════════════
    // CONTROL Y SEGURIDAD
    // ═══════════════════════════════════════
    codigo_verificacion:       Texto(30)          // "ADV-2026-0047-A1B2C3"
    fecha_generacion:          FechaHora          // 26/07/2026 15:30:00
}
```

---

## 2. Maqueta de la Página 1 (Detalle Comercial y Operativo)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                       [LOGO_ADVENTUR]                                       ║
║                HORIZONTE ANDINO COMPANY E.I.R.L.                            ║
║                     RUC: 20XXXXXXXXX                                        ║
║               Jr. [Dirección Fiscal] - Cajamarca, Perú                      ║
║                 Tel: +51 76 XXXXXX | info@adventurtravel.pe                 ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║   TIPO DE DOCUMENTO:    [COTIZACIÓN / CONFIRMACIÓN DE RESERVA]              ║
║   CÓDIGO:               [COT-2026-0124 / ADV-2026-0047]                     ║
║   FECHA DE EMISIÓN:     [26/07/2026]                                        ║
║   VÁLIDO HASTA:         [10/08/2026]  (solo cotización)                     ║
║   VENDEDOR:             [Juan Pérez]                                        ║
║   TELÉFONO VENDEDOR:    [+51 987 654 321]                                   ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   DATOS DEL CLIENTE                                                          ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   NOMBRES:              [María Fernández López]                             ║
║   DOCUMENTO:            [DNI N° 87654321]                                   ║
║   CELULAR:              [+51 987 654 321]                                   ║
║   CORREO:               [maria.fernandez@email.com]                         ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   DETALLE DEL PAQUETE / VIAJE                                               ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   RUTA:                 [Cumbemayo + Otuzco - Tour Full Day]                ║
║   FECHA DE VIAJE:       [15 de agosto de 2026]                              ║
║   HORA DE SALIDA:       [08:00 hrs]                                         ║
║   LUGAR DE RECOJO:      [Hotel Costa del Sol - Cajamarca]                   ║
║   FECHA DE RETORNO:     [16 de agosto de 2026]                              ║
║   N° DE PASAJEROS:      [2]                                                 ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   DESGLOSE DE COSTOS                                                         ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   Concepto                          Cant.   P.Unit.      Subtotal           ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   Movilidad                          1      S/ 200.00   S/ 200.00           ║
║   Guía de turismo                    1      S/ 80.00    S/ 80.00            ║
║   Entradas (Cumbemayo + Otuzco)      2      S/ 60.00    S/ 120.00           ║
║   Alimentación (almuerzo)            2      S/ 80.00    S/ 160.00           ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   SUBTOTAL BASE                                        S/ 560.00            ║
║   MARGEN COMERCIAL (15.00%)                             S/ 84.00            ║
║   DESCUENTO (-5.00%)                                   -S/ 32.20            ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   TOTAL VENTA                                           S/ 611.80            ║
║   PRECIO POR PERSONA (2 pax)                            S/ 305.90            ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   LISTA DE PASAJEROS (SOLO CONFIRMACIÓN)                                     ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   #  NOMBRE                   DOCUMENTO       EDAD  CONTACTO EMERGENCIA     ║
║   1  María Fernández López    DNI 87654321    36    Pedro Fernández         ║
║   2  Carlos Fernández García  DNI 76543210    34    Ana García              ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   INFORMACIÓN DE PAGO (SOLO CONFIRMACIÓN)                                    ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   TOTAL:                     S/ 611.80                                      ║
║   TOTAL PAGADO:              S/ 200.00 (ADELANTO)                           ║
║   ─ 30/07/2026 - YAPE - Ref: YAPE-123ABC                                   ║
║   SALDO PENDIENTE:           S/ 411.80                                      ║
║   ESTADO:                    [CONFIRMADA / PENDIENTE DE PAGO]                ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   DATOS OPERATIVOS (SOLO CONFIRMACIÓN)                                       ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   MOVILIDAD ASIGNADA:       [Toyota Hiace - AB-1234]                        ║
║   GUÍA ASIGNADO:            [Guillermo Torres]                              ║
║   CONDUCTOR ASIGNADO:       [Carlos Ramírez]                                ║
║   NOTAS:                    [Cliente solicita silla de ruedas]              ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   [CÓDIGO DE BARRAS / QR]                                                    ║
║   Documento generado electrónicamente: [26/07/2026 15:30:00]                 ║
║   Código de verificación: [ADV-2026-0047-A1B2C3]                            ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 3. Maqueta de la Página 2 (Políticas y Condiciones)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                      POLÍTICAS Y CONDICIONES                                 ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║   1. POLÍTICA DE CANCELACIÓN                                                 ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   [Contenido configurable desde el módulo de políticas - sección            ║
║    "Cancelación"]                                                            ║
║                                                                              ║
║   Ejemplo de contenido:                                                     ║
║   - Cancelaciones con más de 7 días de anticipación:                       ║
║     Reembolso del 100% del monto pagado.                                    ║
║   - Cancelaciones entre 3 y 7 días antes del viaje:                        ║
║     Reembolso del 50% del monto pagado.                                     ║
║   - Cancelaciones con menos de 3 días de anticipación:                     ║
║     No aplica reembolso.                                                    ║
║   - No-show el día del viaje:                                               ║
║     No aplica reembolso.                                                    ║
║                                                                              ║
║   2. POLÍTICA DE PAGOS                                                       ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   [Contenido configurable - sección "Pago"]                                 ║
║                                                                              ║
║   Ejemplo de contenido:                                                     ║
║   - Se requiere un adelanto mínimo del 30% para confirmar la reserva.       ║
║   - El saldo pendiente debe cancelarse antes del inicio del servicio.       ║
║   - Métodos de pago aceptados: Efectivo, Yape, Plin, Transferencia         ║
║     Bancaria, Tarjeta de crédito/débito.                                    ║
║                                                                              ║
║   3. POLÍTICA DE RESPONSABILIDAD                                             ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   [Contenido configurable - sección "Responsabilidad"]                      ║
║                                                                              ║
║   Ejemplo de contenido:                                                     ║
║   - Adventur Travel no se responsabiliza por objetos personales perdidos   ║
║     durante el servicio turístico.                                          ║
║   - El cliente es responsable de proporcionar información médica veraz      ║
║     sobre condiciones preexistentes, alergias y restricciones alimenticias.  ║
║   - Adventur Travel se reserva el derecho de modificar itinerarios por     ║
║     razones de fuerza mayor (clima, desastres naturales, protestas).        ║
║                                                                              ║
║   4. CONDICIONES GENERALES                                                   ║
║   ─────────────────────────────────────────────────────────────────────      ║
║   [Contenido configurable - sección "General"]                              ║
║                                                                              ║
║   Ejemplo de contenido:                                                     ║
║   - Los horarios y rutas pueden estar sujetos a cambios por fuerza mayor.  ║
║   - Las tarifas no incluyen IGV (según legislación turística peruana,      ║
║     Decreto Legislativo N° 1429).                                          ║
║   - El presente documento es una [COTIZACIÓN / CONFIRMACIÓN] generada      ║
║     electrónicamente y válida sin firma manuscrita.                         ║
║   - Código de verificación: [ADV-2026-0047-A1B2C3]                         ║
║                                                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║   HORIZONTE ANDINO COMPANY E.I.R.L.                                         ║
║   RUC: 20XXXXXXXXX                                                          ║
║   Jr. [Dirección Fiscal] - Cajamarca - Perú                                 ║
║   Tel: +51 76 XXXXXX | info@adventurtravel.pe                               ║
║   www.adventurtravel.pe                                                     ║
║                                                                              ║
║   Documento generado electrónicamente el [26/07/2026 15:30:00]               ║
╚══════════════════════════════════════════════════════════════════════════════╝
```
