# Conector WMS Altatex: alcance técnico e indicadores

| | |
|---|---|
| **Versión** | 1.0 |
| **Fecha** | 19/11/2026 |
| **Objetivo** | Definir qué información entrega el conector WMS Altatex y qué indicadores automáticos se pueden construir con ella para una app de tablero en HTML. |
| **Base** | Definición de cada endpoint del conector y consultas reales de prueba hechas el 19/11/2026 (muestras chicas, solo lectura). |
| **Fuera de alcance** | Desarrollo de la app. Este documento no incluye código. |

---

## 1. Resumen

El conector **WMS Altatex** es un servidor MCP (Model Context Protocol) **de solo lectura** sobre el WMS. Ofrece:

- **10 consultas paginadas** (JSON): pedidos, detalle de pedido, avance de picking, stock, stock distribuible, ingresos, cargas iniciales, cobertura de cargas, productividad por hora y detalle de productividad.
- **9 exportaciones CSV completas**, una por cada consulta salvo cobertura de cargas. Cada una genera el archivo entero en una sola foto de la base y devuelve un link de descarga.

Cubre seis dominios: **pedidos, picking, stock, recepción (ingresos), cargas iniciales y productividad**. No cubre **despacho ni expedición**, **fechas de cierre de pedidos**, **capacidad de ubicaciones** ni **catálogos maestros** (almacenes, estados, transportes). Ver la sección 7.

El conector **no envía eventos ni notificaciones**. Un tablero "automático" tiene que **consultar periódicamente** (polling) y calcular los indicadores sobre esas respuestas.

---

## 2. Arquitectura y acceso

### 2.1 Cómo funciona el conector

| Aspecto | Detalle |
|---|---|
| Protocolo | MCP sobre HTTP (JSON-RPC). Cada consulta es una "tool" con un parámetro `query` en JSON. |
| Autenticación | OAuth con vinculación de cuenta. El conector usa los permisos de la cuenta WMS vinculada. Nunca se envían usuario ni contraseña del WMS en las consultas. |
| Permisos | Las consultas de productividad requieren una cuenta **ADMIN del WMS**. El resto, una cuenta con acceso de lectura. |
| Host observado | `wms-mcp-cjd3fpbehbaxcvh8.westus3-01.azurewebsites.net` (Azure App Service). Es el host de las descargas CSV. |
| Escritura | Ninguna. El conector no modifica, reserva ni asigna nada en el WMS. |
| Zona horaria | Los filtros de fecha se expresan en **hora local del WMS** (formato ISO sin zona, por ejemplo `2026-11-01T00:00:00`). |

### 2.2 Cómo consumirlo desde una app HTML

Un navegador no debería hablar MCP directamente ni guardar tokens OAuth. Opciones:

| Opción | Descripción | Ventajas | Contras |
|---|---|---|---|
| **A. Backend intermedio (recomendada)** | Un servicio propio (Node, Python o similar) actúa como cliente MCP con el token OAuth, consulta el conector cada cierto tiempo, recorre todas las páginas, calcula los indicadores y expone una API REST/JSON simple al HTML. | Token protegido. Caché. Una sola consulta al WMS sirve a muchos usuarios. Indicadores ya calculados. | Hay que mantener un servicio más. |
| B. Exportaciones CSV | El backend o un proceso programado genera exportaciones y descarga los CSV. | Datos completos en una sola foto, sin paginar. | El link vence **en 1 hora**. Con OAuth, la descarga exige la misma cuenta que generó el archivo. No sirve para tiempo real. |
| C. Endpoints REST internos del WMS | Las definiciones del conector mencionan las pantallas o servicios que replica: `ordenRecepcionProducto/find_by`, `picking/find_avance_by`, `indicadores/productividad/hora` e `indicadores/productividad/detalle`. | Sin intermediarios. | **Sin validar.** Hay que pedir al equipo del WMS las URL, la autenticación y la política de CORS. |

> **A validar con el equipo del WMS:** la URL del endpoint MCP, el flujo OAuth para un cliente que no sea Claude (client_id y scopes), los límites de consultas, el `limit` máximo por página y el CORS si se piensa en la opción C.

### 2.3 Paginación y snapshot

