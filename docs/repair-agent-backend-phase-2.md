# Fase 2 — Backend de consultas de reparaciones

**Fecha:** 2026-10-16
**Plan ejecutado:** `docs/superpowers/plans/2026-10-16-repair-backend.md`
**Registro de ejecución:** `.superpowers/sdd/2026-10-16-repair-backend/progress.md`

## 1. Objetivo de la fase

Transformar el backend de chat de inventario/ventas (Google Apps Script) en un backend de **consulta de estado de reparaciones, de solo lectura**, sin tocar el frontend ni realizar ningún despliegue. El backend de inventario (`?action=search`) queda intacto y sigue operativo; solo el flujo de chat (`?action=chat`) se reconecta, pasando de las herramientas de inventario a una única herramienta de reparaciones.

## 2. Archivos modificados y creados (Tareas 1-7)

Todo bajo `backend/apps-script/` (ignorado por git por decisión explícita y reafirmada del usuario — contiene un ID real de Google Sheet; por eso esta fase no generó commits ahí, solo este informe final en `docs/`).

**Creados:**
- `backend/apps-script/RepairLogic.gs` — lógica pura de mapeo de encabezados, autorización de estado, búsqueda por identidad y política de divulgación (Tareas 1-3, portado en Tarea 6).
- `backend/apps-script/RepairChatLogic.gs` — herramienta `consultar_reparacion`, system prompt y su ejecutor (Tarea 5, portado en Tarea 6).
- `backend/apps-script/tests-node/repairLogic.js` / `.test.js` (Tareas 1-3).
- `backend/apps-script/tests-node/repairChatLogic.js` / `.test.js` (Tarea 5).

**Modificados:**
- `backend/apps-script/Config.gs` — nuevas claves `REPAIR_SHEET_ID`/`REPAIR_SHEET_NAME`, constantes `KcChat_REPAIR_HEADER_FIELDS_`/`KcChat_REPAIR_AUTHORIZED_STATES_`, nueva `KcChat_assertRepairConfigured_()` (deliberadamente separada de `KcChat_assertConfigured_()`, tras un hallazgo de revisión — ver sección 9).
- `backend/apps-script/Inventory.gs` — nuevas `KcChat_readRepairRawRows_()` y `KcChat_readAuthorizedRepairRows_()`.
- `backend/apps-script/AiChatLogic.gs` — bucle genérico `KcChat_runAiChatLoop_` generalizado con `deps.executeTool` y `deps.resultKey` (Tarea 4), de forma retrocompatible.
- `backend/apps-script/OpenRouter.gs` — nueva `KcChat_repairCallTool_`; `KcChat_handleChatRequest_` ahora apunta al flujo de reparaciones; `KcChat_callOpenRouterOnce_` ahora declara `tools: [KcChat_CONSULTAR_REPARACION_TOOL_]` en la llamada real a OpenRouter (ver hallazgo Crítico, sección 9).
- `backend/apps-script/Tests.gs` — 42 casos portados desde los módulos Node, más `KcChat_testRepairSheetIsNeverModified` (integración real, no ejecutable en esta sesión).
- `backend/apps-script/tests-node/aiChatLogic.js` / `.test.js` — inyección de `executeTool`/`resultKey` más fix de `resultKey` en las rutas de error (`safeFailure_`).

**No tocados (confirmado, ver sección 7):** `Code.gs`, `appsscript.json`, todo el código de inventario/ventas (`Inventory.gs` conserva sus funciones originales intactas; solo se le añadieron las nuevas), el modelo, el endpoint y la lógica de reintento de OpenRouter.

## 3. Funciones nuevas/modificadas (resumen por capa)

**`RepairLogic.gs`:** `KcChat_buildHeaderIndexMap_`, `KcChat_mapRepairRow_`, `KcChat_formatRepairDate_`, `KcChat_classifyImageCell_`, `KcChat_normalizePhone_`, `KcChat_canonicalAuthorizedState_`, `KcChat_cloneRepairRowWithState_`, `KcChat_filterAuthorizedRepairs_`, `KcChat_searchRepairsByIdentity_`, `KcChat_disambiguateRepairsByInvoice_`, `KcChat_buildDisclosedRecord_`, `KcChat_lookupRepair_`.

