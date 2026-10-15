# Panel de Tesorería — Documentación técnica

Para quien tenga que mantener o ampliar la aplicación. Actualizado el 2 de octubre de 2026.

## 1. Qué es
Aplicación web de una sola página para la tesorería de la empresa: clientes, proveedores, obras, facturas emitidas y recibidas, hoja mensual, resumen, avisos de vencimientos, provisión a 12 meses, gastos fijos con seguimiento de pagos e impresión del sobre de envío a obra (formato DL, 220 × 110 mm).

## 2. Arquitectura
```
Navegador (index.html en GitHub Pages) --POST JSON + clave--> Apps Script (Code.gs) <--> Hoja de cálculo de Google
```
- **`index.html`** (GitHub Pages): toda la interfaz en un único archivo, sin framework. La librería de fechas (dayjs) va incluida dentro del archivo.
- **`Code.gs`** (Apps Script, ejecutado con la cuenta del propietario): "puerta de datos". Solo lee y escribe filas en la hoja. No sirve la página.
- **`appsscript.json`**: manifiesto. Permiso `spreadsheets`; despliegue "Ejecutar como: yo", acceso "Cualquier persona".
- **Hoja de cálculo de Google**: almacén. Su identificador está en `SHEET_ID` de `Code.gs`.

El proyecto de Apps Script puede estar dentro de un documento de Google Drive: **si se borra ese documento, se borra el script**. No eliminarlo.

## 3. Seguridad
- Las peticiones llevan una **clave** que se compara con la constante `CLAVE` de `Code.gs` (mínimo 16 caracteres; si no se ha cambiado, el servicio rechaza todo). Una clave incorrecta provoca una espera de 1,5 s para frenar la fuerza bruta.
- La dirección del servicio (`.../exec`) y la clave **no están en el código público**: cada usuario las escribe una vez en su dispositivo y se guardan en `localStorage` de ese navegador. El botón «Salir» las borra.
- Quien conozca la dirección **y** la clave puede leer y escribir. Para revocar el acceso a alguien: cambiar `CLAVE` y crear una nueva versión del despliegue.
- No hay identificación por persona ni registro de quién hizo cada cambio.
- Alternativa más estricta (por cuenta de Google de cada persona): requeriría Google Cloud (inicio de sesión OAuth). Se descartó porque pedía forma de pago.

## 4. Modelo de datos
Cada **colección** es una pestaña de la hoja con **dos columnas**: `A` = identificador, `B` = registro en JSON. Sin cabecera. Las pestañas se crean solas.
Colecciones (lista `COLS`, duplicada en `Code.gs` y en el bloque de conexión de `index.html`): `clients`, `suppliers`, `works`, `invoicesOut`, `invoicesIn`, `fixedExpenses`, `fixedExpensePayments`, `workBudgetItems`, `priceListItems`, `invoiceOutPayments`, `invoiceInPayments`.

- `works`: `name`, `clientId`, `address`, `foremanName`, `foremanPhone` (alimentan el sobre).
- `fixedExpenses`: `concept`, `category`, `amount`, `frequency` (`mensual`, `trimestral`, `anual`, `quincenal`, `semanal`), `nextDate`, `autoIvaCalc`.
- `fixedExpensePayments`: `expenseId`, `month` (`AAAA-MM`), `amount`, `date`, `notes`. Admite pagos parciales.

No editar las pestañas a mano: el JSON debe seguir siendo válido y los identificadores enlazan colecciones entre sí.

## 5. Protocolo del servicio
Todas las llamadas son `POST` con cuerpo JSON en texto plano (evita la comprobación previa de CORS) y devuelven JSON:

| `action` | Datos enviados | Respuesta |
|---|---|---|
| `getAll` | `key` | `{ok, data:{coleccion:[{id,json}]}}` |
| `add` | `key, c, id, json` | `{ok}` |
| `update` | `key, c, id, patch` (se fusiona con el JSON existente) | `{ok}` |
| `delete` | `key, c, id` | `{ok}` |

Errores: `{error: "mensaje"}`. Las escrituras usan `LockService` (20 s). `GET` devuelve un mensaje de servicio activo.