- Las consultas JSON paginan con `limit` y `offset`. Toda respuesta trae `offset`, `limit`, `returned`, `mayHaveMore` y `nextOffset`.
- **Para cualquier total hay que traer todas las páginas** (seguir mientras `mayHaveMore = true`).
- Cada página es una foto distinta del WMS. Si los datos cambian entre páginas, la suma puede no cuadrar. Si se necesita un corte consistente, conviene usar la exportación CSV, que es una sola foto.
- `stock` usa `limit = 100` por defecto. El máximo no está documentado.

---

## 3. Catálogo de endpoints

Notación: **[R]** = filtro obligatorio. Las fechas `createdFrom` incluyen el límite y `createdTo` lo excluye, salvo que se indique otra cosa.

### 3.1 `wms_pedidos`: pedidos

| | |
|---|---|
| **Qué devuelve** | Pedidos con totales de unidades por pedido. Con `detail=true`, todas las líneas de los pedidos encontrados. |
| **Filtros** | `createdFrom`/`createdTo` **[R]** (máximo 3 meses, salvo búsqueda por `pedidoId`), `mode`, `tipoOrden`, `cliente`, `tienda`, `transporteId`, `operadorLogistico`, `sellerId`, `sku`, `master`, `pedidoId`, `detail`, `limit`, `offset` |
| **Modo (`mode`)** | `ALL_ACTIVE` (por defecto): pendientes, planificados o en picking. `PENDING`: pendientes. `PICKING`: planificados o en picking. `ALL`: todos los estados. |
| **Campos** | `pedidoId`, `pedido`, `tracking`, `createdAt`, `estado`, `tipoOrden`, `sellerId`, `operadorLogistico`, `requestedUnits`, `plannedUnits`, `pickedUnits`, `separatedUnits`, `pendingUnits`, `ageDays` |
| **CSV** | `wms_export_pedidos_csv` (mismos filtros; exporta todas las líneas) |

Valores observados: tipos de orden `ECOM`, `REP` y `B2B`. Estados `OD_PENDIENTE`, `OD_PLANIFICADO`, `OD_PICKING`, `OD_PICKEADA_PARCIAL`, `OD_PICKEADA` y `OD_SEPARADA`. Operadores `INTRALOG` y `REP`.

Notas:
- Si se filtra por `sku` o `master`, se eligen los pedidos que contienen ese producto, pero los totales son **del pedido completo**.
- Cliente, tienda y transporte solo vienen a nivel línea (con `detail=true` o `wms_pedido_details`).

### 3.2 `wms_pedido_details`: líneas de un pedido

| | |
|---|---|
| **Qué devuelve** | Una fila por línea de producto de un pedido, en cualquier estado. |
| **Filtros** | `pedidoId` **[R]**, `limit`, `offset` |
| **Campos** | Los del pedido, más `paqueteId`, `cliente`, `tienda`, `transporteId`, `transporte`, `lineId`, `sku`, `master` y las unidades por línea (`requested`, `planned`, `picked`, `separated`, `pending`). |
| **CSV** | `wms_export_pedido_details_csv` |

### 3.3 `wms_picking_progress`: avance de picking

| | |
|---|---|
| **Qué devuelve** | Avance por picking (ola) y piso. |
| **Filtros** | `createdFrom`/`createdTo` **[R]** (máximo 3 meses, salvo búsqueda por `pickingId`), `pickingId`, `nombre` (texto contenido en el nombre), `tipoOrden`, `finishing`, `limit`, `offset` |
| **`finishing`** | `false` (por defecto): solo pickings activos. `true`: todos los estados. |
| **Campos** | `pickingId`, `tipo` (por ejemplo `OLA`), `nombre`, `createdAt`, `piso`, `tipoOrden`, `estado`, `requiredUnits`, `pickedUnits`, `separatedUnits`, `assignedLocations`, `unassignedLocations`, `pickingPercent`, `finishingPercent`, `assignmentPercent`, `pendingPickingUnits`, `pendingFinishingUnits` |
| **CSV** | `wms_export_picking_progress_csv` |