**`Inventory.gs`:** `KcChat_readRepairRawRows_()` (lectura cruda, nunca llamable desde fuera del archivo por convención), `KcChat_readAuthorizedRepairRows_()` (única función exportable — compone lectura + filtro de estado autorizado en un solo paso, para que el filtro nunca pueda olvidarse).

**`AiChatLogic.gs`:** `KcChat_runAiChatLoop_` ahora acepta `deps.executeTool` (default `KcChat_executeOneTool_`) y `deps.resultKey` (default `"products"`), incluidas ambas rutas de error (no solo la de éxito — ver hallazgo de Tarea 4).

**`RepairChatLogic.gs`:** `KcChat_TOOL_CONSULTAR_REPARACION_`, `KcChat_REPAIR_AUTHORIZED_TOOLS_` (lista blanca de un solo elemento), `KcChat_CONSULTAR_REPARACION_TOOL_` (tool-def OpenAI-style: `phone` requerido, `fullName`/`numeroFactura` opcionales), `KcChat_REPAIR_SYSTEM_PROMPT_`, `KcChat_executeOneRepairTool_`.

**`OpenRouter.gs`:** `KcChat_repairCallTool_` (nuevo, paralelo a `KcChat_aiCallTool_` que ya existía), `KcChat_handleChatRequest_` modificado para conectar ambas capas nuevas, `KcChat_callOpenRouterOnce_` con el arreglo del tools-array (sección 9).

## 4. Mapa de 18 campos y encabezados reales

Mapeo siempre por texto de encabezado normalizado (nunca por letra/índice de columna fijo), definido en `KcChat_REPAIR_HEADER_FIELDS_` (`Config.gs`):

| key | header real en la hoja |
|---|---|
| `fullName` | Full Name |
| `phoneMobile` | Phone Mobile |
| `marcaModelo` | ¿Marca/Modelo? |
| `estado` | Estado de la Reparacion: |
| `pantallaRota` | ¿Pantalla Rota? |
| `equipoEncendido` | ¿Equipo Encendido? |
| `descripcion` | Descripcion: |
| `precio` | Precio: |
| `condicionesEquipo` | ¿Condiciones del equipo? |
| `numeroFactura` | Numero de Factura: |
| `fechaEntrada` | Fecha de Entrada |
| `fechaModificacion` | Fecha de Modificacion |
| `imagenFrontal` | Imagen Frontal |
| `imagenTrasera` | Imagen Trasera |
| `imagenFrontalListo` | Imagen frontal de equipo listo. |
| `imagenTraseraListo` | Imagen trasera de equipo listo. |
| `comentario` | Comentario: |
| `facturaPaga` | Factura Paga |

La columna `V.` (columna G de la hoja) y cualquier otra columna fuera de esta lista nunca se lee ni se expone. Un encabezado faltante o duplicado hace fallar `KcChat_buildHeaderIndexMap_` con `{ok:false, reason, field}` — el error que llega al cliente nombra solo el campo (p. ej. "falta el encabezado esperado para Numero de Factura:"), nunca el `SHEET_ID` real, la fila de encabezados cruda, ni contenido de celda.

## 5. Regla de autorización de estado y las 5 etiquetas canónicas

**Decisión final, autoritativa:** exactamente **5** estados autorizados, tal como los estableció la auditoría de Fase 1:

1. `Recibido.`
2. `Está en Proceso de Reparación.`
3. `Equipo para devolver sin solucion.`
4. `Equipo listo para retirar.`
5. `Equipo Entregado.`

**Nota sobre el "cinco vs. seis":** el texto de la especificación de Fase 2 del usuario, en un momento, listó un sexto estado (`Pieza en Transito.`), y un borrador anterior de este mismo plan lo incluyó tras una pregunta aclaratoria. El usuario retiró explícitamente ese sexto estado en su aprobación final del plan, confirmando que no existe autorización explícita para añadir ningún estado más allá de los 5 originales de la auditoría de Fase 1. Esta es la lista final y vigente — el código (`KcChat_REPAIR_AUTHORIZED_STATES_` en `Config.gs`) implementa solo estos 5.