## 6. Cliente (`index.html`)
La aplicación se escribió contra una API de base de datos tipo Firestore (`collection().add/get/onSnapshot`, `doc().update/delete`). Un bloque `<script>` inicial implementa esa API sobre `fetch` con caché en memoria. Recarga todo al abrir, tras cada escritura y cada 30 s (si la pestaña está visible).
Modos: **conectado** (hoja de Google) y **de prueba** (datos de ejemplo solo en ese navegador). En modo conectado se desactiva la carga de datos de ejemplo (`window.__NO_SEED`).

Gastos fijos: el previsto de cada mes depende de la frecuencia (en la semanal se cuentan los días reales); el atraso se acumula desde el primer pago registrado; estados Pagado / Parcial / Pendiente.
### Integridad de datos (comprobada con un simulador de dos usuarios)
- `add` es idempotente en el servidor: si el identificador ya existe, no se vuelve a insertar. El cliente reintenta hasta 2 veces solo si falla la red, y descarta un segundo alta idéntica en menos de 4 s (doble clic).
- `update` envía al servidor **solo los campos que han cambiado** respecto a lo que ve el usuario, y el servidor los fusiona con el registro actual. Dos personas que editan campos distintos del mismo registro no se pisan; si editan el mismo campo, gana la última escritura.
- `delete` y `update` localizan la fila por identificador dentro de un bloqueo, nunca por número de fila; editar un registro ya borrado no lo recrea.
- Las escrituras de un mismo navegador se ejecutan en cola y en orden.
- Tras guardar, la caché se actualiza y se relee en segundo plano; un fallo de relectura no se notifica como fallo de guardado (evita reintentos manuales duplicados).
- En modo conectado no se guarda ningún dato en el navegador (solo la dirección y la clave). El modo de prueba muestra una barra roja.

### Sobres, cartas y logo
- Constante `ENV` (en `index.html`): medidas del sobre en mm (por defecto sobre americano nº 10, 241 × 105 mm) y posición del bloque de destinatario (`left` = más a la derecha, `top` = más abajo). Los sobres de la empresa llevan el remitente preimpreso arriba a la izquierda, por eso el texto va en la zona central-derecha.
- Efectos y remesas → «Pendiente de pagar a proveedores»: las líneas con forma de pago Pagaré tienen los botones **Sobre** (nombre, contacto y dirección postal del proveedor) y **Carta** (A4, texto de envío del pagaré). Requiere el campo `address` del proveedor (dirección postal, varias líneas separadas por salto de línea).
- La carta usa los datos de la empresa (nombre, ciudad, firmante) que se piden la primera vez y se guardan en el navegador (`localStorage`, clave `panelTesoreriaEmpresa`); se cambian con «Datos de la empresa», abajo a la derecha.
- Logo: archivo `logo.png` en la misma carpeta que `index.html` (GitHub). Aparece en el menú lateral y en la cabecera de las cartas. Si no existe, no se muestra nada ni da error.

Sobre: «Imprimir sobre» usa un iframe y `window.print()`; «Descargar sobre» genera un `.html` que se imprime al abrirlo.

## 7. Puesta en marcha y actualizaciones
1. Hoja de Google nueva; copiar su identificador a `SHEET_ID`; inventar una clave larga en `CLAVE`.
2. Apps Script: pegar `Code.gs` y `appsscript.json`. Implementar → Nueva implementación → Aplicación web → Ejecutar como: **yo**; Acceso: **Cualquier persona**. Copiar la dirección `/exec`.
3. Subir `index.html` a un repositorio de GitHub con Pages activado.
4. Abrir la página, escribir dirección y clave.
Cada cambio en `Code.gs` exige **Implementar → Administrar implementaciones → editar → Nueva versión** (la dirección no cambia).

## 8. Cómo ampliar
- **Nueva colección:** añadirla a `COLS` en `Code.gs` y en `index.html`.
- **Nuevo campo:** basta con añadirlo al formulario en `index.html`.
- **Copia de seguridad:** copiar la hoja de cálculo periódicamente (Archivo → Hacer una copia).

## 9. Limitaciones conocidas
- Sin control de conflictos por registro: gana la última escritura.
- El sondeo de 30 s no es tiempo real.
- Cada pestaña guarda JSON, no columnas legibles.
- Sin importación de facturas arrastrando archivos (dependía de un servicio externo).
- Límites de cuota de Apps Script y de tamaño de celda (~50.000 caracteres por registro).
- Se probó con simuladores del servicio de Google; conviene una prueba completa con datos ficticios tras cada despliegue.