Fórmulas del WMS: `picking % = picked / required`, `finishing % = separated / picked` y `asignación % = ubicaciones asignadas / total de ubicaciones`. La asignación cuenta **ubicaciones, no unidades**. Estados observados: `PI_PENDIENTE` y `PI_EN_PREPARACION`.

### 3.4 `wms_stock`: stock físico

| | |
|---|---|
| **Qué devuelve** | Unidades físicas por almacén, sector, disponibilidad, seller, master y SKU. Con `detail=true`, también pallet y ubicación. |
| **Filtros** | `availability` (`D` disponible, por defecto, o `ND` no disponible), `almacenId` o `almacenNombre`, `sectorId` o `sectorNombre`, `ubicacion` (código exacto, por ejemplo `SP2`), `sellerId`, `master`, `sku`, `minAgeDays`, `cargaNumero`, `cargaCurva`, `detail`, `limit`, `offset` |
| **Campos** | `almacenId`, `sectorId`, `sector`, `availability`, `sellerId`, `master`, `sku`, `units`. Con `detail=true`, además: `palletId`, `pallet`, `ubicacionId`, `ubicacion`, `oldestEntryDate`, `newestEntryDate`. |
| **CSV** | `wms_export_stock_csv` (detalle activado por defecto) |

Notas:
- Son unidades **físicas**: no se descuentan las unidades ya planificadas para pedidos.
- `minAgeDays` mide la antigüedad desde la **fecha de ingreso** al depósito, no la rotación de ventas. Excluye stock sin fecha de ingreso.
- `cargaNumero` y `cargaCurva` traen en una sola consulta el stock de todos los masters de una carga.

Sectores observados: `N2_ALMACEN`, `RECEPCION_CUYO` y `ALMACEN_CUYO`.

### 3.5 `wms_distributable_stock`: stock distribuible

| | |
|---|---|
| **Qué devuelve** | Unidades listas para distribuir en un almacén: stock no disponible (ND), pallet de tipo PT, sin remito de pallet y con contenido positivo. |
| **Filtros** | `almacenId` o `almacenNombre` **[R]**, `sellerId`, `master`, `sku`, `detail`, `limit`, `offset` |
| **Campos** | Los mismos que `wms_stock` con detalle (pallet y ubicación). |
| **CSV** | `wms_export_distributable_stock_csv` |

Es un **pool compartido**, no una asignación: no hay que sumarlo una vez por carga.

### 3.6 `wms_ingresos`: recepción

| | |
|---|---|
| **Qué devuelve** | Productos y etiquetas de las órdenes de recepción (equivale a `ordenRecepcionProducto/find_by`). |
| **Filtros** | `fecha_recepcion_dd`/`fecha_recepcion_hh` (formato `yyyy-MM-dd`, **ambas fechas incluidas**), `identificador`, `sellerId`, `sku`, `ean` (texto contenido), `etiqueta`, `estado_producto_id`, `estado_etiqueta_id`, `tipo_orden`, `origen`, `proveedores`, `legajo`, `color`, `talle`, `temporada`, `jerarquia`, `sub_jerarquia`, `rango_edad`, `seleccion`, `limit`, `offset` |
| **Campos** | `or_producto_id`, `identificador`, `seller_id`, `razon_social`, `fecha_recepcion`, `fecha_ingreso`, `fecha_aprobacion`, `sku`, `master`, `ean`, `descripcion`, `unidades_originales`, `unidades`, `unidades_guardadas`, `unidades_separadas`, `paquetes_x_sku`, `total_paquetes`, `estado_producto_id`/`_valor`, `estado_etiqueta_id`/`_valor`/`_color_hex`, `etiqueta`, `etiqueta_caja`, `pallet`, `pallet_actual`, `ubicacion`, `sucursal_nombre`, `sucursal_codigo`, `tipo_orden`, `origen`, `legajo`, `proveedores`, `proveedores_codigos`, `remito_ingreso`, `color`, `talle`, `rango_edad`, `temporada`, `seleccion`, `jerarquia`, `sub_jerarquia` |
| **CSV** | `wms_export_ingresos_csv` |