Cualquier fila con estado vacío, desconocido o no autorizado se descarta antes de que cualquier otra cosa la vea (búsqueda, desambiguación, la herramienta de IA, logs) — esto ocurre en `KcChat_filterAuthorizedRepairs_`, que es la única vía hacia `KcChat_readAuthorizedRepairRows_()` (la única función de lectura de reparaciones que el resto del sistema puede llamar). La comparación se normaliza con `KcChat_normalizeText_` (tolera mayúsculas/tildes/espacios distintos) pero sin volverse tan laxa que una cadena no autorizada colapse sobre una autorizada — verificado con casos específicos de "casi-coincidencia" (p. ej. "Recibido en tienda." no colapsa con "Recibido."). La etiqueta que se muestra siempre es el literal canónico de la lista, nunca el valor crudo de la hoja.

## 6. Manejo de múltiples coincidencias

`KcChat_lookupRepair_` sigue este algoritmo:
- Sin teléfono utilizable → `{status:"missing_identifier"}`.
- 0 coincidencias → `{status:"not_found"}`.
- Más de 1 coincidencia:
  - Si se proporcionó `numeroFactura`, se desambigua filtrando por ese número; si el resultado no es exactamente 1, → `{status:"not_found"}` (fail-closed; confirmado explícitamente por revisión como un comportamiento correcto y seguro, no un defecto).
  - Si no se proporcionó `numeroFactura` → `{status:"ambiguous"}`, un objeto con **únicamente** esa clave — nunca el conteo de coincidencias ni ningún dato de los registros candidatos.
- Exactamente 1 coincidencia → `{status:"ok", record: {estado, marcaModelo}}`.

El asistente, al recibir `ambiguous`, pide el número de factura sin dar ninguna pista de cuántas reparaciones hay.

## 7. Decisión de privacidad/divulgación y sus dos capas de aplicación

**Decisión:** el teléfono (y, si se da, el nombre) son usados **únicamente como clave de búsqueda**, nunca como prueba de identidad. El usuario rechazó explícitamente, tras revisar el primer borrador de este plan, que "nombre + teléfono" bastara como verificación de identidad suficiente ("no se debe considerar el nombre y el teléfono como prueba suficiente de identidad").

**Consecuencia:** cuando la búsqueda resuelve a exactamente un registro, lo que se entrega — a la herramienta, al modelo y de ahí al cliente — se reduce **siempre** a `{estado, marcaModelo}`. Nunca `phoneMobile`, `fullName`, `descripcion`, `precio`, `condicionesEquipo`, `numeroFactura`, `fechaEntrada`, `fechaModificacion`, ningún campo de imagen, `comentario` ni `facturaPaga`.

Esto **no** se implementó una sola vez y se confió en que se mantuviera — se construyó, y se puso a prueba, como **dos capas independientes**:

1. **`KcChat_buildDisclosedRecord_`** (`RepairLogic.gs`) — dentro de `KcChat_lookupRepair_`, construye el objeto devuelto leyendo solo `.estado`/`.marcaModelo` del registro completo.
2. **`KcChat_executeOneRepairTool_`** (`RepairChatLogic.gs`) — nunca confía en lo que devuelve `deps.callTool`: reconstruye el objeto divulgado leyendo de nuevo solo `.estado`/`.marcaModelo` del resultado, sin spread ni reenvío del objeto recibido, y además pasa cada valor por `KcChat_safeDisclosedField_` (reduce a `null` cualquier valor que no sea un string, por si una capa anterior defectuosa entregara un objeto en el campo correcto).

**Esto no es una garantía solo documentada — está respaldada por hallazgos reales de revisión que probaron precisamente este mecanismo:**