## 10. Historial de decisiones
- Se descartó Google Cloud (OAuth) por pedir forma de pago.
- Se intentó servir la página completa desde Apps Script, pero **Apps Script cortaba el archivo** (por contener etiquetas `</body></html>` dentro de textos del programa y, después, por tamaño), y la página no arrancaba. Si se vuelve a intentar: no escribir etiquetas HTML literales en los scripts y trocear el archivo en partes pequeñas con `include()`.
- Se adoptó el modelo mixto: página en GitHub Pages y Apps Script solo como puerta de datos con clave (el mismo patrón que la web de caja contable).

## 11. Sobres, cartas y logo
- **Sobre de obra** (ficha de obra) y **sobre de proveedor** (Efectos y remesas → Pendiente de pagar a proveedores, solo líneas con forma de pago Pagaré): tamaño del sobre y posición del texto en la constante `ENV` de `index.html` (`w`, `h`, `left`, `top`, en mm; por defecto sobre americano nº 10, 241 × 105 mm, texto a 105 mm del borde izquierdo y a 40 mm de arriba, porque el remite va preimpreso arriba a la izquierda). El navegador solo sugiere el tamaño de papel: hay que elegir «Sobre» en la impresora una vez.
- **Carta de envío de pagaré** (A4, función `pagareLetterHtml`): usa el proveedor (`name`, `contactName`, `address`), el importe y vencimiento de la línea y el número y fecha de la factura. El nombre de la empresa, la ciudad y el firmante se piden la primera vez y se guardan en el navegador (clave `panelTesoreriaEmpresa`); se cambian desde «Datos de la empresa», abajo a la derecha.
- `suppliers` tiene un campo nuevo, `address` (dirección postal, varias líneas separadas por salto de línea).
- **Logo**: archivo `logo.png` en la misma carpeta que `index.html` (GitHub). Se muestra en la pantalla de acceso, el menú lateral y la cabecera de la carta; si el archivo no existe, no aparece nada roto.

## 12. Fechas, importes y casillas (formatos)
- **Formato interno:** fechas como texto `AAAA-MM-DD`, importes como número JSON, casillas como `true`/`false`. Todo el JSON vive en una celda de texto, así que Google Sheets nunca lo interpreta como fecha ni número. Las pestañas nuevas se crean con las columnas A:B en formato texto plano.
- **Visualización:** `fmtDate` muestra `DD/MM/AAAA`; `fmtMoney` muestra importes con 2 decimales en formato `es-ES` (los miles de 4 cifras salen sin punto: `4521,30 €`).
- **Cálculos de fecha** con dayjs en hora local (sin pasar por UTC): no hay desfases de un día. Probado en Madrid (incluido el cambio de hora de marzo), UTC, Los Ángeles y Auckland, cerca de la medianoche.
- **Defensa al leer (`normRecord` en el bloque de conexión):** si un valor llega en otro formato, se interpreta al cargar sin tocar la hoja: fechas `dd/mm/aaaa` (día primero; si el mes no puede pasar de 12 se entiende mes/día), `d-m-aa`, `AAAA-MM-DDTHH:MM...` y números de serie de Excel (de 5 cifras); importes como texto (`4.521,30`, `1234,5`, `1234.56`; ojo: `1.234` se lee como 1234); casillas `TRUE/VERDADERO/sí/no/...`.
- Lo que no se reconoce (`mañana`, `2026-02-30`, `mil euros`) se deja tal cual y aparece un aviso amarillo abajo a la izquierda con la lista de datos para revisar.
- Campos vigilados: fechas `date, dueDate, paidDate, collectedDate, nextDate, remittanceDate, emissionDate, issueDate`; números `amount, budget, paymentTermDays, installmentsRemaining, quantity, unitPrice, price, base, iva, ivaAmount, ivaRate`; casillas `collected, paid, remesado, autoIvaCalc, received`. Si se añaden campos nuevos de ese tipo, hay que sumarlos a las listas `DATE_F`, `NUM_F` y `BOOL_F`.