Notas:
- Cada fila es un grupo producto-etiqueta. `unidades_originales` **se repite en cada etiqueta**, así que para totales por producto hay que **deduplicar por `or_producto_id`**.
- `ubicacion` es un campo heredado y puede venir vacío.
- Valores observados: `tipo_orden` y `origen` = `DEV_TIENDA`. Estado `1` = `OR_PENDIENTE`, con color `#5ea3cb`.

### 3.7 `wms_cargas_iniciales`: cargas iniciales

| | |
|---|---|
| **Qué devuelve** | Avance de asignación de cada carga inicial, por etapa (`curva`). |
| **Filtros** | `numero`, `curva` (etapa), `master`, `sku`, `groupBy` (`CARGA` por defecto, `ETAPA`, `MASTER` o `SKU`), `activeOnly` (por defecto `true`: estados `CI_PENDIENTE` y `CI_EN_PREPARACION`), `pendingOnly`, `limit`, `offset` |
| **Campos** | `cargaInicialId`, `numero`, `estado`, `curva`, `requestedUnits`, `assignedUnits`, `pendingAssignmentUnits`, `assignmentPercent`. Los campos de separación no se usan; ver la regla de abajo. |
| **CSV** | `wms_export_cargas_iniciales_csv` (agrupa por `SKU` por defecto) |

Reglas de negocio:
- La etapa es `curva` y **no depende del master**. Cada etapa se informa por separado.
- **Unidades completadas = `assignedUnits`**. El WMS no usa `unidades_separadas` en las cargas, así que hay que ignorar `separatedUnits`, `pendingSeparationUnits` y `separationPercent`.
- Pendiente = pedido − asignado. Avance = asignado / pedido × 100 (cero si no hay unidades pedidas).
- Que una carga esté asignada no significa que esté despachada.
- El significado del filtro `pendingOnly` hay que validarlo antes de usarlo.

### 3.8 `wms_carga_separables`: cobertura inmediata de una carga

| | |
|---|---|
| **Qué devuelve** | Por SKU, cruza lo pendiente de asignar de una carga y etapa con el stock disponible en una ubicación. |
| **Filtros** | `numero` **[R]**, `curva` **[R]**, `sellerId` **[R]**, `almacenId` o `almacenNombre` **[R]**, `ubicacion` (por defecto `SP`), `limit`, `offset` |
| **Campos** | `sku`, `requestedUnits`, `assignedUnits`, `pendingAssignmentUnits`, `availableUnits`, `separableUnits` (el menor entre disponible y pendiente), `remainingPendingUnits`, `numero`, `curva`, `almacenId`, `sellerId`, `ubicacion` |
| **CSV** | **No tiene.** Solo JSON paginado. |

El total cubrible es la **suma de `separableUnits` de todas las páginas**. Nunca se calcula como el mínimo entre los totales globales. Incluye los SKU pendientes que no tienen stock. No reserva stock.

### 3.9 `wms_productividad_hora`: productividad por hora (requiere ADMIN)

| | |
|---|---|
| **Qué devuelve** | Unidades y eventos por día, hora, usuario y proceso (equivale a `indicadores/productividad/hora`). Incluye conteos cíclicos. |
| **Filtros** | `desde`/`hasta` **[R]** (**ambos incluidos**, máximo 31 días), `ssoId` (usuario), `funcion` (proceso), `grupos` (`A`, `B`, `C`; si se omite, todos; la pantalla del WMS arranca con `A`), `marcas` (por ejemplo `CHEEKY,AWADA`), `limit`, `offset` |
| **Campos** | `dia`, `hora`, `ssoId`, `funcion`, `unidades`, `eventos`, `minFecha`, `maxFecha` |
| **CSV** | `wms_export_productividad_hora_csv` |

Procesos observados: `Recibir OC`, `Picking`, `Conteo ciclico`, `Finishing B2b Agrupado`, `Transferencia SKU` y `Ajuste diferencia contenidos - movimientos`.

### 3.10 `wms_productividad_detalle`: movimientos (requiere ADMIN)

| | |
|---|---|
| **Qué devuelve** | Cada movimiento de pallet (equivale a `indicadores/productividad/detalle`). Solo entradas positivas y sin conteos cíclicos. |
| **Filtros** | Los mismos que productividad por hora. |
| **Campos** | `palletMovId`, `fecha`, `entrada`, `salida`, `numeroPalletOrigen`, `numeroPalletDestino`, `ssoId`, `funcion`, `unidadesProductivas`, `tipo` |
| **CSV** | `wms_export_productividad_detalle_csv` |