- **Tarea 3 (capa 1), hallazgo Importante:** el fixture de la prueba de "aislamiento por substring" de teléfonos tenía una afirmación falsa en su comentario/reporte — los dos números usados en la prueba no compartían en realidad el substring que se alegaba, por lo que la prueba no demostraba que el emparejamiento fuera estrictamente exacto (una implementación con `includes()` también la habría pasado). El código en sí era correcto (verificado a mano); solo la prueba era débil. Se corrigió en la ronda de fixes 1/5 y se verificó independientemente.
- **Tarea 5 (capa 2), hallazgo Crítico:** cualquier valor de `status` distinto de `ok`/`ambiguous` pasaba sin control hacia el contenido devuelto al modelo (`repairChatLogic.js:183-187` en ese momento) — un defecto de la capa 1 (p. ej. un objeto `{count, candidates}`, o un string como `"ambiguous_3"`) se habría filtrado a través de esta supuesta capa independiente. Corregido: solo `"not_found"`/`"missing_identifier"` se reenvían tal cual; cualquier otro valor se trata como error genérico.
- **Tarea 5, hallazgo Importante #1:** los valores divulgados de `estado`/`marcaModelo` no se verificaban por tipo, solo por nombre de clave — un valor no-string (p. ej. un objeto) de un bug de capa 1 habría pasado sin cambios. Corregido con `KcChat_safeDisclosedField_` (coerción a string-o-null).
- **Tarea 5, hallazgo Importante #2:** un `record` nulo o ausente en el resultado lanzaba una excepción no capturada, rompiendo todo el turno de chat (sin filtrar datos, pero sin resiliencia). Corregido con una guarda explícita que devuelve un error genérico.

Los tres hallazgos de Tarea 5 se abordaron en una ronda de fixes y se re-verificaron de forma independiente (trazado a mano de entradas adversariales, pruebas de regresión correspondientes confirmadas). Esta historia — no un resultado perfecto a la primera — es la evidencia de que la política de divulgación está realmente aplicada, no solo declarada.

**Riesgo residual:** incluso con esta divulgación reducida, cualquier persona que conozca o adivine el número de teléfono registrado de un cliente puede conocer el estado de su reparación y el modelo del equipo (nada más sensible). Expandir lo que se divulga por este canal requeriría primero un mecanismo real de verificación (p. ej. un OTP enviado a ese teléfono por WhatsApp/SMS) — explícitamente fuera del alcance de esta fase.

## 8. Formato real de los campos de imagen — sin resolver

`KcChat_classifyImageCell_` solo marca `available:true` cuando el valor de la celda es una URL `https://` directa que termina en una extensión de imagen común (`png`/`jpg`/`jpeg`/`gif`/`webp`), opcionalmente con query string. Cualquier otra cosa — celda vacía, enlace de compartición de Google Drive, ID de archivo de Drive, texto plano, remanente de fórmula — se clasifica como `available:false`. Esto es una **lista blanca conservadora, no una confirmación del formato real**: esta sesión no tiene acceso a la hoja de Google en vivo para inspeccionar los valores reales de las celdas Q/R/S/T (`imagenFrontal`/`imagenTrasera`/`imagenFrontalListo`/`imagenTraseraListo`). Si el formato real resulta ser, por ejemplo, un enlace de Drive, esta clasificación los marcará todos como no disponibles — correcto desde el punto de vista de seguridad (nunca asumir que un enlace de Drive es público), pero probablemente significa que, hoy, ningún campo de imagen se mostrará como disponible hasta que alguien con acceso a la hoja confirme el formato real y, si es necesario, se ajuste el clasificador o el proceso de carga de imágenes.

## 9. Hallazgos Críticos/Importantes reales y decisiones del controlador (historia completa, sin sanitizar)

Esta fase pasó por múltiples rondas de revisión con hallazgos reales, no solo trámite:

- **Tarea 1** — Importante: claves de campo mal escritas (`encendido`/`condiciones` en vez de `equipoEncendido`/`condicionesEquipo`, divergiendo de las restricciones globales). 1 ronda de fix, verificado.
- **Tarea 3 (la tarea más crítica de privacidad)** — Importante: fixture de prueba de aislamiento por substring con afirmación falsa (ver sección 7). 1 ronda de fix, verificado. 4 hallazgos Menores diferidos (sin riesgo de fuga entre clientes): bypass de la puerta de ambigüedad con `numeroFactura` vacío/"?"; `searchRepairsByIdentity_` sin guarda si se llamara fuera de `lookupRepair`; faltan pruebas negativas aisladas para un par de funciones internas; `normalizePhone` conserva el código de país (gap de usabilidad, no de privacidad).
- **Tarea 4** — Importante: las rutas de error de `safeFailure_` seguían devolviendo `products:[]` fijo sin importar `resultKey`, dando una forma de contrato inconsistente (`{repairs:[...]}` en éxito pero `{products:[]}` sin clave `repairs` en error). 1 ronda de fix, verificado con pruebas para ambas mitades.
- **Tarea 5 (segunda capa de privacidad)** — 1 Crítico + 2 Importantes, los tres detallados en la sección 7. 1 ronda de fix, los tres verificados de forma independiente.
- **Tarea 6 (el puente Node → `.gs`)** — Crítico, autodetectado por el implementador y confirmado por el controlador mediante una **ruling de defecto del plan**: la llamada HTTP real a OpenRouter (`KcChat_callOpenRouterOnce_`) nunca declaraba `consultar_reparacion` como herramienta disponible para el modelo — seguía con el array fijo de las dos herramientas de inventario. El modelo real, en producción, jamás habría podido invocar la herramienta de reparaciones aunque el system prompt le dijera que existía. El plan original instruía dejar esa función "completamente sin tocar" bajo la premisa de que era plomería de red genérica — una premisa incorrecta, porque el array de herramientas estaba fijo (hardcoded) dentro de esa misma función. Resolución: se modificó el array `tools:` al único tool de reparaciones, siendo `KcChat_handleChatRequest_` el único llamador real de esa función hoy. 1 ronda de fix, verificada de forma independiente por un segundo revisor que además ejecutó el `.gs` portado contra los módulos Node de referencia en un sandbox de Node `vm`, confirmando salidas idénticas en mapeo de encabezados, filtrado de estado, las 6 ramas de `lookupRepair`, ambas capas de divulgación, el flujo end-to-end con el wiring real, constantes de configuración byte-idénticas, el punto único de entrada, y ausencia de escrituras (grep ampliado incluyendo `DriveApp`/`insertSheet`/etc.).
- **Tarea 7** — Importante: `KcChat_assertConfigured_` acoplaba configuración de inventario y de reparaciones, contradiciendo el propio comentario de `Config.gs` de que son "completamente independientes" — sin beneficio real, ya que `KcChat_readRepairRawRows_` ya tenía su propia guarda independiente. 1 ronda de fix: se revirtió `KcChat_assertConfigured_` a solo-inventario y se creó `KcChat_assertRepairConfigured_` separada, verificado.

Ninguno de estos hallazgos quedó sin resolver cuando era Crítico o Importante; los Menores diferidos están listados arriba y en el ledger (`progress.md`) para que una fase futura decida si se abordan.

## 10. Nota de permisos de Google Sheets

Abrir una segunda hoja por ID (`REPAIR_SHEET_ID`) **no requiere ningún alcance (scope) OAuth nuevo** de Apps Script más allá de lo que `SpreadsheetApp` ya requería para la hoja de inventario existente — confirmado de forma concreta: `appsscript.json` tiene diff cero contra el snapshot más antiguo disponible (ver sección 12, verificación #3).

Lo que **sí** se requiere, y que está fuera del repositorio y fuera de la visibilidad de esta sesión: que la cuenta de Google con la que se ejecuta el script (hoy: "Ejecutar como: Yo", es decir, el propietario del script) tenga al menos acceso de lector compartido sobre esa hoja específica, en la configuración de compartición propia de Google. Nadie en esta sesión puede confirmar si ese acceso ya está otorgado.

## 11. Nota de logging

Ninguna llamada a `Logger.log` en el código nuevo o modificado registra algún campo de un registro de reparación ni la clave de API de OpenRouter. Esto se verificó con un grep dirigido, no se asume por construcción únicamente:

```
grep -niE "Logger\.log" backend/apps-script/RepairLogic.gs backend/apps-script/RepairChatLogic.gs backend/apps-script/Inventory.gs backend/apps-script/OpenRouter.gs
```

Resultado: **cero coincidencias** en los cuatro archivos. La clave de OpenRouter se sigue leyendo exclusivamente vía `PropertiesService.getScriptProperties().getProperty("OPENROUTER_API_KEY")` en `Config.gs`, nunca impresa ni registrada en ningún archivo tocado por esta fase.

## 12. Barrido final de verificación (ejecutado en esta sesión)

**1) Suite de pruebas Node, conteos exactos:**