## 13. Bancos, saldos, gastos puntuales e IVA (fase 1 de la ampliación)
- **Colecciones nuevas** (añadidas a `COLS` en `Code.gs` y en `index.html`): `bankAccounts` (`name`, `archived`), `bankBalances` (`bankId`, `date`, `amount`, `note`; cada anotación es una fila nueva, así se conserva el histórico) y `oneOffExpenses` (`concept`, `amount`, `dueDate`, `bank`, `paid`, `paidDate`, `paidBank`, `notes`).
- **El efectivo** es una cuenta fija con identificador `cash`; no tiene fila en `bankAccounts`.
- **Campos nuevos:** facturas emitidas `emissionDate`, `collectedBank`, `ivaExempt`; facturas recibidas `emissionDate`, `paidBank`, `ivaExempt`; `fixedExpenses.bank`; `fixedExpensePayments.bank`. Los pagos parciales (`invoiceOutPayments`/`invoiceInPayments`) admiten los mismos campos `emissionDate` y `collectedBank`/`paidBank`.
- **Saldo estimado** de una cuenta (`bankBalanceNow`) = último saldo anotado (el de fecha más reciente; en empate, el último de la hoja) + cobros − pagos con fecha **posterior** a la del saldo. Los movimientos del mismo día del saldo se consideran ya incluidos. Un movimiento «Efectivo» sin banco indicado va a la cuenta `cash`; un pago de gasto fijo sin banco usa el banco habitual del gasto.
- **Importes:** los importes de las facturas incluyen IVA. El IVA estimado es importe × 21/121, se excluyen las facturas con `ivaExempt` y el IVA del trimestre es (emitidas − recibidas) × 21/121.
- **Formato de dinero** con punto de millar (`4.521,30 €`).

## 14. Provisión (fase 2 de la ampliación)
- **Pantalla `Provisión`** (`renderProvision`, `provisionItems`, `ivaQuarterItems`): horizonte seleccionable (1, 2, 3, 6 o 12 meses; por defecto 1, el mes actual) y cuatro cifras: *Disponible hoy* (suma de los saldos estimados de la sección Bancos, efectivo incluido), *Cobros asegurados*, *Obligaciones* y *Saldo previsto al final* (= disponible + asegurados − obligaciones).
- **Cobros asegurados:** solo líneas no cobradas con forma de pago Pagaré, Confirming o Cheque, en su fecha de vencimiento. Lo pendiente por otros medios no se suma; se muestra aparte como «No contado».
- **Obligaciones:** facturas de proveedores pendientes (todas las formas de pago), gastos fijos del periodo no pagados (los pagos registrados en `fixedExpensePayments` se descuentan y los atrasos acumulados pasan a «Vencido»), gastos puntuales no pagados e **IVA trimestral**.
- **IVA trimestral:** vence el día 20 del mes siguiente al trimestre (30 de enero para el cuarto). Importe = max(0, (emitidas − recibidas del trimestre, sin las marcadas «sin IVA») × 21/121). Solo se incluyen vencimientos desde hoy; el del trimestre en curso se marca como estimado con lo registrado hasta hoy. Los gastos fijos de categoría impuesto con `autoIvaCalc` se ignoran en esta pantalla para no contar el IVA dos veces.
- **Columna «Vencido»:** efectos y pagos con fecha anterior a hoy que siguen sin cobrar o pagar.
- **Tabla mensual** con ingresos en verde y gastos en rojo, saldo del periodo y saldo acumulado; tabla **Por banco** (saldo estimado frente a obligaciones asignadas a esa cuenta, con aviso ⚠ si no llega) y **detalle** de cada movimiento previsto.
- Se mantienen el seguimiento de pagos de gastos fijos del mes y la tabla de gastos fijos configurados.