Sirve para profundizar en una hora, usuario o proceso. **No reemplaza los totales por hora**: sumarlo no da lo mismo.

### 3.11 Exportaciones CSV (todas)

Respuesta observada:

```json
{
  "filename": "pedidos-<id>.csv",
  "rowCount": 14,
  "complete": true,
  "bytes": 3340,
  "sha256": "…",
  "expiresAt": "<+1 hora>",
  "resourceUri": "wms-export://pedidos/<id>.csv",
  "downloadUrl": "https://<host>/exports/<id>/pedidos.csv"
}
```

- Es un archivo completo, UTF-8, generado en una sola foto. No usa `limit` ni `offset`.
- **El link vence aproximadamente 1 hora después de generarse.**
- Con OAuth, la descarga exige iniciar sesión con **la misma cuenta** que generó el archivo.
- También queda disponible como recurso MCP (`resourceUri`).

---

## 4. Indicadores automáticos que se pueden construir

Leyenda de **actualización sugerida**: **TR** = cada 2 a 5 minutos · **F** = cada 15 a 30 minutos · **D** = diaria o al cierre del turno.

### 4.1 Pedidos y backlog

| # | Indicador | Fórmula | Fuente y filtros | Actualización |
|---|---|---|---|---|
| P1 | Pedidos activos | Cantidad de pedidos | `wms_pedidos` en modo `ALL_ACTIVE` | TR |
| P2 | Pedidos por estado, tipo y operador | Cantidad agrupada por `estado`, `tipoOrden` y `operadorLogistico` | `wms_pedidos` | TR |
| P3 | Unidades pendientes | Σ `pendingUnits` | `wms_pedidos` | TR |
| P4 | Antigüedad del backlog | Distribución de `ageDays` en rangos (0, 1, 2, 3 a 5 y más de 5 días). Pedidos y unidades por rango. | `wms_pedidos` | F |
| P5 | Pedidos fuera de plazo | Pedidos con `ageDays` mayor al umbral definido para cada `tipoOrden` (por ejemplo ECOM más de 1 día) | `wms_pedidos` | TR |
| P6 | Faltante al planificar (quiebre) | (Σ requested − Σ planned) / Σ requested | `wms_pedidos` | F |
| P7 | Faltante en picking | (Σ planned − Σ picked) / Σ planned, sobre pedidos ya pickeados | `wms_pedidos` en modo `ALL` | F |
| P8 | Cumplimiento por unidades (fill rate) | Σ picked / Σ requested | `wms_pedidos` en modo `ALL` | D |
| P9 | Pedidos completos | Pedidos con `pendingUnits = 0` / total | `wms_pedidos` | D |
| P10 | Pedidos que entran por día u hora | Cantidad por fecha de `createdAt` | `wms_pedidos` en modo `ALL` | F |
| P11 | Pendiente por tienda, cliente o transporte | Σ `pendingUnits` por línea, agrupado | `wms_pedidos` con `detail=true` | F |
| P12 | SKU más pedidos o con más pendiente | Ranking por SKU o master | `wms_pedidos` con `detail=true` | F |

### 4.2 Picking y finishing

| # | Indicador | Fórmula | Fuente | Actualización |
|---|---|---|---|---|
| K1 | Avance global de picking | Σ picked / Σ required (sumar numerador y denominador; no promediar porcentajes) | `wms_picking_progress` | TR |
| K2 | Avance global de finishing | Σ separated / Σ picked | `wms_picking_progress` | TR |
| K3 | Asignación de ubicaciones | Σ assigned / (Σ assigned + Σ unassigned) | `wms_picking_progress` | TR |
| K4 | Avance por piso | K1 y K2 agrupados por `piso` | `wms_picking_progress` | TR |
| K5 | Olas sin iniciar | Pickings en `PI_PENDIENTE` con picked = 0, y sus unidades | `wms_picking_progress` | TR |
| K6 | Olas antiguas | Pickings activos ordenados por `createdAt` (antigüedad en horas o días) | `wms_picking_progress` | TR |
| K7 | Unidades pendientes de picking y de finishing | Σ `pendingPickingUnits` y Σ `pendingFinishingUnits` | `wms_picking_progress` | TR |
| K8 | Avance por tipo de orden | K1 y K2 agrupados por `tipoOrden` | `wms_picking_progress` | TR |