```
matchLogic.test.js      → 30 pasaron, 0 fallaron
aiChatLogic.test.js     → 16 pasaron, 0 fallaron
repairLogic.test.js     → 40 pasaron, 0 fallaron
repairChatLogic.test.js → 10 pasaron, 0 fallaron
```

Nota: `aiChatLogic.test.js` cerró en 16 (no 15) porque la ronda de fix de Tarea 4 añadió una prueba adicional para cubrir ambas mitades del fix de `resultKey` en `safeFailure_` (`repairs:[]` sin clave `products`); `repairLogic.test.js` cerró en 40 (no 39) por la prueba añadida en la ronda de fix de Tarea 3 (aislamiento real de prefijo/sufijo de teléfono). Ambos conteos coinciden con lo registrado en el ledger tras cada ronda de fix — no son discrepancias, son el estado final correcto.

**2) Alcance — `git status --short`:**

```
?? docs/
```

Único resultado: el directorio `docs/` sin seguimiento (contiene el plan ya existente en `docs/superpowers/plans/` y, ahora, este informe). Cero líneas para `frontend/`, `.github/`, o cualquier configuración de despliegue.

**3) Diff de `Code.gs`/`appsscript.json` contra el snapshot más antiguo disponible (`task-6-before/`, el primero que capturó ambos archivos):**

Ambos diffs retornan vacío (exit code 0, sin salida) — **byte-idénticos**. Enrutamiento, rate limiting y declaración de alcances OAuth sin cambios.

**4) Listado de `backend/apps-script/` — conjunto de archivos:**

```
AiChatLogic.gs (modificado)      Config.gs (modificado)
Code.gs (sin tocar)               Inventory.gs (modificado)
MatchLogic.gs (sin tocar)         OpenRouter.gs (modificado)
README.md (sin tocar)             RepairChatLogic.gs (NUEVO)
RepairLogic.gs (NUEVO)            Tests.gs (modificado)
appsscript.json (sin tocar)       consolidated/KcChat_Consolidado.gs (sin tocar — ver sección 13)
consolidated/appsscript.json (sin tocar)
tests-node/aiChatLogic.js (modificado)       tests-node/aiChatLogic.test.js (modificado)
tests-node/matchLogic.js (sin tocar)         tests-node/matchLogic.test.js (sin tocar)
tests-node/repairChatLogic.js (NUEVO)        tests-node/repairChatLogic.test.js (NUEVO)
tests-node/repairLogic.js (NUEVO)            tests-node/repairLogic.test.js (NUEVO)
```

Exactamente el conjunto esperado — ningún archivo nuevo inesperado.

**5) Grep final de operaciones de escritura, en todo `backend/apps-script/*.gs` y `consolidated/*.gs`:**

```
grep -niE "setValue|setValues|appendRow|deleteRow|\.clear\(|insertRow|removeRow" backend/apps-script/*.gs backend/apps-script/consolidated/*.gs
```

Única coincidencia: dentro de `Inventory.gs` (líneas 3-4, 82) y `consolidated/KcChat_Consolidado.gs` (líneas 320-321) — en ambos casos, texto de **comentario** que describe la garantía de solo-lectura, nunca código ejecutable. Idéntico en naturaleza al resultado de la auditoría de Fase 1. Ningún archivo nuevo de esta fase (`RepairLogic.gs`, `RepairChatLogic.gs`, `Tests.gs`) contiene ninguna coincidencia.

## 13. Pruebas ejecutadas vs. pruebas que no pudieron ejecutarse

**Ejecutadas en esta sesión (Node, resultados reales arriba):** las 4 suites completas — 96 pruebas en total (30+16+40+10), 0 fallos.