## 15. Hoja Bancos como desglose de Provisión (parte 1 de los cambios de octubre)
- **Hoja Bancos**, tarjetas: Saldos (con columna *Principal*), Remesas, Traspasos entre cuentas, Gastos puntuales y Gastos fijos, préstamos, impuestos y nóminas configurados (movida desde Provisión).
- **Banco principal** (`mainBankId`): el cuenta marcada con la casilla `main` en `bankAccounts`; si no hay ninguna marcada, la de mayor último saldo anotado (sin contar el efectivo). Regla de asignación de banco cuando no se indica: forma de pago *Efectivo* → caja; *Transferencia* o sin forma de pago → banco indicado o el principal; *Pagaré/Confirming/Cheque* → banco indicado o el banco de la remesa (`remittanceBankId`); gastos fijos y puntuales sin banco → el principal.
- **Traspasos:** colección nueva `bankTransfers` (`fromId`, `toId`, `date`, `amount`, `note`). El saldo estimado suma o resta los traspasos con fecha posterior a la del último saldo anotado.
- **Remesas:** «Remesar seleccionados» (hoja Efectos y remesas) exige elegir un banco de la lista y guarda en cada efecto `remittanceDate`, `remittanceId`, `remittanceBankId` y `remittanceBank`. La tarjeta Remesas agrupa por `remittanceId`; las remesas antiguas sin identificador se agrupan por fecha y banco escrito. Se corrigió que las líneas sin plazos parciales no arrastraban la fecha de remesa.
- **Préstamos:** el campo `installmentsRemaining` se sustituye por `lastDate` (fecha del último plazo); los plazos restantes se calculan solos. Si un préstamo solo tiene el dato antiguo, se sigue usando.

## 16. Provisión definitiva, Negociar y arreglos (parte 2 de los cambios de octubre)
- **Formularios:** un desplegable sin valor elegido muestra ahora «— Elige —». Antes, la primera opción (p. ej. «Efectivo») parecía elegida sin registrarse y el registro se guardaba sin ese dato (traspasos sin destino). Además, el formulario avisa de los campos obligatorios que faltan en vez de guardar incompleto. Los traspasos incompletos se ignoran en los saldos y se marcan en rojo en su lista. Una cuenta sin saldo anotado que tenga traspasos parte de 0 desde el día anterior al primero.
- **Provisión** (`renderProvision`): horizontes `7 días` (columnas por día, desde hoy), 1, 2, 3, 6 y 12 meses. Filas: *Saldo bancario* y *Cobros asegurados* (verde oscuro), *Cobros anticipados* (si los hay), una fila por concepto de obligación (rojo), línea divisoria y *SALDO RESULTANTE* en negrita (verde o rojo). Columna *Total*. Sin fila de suma de obligaciones.
- **Saldo bancario por periodo:** arranca con el disponible de hoy (sin lo anticipado aún no absorbido) y cada periodo parte del resultado del anterior.
- **Cobros anticipados** (efectos con `negotiated`, `collected` y `collectedBank`, cobrados después del último saldo anotado de ese banco): aparecen en el periodo de su fecha; los pagos de los periodos siguientes se descuentan **primero** de ese colchón, sin restarlos de nuevo en el saldo resultante; al cambiar de mes lo que sobra pasa a la línea *Saldo bancario*. Caso de prueba: 13.000 € en el banco, 8.000 € negociados el 7/2 y 6.000 € de pagos el 20/2 → semana del ingreso: 13.000 / 8.000; semana del pago: 13.000 / 2.000 (resultante 15.000); marzo: 15.000 / —. Un saldo anotado a mano ya incluye lo anticipado y remesado antes de su fecha.
- **Vista de 7 días:** tarjeta *Ingreso necesario*: por cuenta, pagos asignados (incluidos los vencidos) menos saldo estimado; no cuenta los cobros que puedan entrar esa semana.
- **Notas:** cobros pendientes por transferencia no contados, y total de pagos por transferencia asignados al banco principal.
- **Gastos fijos** del mes elegido: tarjeta *Gastos fijos* (fecha de vencimiento del mes, concepto, atraso, previsto = cuota + atraso, pagado, pendiente, estado, botón Pagar) y tarjeta *Pagos y cobros registrados* (pagos de gastos fijos, pagos a proveedores, gastos puntuales pagados y cobros de clientes, incluidos los negociados).
- **Negociar** (hoja Efectos y remesas, pestaña Pendientes): se elige efecto, banco y fecha; queda cobrado por el **nominal** con `negotiated: true`. Los intereses y gastos del banco no se calculan: se añaden después como gasto puntual para que cuenten en el balance.
- Campos nuevos: `negotiated` (casilla en la factura emitida, visible si está cobrada y es un efecto).