### 4.3 Stock e inventario

| # | Indicador | Fórmula | Fuente y filtros | Actualización |
|---|---|---|---|---|
| S1 | Stock disponible y no disponible | Σ `units`, según `availability` D o ND | `wms_stock` | F |
| S2 | Stock por almacén y sector | Σ `units` agrupado | `wms_stock` | F |
| S3 | Stock envejecido | Σ `units` con `minAgeDays` = 90, 180 y 365. Porcentaje sobre el total. | `wms_stock` | D |
| S4 | Top SKU o master envejecidos | Ranking con `minAgeDays` y `detail=true` (usar `oldestEntryDate`) | `wms_stock` | D |
| S5 | Dispersión de un SKU | Cantidad de ubicaciones distintas por SKU | `wms_stock` con `detail=true` | D |
| S6 | Ubicaciones ocupadas | Cantidad de `ubicacionId` distintos con stock, por sector. Es un valor absoluto: no hay dato de capacidad. | `wms_stock` con `detail=true` | D |
| S7 | Stock distribuible | Σ `units` por almacén, master y SKU | `wms_distributable_stock` | F |
| S8 | Stock en una ubicación clave | Σ `units` filtrando, por ejemplo, `ubicacion=SP` | `wms_stock` | F |

### 4.4 Recepción (ingresos)

| # | Indicador | Fórmula | Fuente | Actualización |
|---|---|---|---|---|
| R1 | Unidades recibidas por día | Σ `unidades_originales` **deduplicando `or_producto_id`**, agrupado por `fecha_recepcion` | `wms_ingresos` | F |
| R2 | Recepción por origen o tipo | R1 agrupado por `tipo_orden`, `origen` (por ejemplo `DEV_TIENDA`) y `sucursal_nombre` | `wms_ingresos` | F |
| R3 | Recepción por proveedor | R1 agrupado por `proveedores` | `wms_ingresos` | D |
| R4 | Recepción por atributos de producto | R1 agrupado por `temporada`, `jerarquia`, `rango_edad`, `color` o `talle` | `wms_ingresos` | D |
| R5 | Recepción pendiente | Productos o etiquetas por estado (por ejemplo `OR_PENDIENTE`), con unidades y antigüedad desde `fecha_recepcion` | `wms_ingresos` | TR |
| R6 | Días de recepción a ingreso (dock-to-stock) | `fecha_ingreso` − `fecha_recepcion`, promedio y percentil 90 | `wms_ingresos` | D |
| R7 | Días de ingreso a aprobación | `fecha_aprobacion` − `fecha_ingreso` | `wms_ingresos` | D |
| R8 | Porcentaje guardado | Σ `unidades_guardadas` / Σ `unidades` | `wms_ingresos` | F |
| R9 | Diferencias de recepción | Productos con `unidades` distinto de `unidades_originales` | `wms_ingresos` | D |
| R10 | Bultos recibidos | Σ `total_paquetes` y cantidad de `etiqueta_caja` distintas por día | `wms_ingresos` | F |

### 4.5 Cargas iniciales

| # | Indicador | Fórmula | Fuente | Actualización |
|---|---|---|---|---|
| C1 | Avance de cada carga y etapa | Σ assigned / Σ requested × 100 | `wms_cargas_iniciales` agrupado por `CARGA` | F |
| C2 | Avance consolidado por etapa | C1 agrupado por `curva` | `wms_cargas_iniciales` agrupado por `ETAPA` | F |
| C3 | Unidades pendientes de asignar | Σ (requested − assigned) | `wms_cargas_iniciales` | F |
| C4 | Cobertura inmediata | Σ `separableUnits` / Σ `pendingAssignmentUnits` | `wms_carga_separables` (todas las páginas) | F |
| C5 | SKU sin stock en la carga | SKU con `availableUnits = 0` y pendiente mayor a 0. Lista y unidades. | `wms_carga_separables` | F |
| C6 | Pendiente sin cubrir | Σ `remainingPendingUnits` | `wms_carga_separables` | F |
| C7 | Stock total de los masters de la carga | Σ `units` | `wms_stock` con `cargaNumero` y `cargaCurva` | F |
| C8 | Productos de la carga con menor avance | Ranking por master o SKU de menor % de asignación | `wms_cargas_iniciales` agrupado por `MASTER` o `SKU` | F |