**No ejecutadas, honestamente:** todo lo que vive en `Tests.gs` (el runner de pruebas de Apps Script, incluyendo las 42 pruebas portadas desde los módulos Node en la Tarea 7 y la nueva `KcChat_testRepairSheetIsNeverModified`) **nunca se ejecutó** en ninguna tarea de esta fase, porque este entorno no tiene runtime de Apps Script ni una cuenta de Google con acceso real a ninguna de las dos hojas. Esto se declaró explícitamente desde la Tarea 7 (puerto estático únicamente) y se reafirma aquí: `KcChat_runAllTests` necesita ejecutarse manualmente, dentro del editor de Apps Script, por alguien con acceso a la cuenta de Google del proyecto, antes de que estas pruebas cuenten como "pasaron" en sentido real. Lo mismo aplica a cualquier llamada real a OpenRouter (la ronda de fixes verificó el wiring por lectura de código e inspección en un sandbox Node `vm`, no con una llamada HTTP real).

## 14. Pasos necesarios antes de conectar el frontend (Fase 3)

1. El frontend debe leer el nuevo campo `repairs` en la respuesta de `?action=chat` en lugar de `products` (el backend ya no devuelve `products` desde este endpoint — ver sección 2/3).
2. Construir un nuevo componente de tarjeta/vista para mostrar `{estado, marcaModelo}` — el formato de tarjeta de producto existente no aplica a una reparación.
3. Alguien con acceso real a Google Sheets debe:
   - Confirmar que los 18 encabezados reales de la hoja `kuki CompuCell Base Datos` coinciden exactamente con los listados en la sección 4 (mayúsculas, tildes, signos de puntuación incluidos — el mapeo es sensible a eso, aunque tolera espacios/mayúsculas distintas vía normalización).
   - Inspeccionar el formato real de las celdas de imagen (Q/R/S/T) y decidir si `KcChat_classifyImageCell_` necesita ajustarse (sección 8).
   - Confirmar que la cuenta de ejecución del script tiene acceso de lector sobre `REPAIR_SHEET_ID` (sección 10).
   - Ejecutar `KcChat_runAllTests` dentro del editor de Apps Script y confirmar que las 42+ pruebas portadas (incluyendo la integración real `KcChat_testRepairSheetIsNeverModified`) pasan contra la hoja real.

## 15. Limitaciones conocidas que el propietario del proyecto debe conocer

- **`backend/apps-script/consolidated/KcChat_Consolidado.gs` ("Opción B", archivo único) está ahora desactualizado** respecto a los cambios de esta fase — no contiene ninguna de las funciones ni el wiring de reparaciones, y conserva su propio `KcChat_assertConfigured_` sin el desacople de la Tarea 7. Regenerarlo es una tarea futura, explícitamente fuera del alcance de esta fase (el plan de Fase 2 solo listó los 6 archivos individuales de "Opción A"). No usar "Opción B" para el flujo de reparaciones hasta que se regenere.
- **La semántica de la bandera `whatsapp` para resultados de reparación** (`not_found`/`ambiguous`/`missing_identifier` la ponen todas en `true`) es un valor heredado del bucle genérico de inventario, no una decisión de diseño explícita para el flujo de reparaciones. La Fase 3/el frontend debería revisar si ese comportamiento tiene sentido para una consulta de reparación (p. ej., si "ambiguous" realmente debería sugerir WhatsApp, o si debería pedir el número de factura dentro del mismo chat).

## 16. Conclusión de la verificación final

Los 5 chequeos de la barrida de verificación final (sección 12) pasaron sin excepciones. El alcance de esta fase se mantuvo dentro de `backend/apps-script/` (fuera de git por decisión del usuario) más este único archivo bajo `docs/`. `Code.gs` y `appsscript.json` permanecen byte-idénticos al estado anterior a la Tarea 1. No se introdujo ninguna operación de escritura sobre ninguna hoja. El conjunto de archivos es exactamente el esperado. La política de divulgación de dos capas está implementada y fue puesta a prueba por hallazgos de revisión reales (Críticos e Importantes), no solo declarada.