## 17. Control de efectos (fusión de Hoja mensual y Efectos y remesas)
- **Hoja única `Control de efectos`** (`renderControl`), con tres botones: **CLIENTES**, **PROVEEDORES** y **TOTAL EFECTOS**. Se retiran del menú «Hoja mensual» y «Efectos y remesas» (funciones `renderSheet`, `renderEffects`, `renderEffectTable` y `renderGroupedSection` eliminadas).
- **CLIENTES / PROVEEDORES:** navegación por meses (por fecha de vencimiento) y solo efectos (Pagaré, Confirming, Cheque; las transferencias no salen aquí, se ven en Provisión). Columnas, en este orden: fecha de emisión del efecto (`emissionDate`, «—» si no consta), nombre, nº de factura, vencimiento, importe, banco emisor (`issuingBank`), y después tipo, estado y casilla Cobrado/Pagado. Total del mes, cobrado/pagado y pendiente. En móvil se muestra como tarjetas.
- **Marcar cobrado o pagado:** abre una ventana que pregunta el **banco** (por defecto el de la remesa o el principal) y la fecha (`askBank`). Guarda `collectedBank`/`paidBank` y la fecha; el saldo estimado del banco cambia solo, porque se calcula con los movimientos posteriores al último saldo anotado (no se escribe en el histórico de saldos). Quitar la marca lo deshace.
- **Remesar seleccionados** y **Negociar un efecto** están en CLIENTES; **Sobre** y **Carta** de pagaré, en PROVEEDORES. El nombre abre la factura para editarla.
- **TOTAL EFECTOS:** tabla por mes de vencimiento (desde el mes actual) con filas por tipo y totales de cobros (verde) y pagos (rojo), solo efectos no vencidos y sin cobrar/pagar; y sección aparte **Vencidos sin cobrar / pagar** con el resumen por tipo y la lista.
- **Facturas recibidas:** campo nuevo `issuingBank` («banco emisor», nuestro banco que emite el efecto).
- **Móvil:** el diseño de móvil se activa también en pantallas táctiles anchas (modo «sitio de escritorio») y se evitó que un desplegable con texto largo ensanchara la página (`html, body { overflow-x: hidden }`, selectores con `max-width: 100%`).

## 18. Medidas y precios en las partidas de presupuesto
- **Medidas en centímetros** en cada partida (`workBudgetItems`): `material`, `finish` (acabado), `unit` (`m2`, `ml` o `ud`), `pieces` (nº de piezas), `length` (largo), `width` (ancho), `thickness` (grosor), `polishedFaces` (0-2), `drips` (goterones 0-2) y `unitPrice`. `quantity` se calcula y se guarda: **m²** = piezas × largo × ancho / 10.000; **ml** = piezas × largo / 100 (el ancho no cuenta); **ud** = piezas. Se redondea a 2 decimales y el importe es cantidad × precio unitario (`measureTotal`, `budgetAmount`). Ejemplo: 12 baldosas de 60×30 a 35 €/m² → 2,16 m² → 75,60 €.
- **Partidas antiguas** (sin medidas, con unidades como `ML` o `kg`): conservan su cantidad guardada y su importe.
- **Formato:** `fmtQty` muestra un máximo de 2 decimales sin ceros sobrantes (`2,16 m²`, `2 m²`, `12 ml`).
- **Lectura del nombre:** `60x30x3` = largo × ancho × grosor; `35x2` en una pieza por ml = ancho × grosor, y por m² = largo × ancho (`parseDims`).
- **Glosario** (`materialGlossary`, tarjeta en la hoja Obras): `canonical`, `synonyms` (separados por comas), `defaultUnit`, `defaultThickness` y `standardSizes`. Reconoce la pieza por cualquiera de sus nombres (sin tildes ni mayúsculas, singular o plural) y aplica la unidad y el grosor estándar. Una medida que no esté en `standardSizes` avisa de que suele cobrarse por unidad. El botón «Cargar glosario inicial» crea Baldosa (m², 3 cm), Tabica (2 cm), Peldaño (3 cm) y Alféizar (ml); es un punto de partida editable.
- **Precios de proveedor** (`priceListItems`): ahora con `finish`, `thickness`, `polishedFaces`, `drips`, `unit`, `price` y `offerDate`. Al rellenar una partida se sugieren los precios que encajan (misma pieza, acabado, grosor, caras pulidas y goterones), con proveedor y fecha, más recientes primero.
- **Sobre de proveedor:** botón «Imprimir sobre» en la ficha del proveedor (nombre, atención de y dirección postal) y botones Sobre y Carta en los efectos Pagaré y Cheque de Control de efectos > PROVEEDORES.
- **Arreglo de datos:** los registros nuevos solo guardan los campos propios de su ficha (antes arrastraban valores por defecto ajenos como `status` o `category`).