### 4.6 Productividad (requiere ADMIN)

| # | Indicador | Fórmula | Fuente | Actualización |
|---|---|---|---|---|
| U1 | Unidades por proceso | Σ `unidades` por `funcion` (día o turno) | `wms_productividad_hora` | F |
| U2 | Unidades por hora y operario | Σ `unidades` / cantidad de horas con actividad, por `ssoId` | `wms_productividad_hora` | F |
| U3 | Mapa de calor hora × proceso | Σ `unidades` por `hora` y `funcion` | `wms_productividad_hora` | F |
| U4 | Ranking de operarios | Σ `unidades` y `eventos` por `ssoId`, filtrable por proceso | `wms_productividad_hora` | F |
| U5 | Operarios activos por hora | Cantidad de `ssoId` distintos por `hora` | `wms_productividad_hora` | F |
| U6 | Unidades por evento | Σ `unidades` / Σ `eventos`, por proceso | `wms_productividad_hora` | D |
| U7 | Tiempo activo aproximado | Σ (`maxFecha` − `minFecha`) por operario y hora (es una aproximación) | `wms_productividad_hora` | D |
| U8 | Productividad por marca o grupo | U1 filtrando por `marcas` y `grupos` | `wms_productividad_hora` | D |
| U9 | Detalle de movimientos | Movimientos de una hora, operario o proceso elegidos | `wms_productividad_detalle` | Bajo demanda |

### 4.7 Indicadores que combinan dominios

| # | Indicador | Cálculo | Fuentes |
|---|---|---|---|
| X1 | Balance del día | Unidades recibidas (R1), unidades de pedidos nuevos (P10) y unidades pickeadas (U1, proceso Picking) | `ingresos`, `pedidos`, `productividad_hora` |
| X2 | Backlog contra capacidad | Unidades pendientes de picking (K7) / promedio de unidades por hora de picking (U2), en horas | `picking_progress`, `productividad_hora` |
| X3 | Cobertura del backlog con stock | Pendiente por SKU (P12) contra stock disponible por SKU (S1) | `pedidos` con detalle, `stock` |
| X4 | Envejecido contra demanda | SKU con stock de más de 180 días (S3) que no aparecen en pedidos activos (P12) | `stock`, `pedidos` con detalle |
| X5 | Cobertura de cargas con distribuible | Pendiente de la carga (C3) contra stock distribuible del almacén (S7), por SKU | `cargas_iniciales`, `distributable_stock` |

> X2 a X5 requieren muchas páginas de datos. Conviene calcularlos en el backend con caché, o a partir de exportaciones CSV programadas.

---

## 5. Módulos propuestos para la app HTML

| Módulo | Indicadores | Actualización |
|---|---|---|
| **Inicio / resumen operativo** | P1, P3, P5, K1, K2, R5, C2, U1 | TR |
| **Pedidos** | P1 a P12, con detalle por pedido (`wms_pedido_details`) | TR / F |
| **Picking** | K1 a K8, con detalle por ola | TR |
| **Stock** | S1 a S8, con búsqueda por SKU, master y ubicación | F / D |
| **Recepción** | R1 a R10 | F / D |
| **Cargas iniciales** | C1 a C8, con selector de carga y etapa | F |
| **Productividad** (solo ADMIN) | U1 a U9 | F |
| **Análisis cruzado** | X1 a X5 | D |
| **Descargas** | Botones a las 9 exportaciones CSV (el backend genera el archivo y entrega el link) | Bajo demanda |

---

## 6. Restricciones y reglas para el desarrollo