## 19. Balance financiero (`renderBalance`, `balModel`)
- **Hoja nueva** «Balance financiero» (tras Provisión). Periodos: año en curso, últimos 12 meses o todo. Sin librerías externas: las gráficas son SVG propios (barras horizontales, barras mensuales con línea de resultado y burbujas) con información al pasar el ratón y salto a la ficha del cliente al pulsar.
- **Base de cálculo:** importes **sin IVA** (importe / 1,21 salvo facturas marcadas «sin IVA»), por fecha de factura. Beneficio bruto = ventas − compras. Resultado operativo = bruto − gastos fijos (categorías gasto y nómina, sin los impuestos automáticos) − gastos puntuales; no incluye préstamos ni impuestos.
- **Margen por cliente:** ventas del cliente − compras de las obras del cliente (`invoicesIn.workId` → `works.clientId`). Solo se calcula si hay compras imputadas; si no, el panel lo indica y muestra la facturación. **Margen por material:** partidas con `unitCost` (coste unitario de compra); agrupadas por pieza del glosario o por nombre.
- **Tiempo de cobro:** media ponderada por importe de los días entre la fecha de la factura y la de cobro. **Retraso:** días de cobro posteriores al vencimiento.
- **Riesgo por cliente (propuesta pendiente de validar):** 40 % retraso medio (0 días = 0; ≥ 30 = 100), 25 % % de lo pendiente ya vencido, 20 % peso en el pendiente total (≥ 30 % = 100), 15 % forma de pago (efectos = 0, transferencia = 100). Bajo < 35, medio 35-65, alto > 65. Sin historial de cobros, el retraso no cuenta y se reajustan los pesos.
- **Estimación (6 meses):** media de ventas y compras de los últimos 6 meses completos con datos (sin IVA) y gastos fijos y puntuales programados; dos deslizadores ±30 % para simular escenarios.
- **Puntos débiles:** reglas con umbrales orientativos (liquidez a 3 meses, dependencia de cliente > 30 % y proveedor > 25 %, cobro medio > 60 días, cobro vencido > 25 %, pagos vencidos, cobros sin vencimiento, compras sin obra, gastos > 25 % de la facturación, margen bruto < 15 %, obras sin presupuesto, sin gastos configurados).
- **Informe imprimible** (`balanceReportHtml`): A4 con resumen ejecutivo, gráficas, clientes, materiales, avisos con recomendaciones, calidad de datos, contexto de mercado con fuentes (constante `MARKET`, consultada el 6/10/2026) y oportunidades (`OPORT`, marcadas como hipótesis). Los datos de mercado son texto fijo en el código: hay que revisarlos y actualizarlos de vez en cuando.
- **Campo nuevo** en las partidas: `unitCost`.

## 20. Buscador, códigos, Resumen y previsión con plazos pactados
- **Buscador** (`setupListSearch`) en Clientes, Proveedores y Obras: busca en cualquier parte del texto, sin distinguir mayúsculas ni tildes, con varias palabras en cualquier orden (todas deben aparecer). Clientes y proveedores: nombre, código, CIF, contacto, teléfono y email; obras: nombre, cliente, dirección, notas, encargado y estado. Un código exacto sale el primero. La búsqueda se conserva al volver a la lista.
- **Código** (`code`) de cliente y proveedor: los 3 últimos dígitos de la cuenta contable (430… / 400…). Columna «Cód.» en las listas y campo en la ficha. **Dirección postal** del cliente (`address`): se guarda, pero no se muestra en la ficha.
- **Resumen:** en el ordenador, tarjetas en 3 columnas (clientes arriba, proveedores abajo) y más grandes; el móvil mantiene su diseño. Los avisos de 15 días ya no tienen tope (lista con desplazamiento); los vencidos van en un desplegable aparte, y los cobros o pagos **sin fecha de vencimiento no salen como «vence hoy»**: se cuentan en una nota.
- **Previsión a 6 meses según plazos pactados** (`expectedFlows`, `expectedCardHtml`): todos los cobros y pagos pendientes en su fecha esperada, **si todos cumplen**, más gastos fijos, puntuales e IVA trimestral. Fecha esperada de una línea: su vencimiento; si no tiene, fecha de factura + plazo de pago del cliente o proveedor (`paymentTermDays`); si tampoco, **fecha de factura + 120 días (`PROV_TERM`), marcada como provisional**. Lo provisional se dibuja rayado y se avisa de su importe; en las fichas el plazo sin indicar aparece como «120 días (provisional, sin revisar)». Columna «Vencido» para lo atrasado. Es distinta de Provisión, que solo cuenta lo asegurado y el dinero disponible.

## 21. Formas de pago Tarjeta y Domiciliado
- `PAYMENT_METHODS` ahora es: Transferencia, Domiciliado, Tarjeta, Pagaré, Confirming, Cheque y Efectivo. `BANK_METHODS` agrupa las que se mueven por el banco (Transferencia, Domiciliado y Tarjeta): sin banco indicado van al banco principal, igual que la transferencia; Efectivo sigue yendo a la caja y los efectos al banco de su remesa.
- **Provisión:** los cobros por Tarjeta o Domiciliado siguen sin contar como «cobros asegurados» (solo los efectos). La nota de pagos por banco incluye ahora transferencia, domiciliación y tarjeta.
- **Riesgo por cliente** (propuesta pendiente de validar): forma de pago con efectos y tarjeta = 0 y transferencia = 100 (el domiciliado no se usa con clientes).
- En el volcado desde A3, los valores permitidos de `paymentMethod` / `method` son: Transferencia | Domiciliado | Tarjeta | Pagaré | Confirming | Cheque | Efectivo.

## 22. Domiciliado solo en pagos, confirming, resaltados e IVA fraccionado
- **Domiciliado** solo se ofrece en facturas recibidas, fichas de proveedor y gastos fijos (`CLIENT_METHODS` lo excluye en clientes y facturas emitidas). Los pagos domiciliados cuentan siempre como obligaciones ciertas en Provisión, Resumen y Balance.
- **Confirming:** no se remesa (no tiene casilla de remesar); entra solo al banco al vencimiento y se puede **negociar**. Estado: «Entra solo al vencimiento».
- **Resaltado en Control de efectos** (clase `hl-yellow` / `hl-fuchsia`): CLIENTES, amarillo = pagaré o cheque remesado o cobrado y confirming cobrado; fucsia = efecto negociado (tiene prioridad). PROVEEDORES, amarillo = efecto pagado por banco (pagado con banco distinto de la caja). Leyenda sobre la tabla.
- **IVA fraccionado** (`openIvaSplit`, `ivaSplitCreate`): botón «Fraccionar IVA» en Provisión y en la tarjeta de gastos fijos de Bancos. Pide trimestre (importe estimado propuesto), nº de plazos, fecha del primer pago y banco, y crea **un gasto fijo mensual** de categoría `impuesto`, forma de pago `Domiciliado`, `lastDate` = último plazo y `ivaQuarter` (por ejemplo `2026-3T`). Mientras exista ese gasto fijo, `ivaQuarterItems` no proyecta el pago único de ese trimestre; al borrarlo, vuelve a proyectarse. Un plazo = total / nº de plazos redondeado a céntimos (la diferencia de redondeo queda en las notas).
- Los gastos fijos tienen ahora campo opcional `paymentMethod`, y `lastDate` vale para préstamos e impuestos.

## 22. Imprimir en Control de efectos y Provisión al mes actual (15/10/2026)
- **Botón «🖨 Imprimir»** arriba a la derecha de Control de efectos (`ctlPrintHtml`). Imprime la pestaña que se está viendo: en CLIENTES y PROVEEDORES, el mes elegido (A4 vertical: fecha de emisión, nombre, nº de factura, vencimiento, importe, banco emisor, tipo y estado, con total del mes, cobrado o pagado y pendiente); en TOTAL EFECTOS, las dos tablas (A4 apaisado). Sin casillas ni botones. La cabecera usa el nombre de «Datos de la empresa» o, si no está, «Hermanos Vizcaíno, S.C.A.». Usa `printDoc` (iframe + `window.print()`), igual que los sobres.
- **Provisión:** el horizonte por defecto pasa a **1 mes** (mes actual): `let provisionHorizon = 1`.