1. **Solo lectura.** No se puede asignar, reservar ni cambiar estados desde la app.
2. **Rangos de fecha:**
   - Pedidos y picking: hasta **3 meses**; `createdTo` excluido (para un día completo, usar la medianoche del día siguiente).
   - Productividad: hasta **31 días**; `desde` y `hasta` incluidos.
   - Ingresos: fechas `yyyy-MM-dd`, ambas incluidas.
3. **Paginación obligatoria** para totales (`mayHaveMore` y `nextOffset`).
4. **Porcentajes globales:** sumar numeradores y denominadores. Nunca promediar porcentajes.
5. **Valores de 0 a 100:** el conector ya devuelve los porcentajes en esa escala.
6. **Ingresos:** deduplicar por `or_producto_id` antes de sumar unidades.
7. **Cargas:** usar las unidades asignadas e ignorar los campos de separación. Cada etapa (`curva`) va por separado.
8. **Stock distribuible y cobertura:** son stock compartido. No sumarlo una vez por carga.
9. **Productividad:** requiere usuario ADMIN. No reemplazar los totales por hora con la suma del detalle.
10. **Limpieza de texto:** algunos campos vienen con espacios de relleno (`transporte`, `sub_jerarquia`, `sucursal_nombre`) o con comillas vacías (`proveedores = "''"`). Hay que aplicar trim y normalizar.
11. **Zona horaria:** los filtros van en hora local del WMS, pero `createdAt` llega con sufijo `+00:00`. **Validar** si es UTC real u hora local con la marca de UTC antes de mostrar horas o calcular antigüedades.
12. **CSV:** el link vence en 1 hora y está atado a la cuenta OAuth que lo generó.
13. **Seguridad:** el token OAuth vive solo en el backend. El HTML nunca lo recibe.

---

## 7. Información que el conector **no** entrega

| Falta | Impacto | Alternativa |
|---|---|---|
| Despacho, expedición y remitos de salida | No se puede medir lo despachado, el OTIF ni los tiempos de despacho | Otra fuente o un nuevo endpoint en el WMS |
| Fecha de cierre o finalización de pedido | No hay tiempo de ciclo del pedido ni lead time | Pedir un campo de fecha por estado |
| Fecha de inicio y fin de cada picking | No hay duración de las olas | Aproximar con `createdAt` y productividad |
| Compromiso de entrega (SLA) por pedido | El "fuera de plazo" depende de umbrales fijados por la app | Configurar umbrales por `tipoOrden` |
| Capacidad de ubicaciones y almacenes | No hay % de ocupación | Maestro de ubicaciones con capacidad |
| Catálogos: almacenes, sectores, estados, tipos de orden, transportes, sellers | Los filtros no se pueden armar dinámicamente y hay IDs sin nombre | Mantener catálogos en la app o pedir endpoints |
| Exactitud de inventario (diferencias de conteo cíclico) | Solo hay unidades contadas (productividad), no diferencias | Nuevo endpoint |
| Maestro de productos | Solo hay atributos del producto en ingresos | Usar los atributos de ingresos o un maestro externo |
| Valorización (costo o precio) | No hay indicadores en dinero | Fuente externa (ERP) |
| Eventos o webhooks | Todo debe ser polling | Backend con consultas programadas |
| Historia del stock | Solo hay foto actual | Guardar fotos periódicas en el backend para tener tendencias |

> **Tendencias:** para graficar la evolución de backlog, stock o avance a lo largo del tiempo, el backend tiene que **guardar fotos periódicas**. El conector solo devuelve el estado actual, salvo en ingresos, pedidos por `createdAt` y productividad, que sí tienen fecha.

---

## 8. Pendientes a validar con el equipo del WMS

1. URL del endpoint MCP y flujo OAuth para un cliente propio (backend).
2. `limit` máximo por página y límites de consultas.
3. Si `createdAt` está en UTC real u hora local.
4. Catálogo completo de estados de pedidos, pickings, cargas, productos y etiquetas de recepción.
5. Lista de almacenes, sectores y códigos de ubicación relevantes (por ejemplo `SP` y `SP2`).
6. Qué significa exactamente el filtro `pendingOnly` en cargas.
7. Si existen endpoints REST directos (opción C) y su política de CORS.
8. Qué usuario ADMIN usaría la app para productividad.
