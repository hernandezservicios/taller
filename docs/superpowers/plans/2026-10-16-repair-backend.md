# Repair Query Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform the existing inventory-sales chat backend (Google Apps Script) into a read-only repair-status lookup backend, without touching the frontend or deploying anything.

**Architecture:** Add a new pure-logic layer (`RepairLogic.gs`) for header mapping, status authorization, and identity search, mirrored first as plain Node modules under `tests-node/` (the project's existing TDD convention: pure JS verified in Node, then copied verbatim into `.gs`). Add one new read-only Sheets accessor in `Inventory.gs`. Generalize the existing generic tool-calling loop (`KcChat_runAiChatLoop_` in `AiChatLogic.gs`) with two small backward-compatible injection points (`deps.executeTool`, `deps.resultKey`) so a new `RepairChatLogic.gs` can reuse it instead of duplicating it. Wire `OpenRouter.gs`'s existing entry point to the repair tool instead of the inventory tools. Leave `Code.gs`, rate limiting, the OpenRouter network layer, and all inventory code untouched and in place.

**Tech Stack:** Google Apps Script (V8 runtime), plain Node.js for pure-logic tests (no test framework / no `package.json` in this repo — same convention as the existing `tests-node/*.test.js` files, which use hand-rolled `assert`-style helpers and are run with `node file.test.js`).

**Spec:** The user's two pasted messages titled "FASE 2 — IMPLEMENTACIÓN DEL BACKEND DE CONSULTAS DE REPARACIONES" and the Phase 1 audit answer in this conversation, plus two clarifications already resolved with the user:
- Authorized states: **5**, exactly as the Phase 1 audit established — `Recibido.` / `Está en Proceso de Reparación.` / `Equipo para devolver sin solucion.` / `Equipo listo para retirar.` / `Equipo Entregado.`. (The Phase 2 spec's own list briefly added a sixth, `Pieza en Transito.`; an earlier draft of this plan used 6 after a clarifying question, but the user has since explicitly withdrawn that and confirmed there is no explicit authorization to add it — final answer: 5, no `Pieza en Transito.`, no invented equivalences.)
- Identity verification: phone number is the **primary** query the assistant asks for (the one registered when the repair was created) and is used only as a **lookup key**, never as proof of identity — the user explicitly re-confirmed, after seeing the first draft of this plan, that "no se debe considerar el nombre y el teléfono como prueba suficiente de identidad" and that personal data must not be exposed without adequate verification. Resolution: an exact phone match (optionally narrowed by invoice number when ambiguous) is enough to run the search and confirm a record exists, but the record handed back is reduced to `estado` + `marcaModelo` only — see the Disclosure Policy constraint below. Expanding this requires a real verification mechanism, out of scope here.

## Process Adaptation: no git for `backend/apps-script/`

`.gitignore` excludes `/backend/` entirely, by the user's explicit, reaffirmed decision (it contains a real Google Sheet ID and the user does not want it in git history at all, not even locally, not even temporarily). This means every file Tasks 1-7 touch is invisible to `git` — `git add`/`git commit`/`git diff` on those paths are no-ops or errors. Wherever a task below says "Commit" for a file under `backend/apps-script/`, that step does not apply: the implementer saves the file to disk (Write/Edit already do this regardless of git tracking) and reports its path — no `git add`/`git commit`. The controller cannot generate a git-based review package for these tasks either; instead, before dispatching each task's implementer, the controller snapshots the current content of every file the task's brief names (absent = new file) into `<workspace>/task-N-before/`, and after the implementer reports, builds a plain-text diff (`diff -u` per file, or the full content for a new file) as that task's review package, handed to the reviewer in place of a git diff. Task 8's `docs/repair-agent-backend-phase-2.md` is the one exception — `docs/` is not gitignored, so that file DOES use a normal `git add`/`git commit`, and Task 8's own verification step adapts accordingly (see Task 8, Step 2).

## Global Constraints

- **Intentional duplication, not a DRY violation:** this project already duplicates pure logic between a plain-Node file (`tests-node/matchLogic.js` → `MatchLogic.gs`, `tests-node/aiChatLogic.js` → `AiChatLogic.gs`) because Google Apps Script cannot `require()` a Node module — there is no shared-module mechanism across the two runtimes. Tasks 1-3 and 5 build `tests-node/repairLogic.js`/`repairChatLogic.js`; Task 6 ports them into `RepairLogic.gs`/`RepairChatLogic.gs` by copying the logic (renaming to the `KcChat_`-prefixed convention and wiring real Sheets/OpenRouter calls in place of test mocks). This copy is the established, deliberate pattern for this codebase, not an oversight — do not flag it as a DRY violation in any task review.

- Only modify files under `backend/apps-script/` (including `backend/apps-script/tests-node/`) and create `docs/repair-agent-backend-phase-2.md`. Do not touch `frontend/`, `.github/`, or any other path.
- Do not deploy, publish, or change the active Apps Script deployment or GitHub Pages config.
- Do not delete any existing file. Do not remove or weaken any existing test. `node matchLogic.test.js` must still report `30 pasaron, 0 fallaron` and `node aiChatLogic.test.js` must still report `12 pasaron, 0 fallaron` after every change.
- Keep `OPENROUTER_API_KEY` read exactly as today (`PropertiesService.getScriptProperties().getProperty(...)` in `Config.gs`). Never print, log, or hardcode it.
- Keep the model `deepseek/deepseek-v4-flash`, the OpenRouter endpoint, and the retry/error-handling logic in `OpenRouter.gs` exactly as today.
- Keep `Code.gs` (routing, rate limiting) completely unmodified.
- New Sheet: ID `1BYdv9Bq28CJhV8vJsFr-nmMrkgIbz3Qe4ZenSaODm_s`, tab `kuki CompuCell Base Datos`. Read-only. Never call any Sheets write method (`setValue`, `setValues`, `appendRow`, `deleteRow`, `.clear(`, `insertRow`, `removeRow`) anywhere in new or modified code.
- Column mapping is by normalized header text, never by fixed column letter/index. Required fields, in this exact output order: `fullName` (Full Name), `phoneMobile` (Phone Mobile), `marcaModelo` (¿Marca/Modelo?), `estado` (Estado de la Reparacion:), `pantallaRota` (¿Pantalla Rota?), `equipoEncendido` (¿Equipo Encendido?), `descripcion` (Descripcion:), `precio` (Precio:), `condicionesEquipo` (¿Condiciones del equipo?), `numeroFactura` (Numero de Factura:), `fechaEntrada` (Fecha de Entrada), `fechaModificacion` (Fecha de Modificacion), `imagenFrontal` (Imagen Frontal), `imagenTrasera` (Imagen Trasera), `imagenFrontalListo` (Imagen frontal de equipo listo.), `imagenTraseraListo` (Imagen trasera de equipo listo.), `comentario` (Comentario:), `facturaPaga` (Factura Paga). Column G (`V.`) and any other column must never be read or exposed.
- The 5 authorized states above are the only ones a repair record may carry once returned by `KcChat_readAuthorizedRepairRows_()`. Any row with an empty, unknown, or unauthorized state is dropped before anything else sees it (search, disambiguation, the AI tool, logs). The literal label shown is always the canonical string from the list above, never the raw sheet value.
- Exact not-found message, verbatim: `No se encontró una reparación consultable con los datos proporcionados.`
- Never expose, to the model or in any tool JSON content: how many records matched, or any data belonging to a record that wasn't the single resolved match.
- **Disclosure policy (revised per explicit user requirement — phone/name are search keys, never proof of identity):** when `lookupRepair` resolves to exactly one record, the `record` it returns contains **only** `estado` (the canonical label) and `marcaModelo` — never `phoneMobile`, `fullName`, `descripcion`, `precio`, `condicionesEquipo`, `numeroFactura`, `fechaEntrada`, `fechaModificacion`, any image field, `comentario`, or `facturaPaga`. This is enforced twice: once inside `lookupRepair` (`RepairLogic.gs`/Node mirror) and again, independently, inside `KcChat_executeOneRepairTool_` (`RepairChatLogic.gs`/Node mirror), which must rebuild `{estado, marcaModelo}` itself from whatever `deps.callTool` returned rather than trust it verbatim — so a defect in one layer can't leak the rest of the record. The system prompt must tell the model that any other detail (price, invoice, photos, comments) requires the customer to contact a technician directly; it must never read those fields from the tool result because they will not be there. Expanding this disclosure set is explicitly out of scope for this phase and requires a real verification mechanism (e.g., an OTP sent to the registered phone) first.
- No sales/inventory/discount/promotion tools may be reachable from the live `action=chat` path after this change. The existing inventory files and functions stay in the repo untouched (still reachable via `action=search` and their own tests) — they are just no longer wired into the chat tool loop.
- Image fields (Q/R/S/T): never invent a URL, never assume a Drive link is public, never request new Drive scopes. Only a value that is an exact `https://` URL ending in a common image extension counts as `available`; everything else (empty, Drive share link, file ID, plain text, formula remnant) is `available:false`.

## Review Focus

- A duplicate or missing header must fail with a generic, field-named error (e.g. "falta el encabezado esperado para Numero de Factura:") — never with the real `SHEET_ID`, the raw header row, or any cell content. Task 6 tests this explicitly.
- An ambiguous match (phone matches 2+ repairs) must never leak the count or any partial record — only a flag telling the caller to ask for the invoice number. Task 3 and Task 5 both test this independently (pure layer and chat-loop layer).
- Phone matching must be exact-normalized-digits, never substring/partial — a shorter or longer number that merely contains the query must not match (unlike the existing inventory product search, which intentionally does partial matches). Task 3 tests two distinct phone numbers that share a long common substring and asserts no cross-match.
- Status comparison normalization (reusing `KcChat_normalizeText_`, which strips accents/case/punctuation/spaces) must not become so loose that an unauthorized status string accidentally collapses onto an authorized one. Task 2 tests a handful of near-miss unauthorized strings (e.g. "Recibido" without the period is fine — it's meant to match — but "Cancelado." or an empty string must not).
- The image-cell classifier must reject a realistic Google Drive share URL (`https://drive.google.com/file/d/...`) as `available:false`, not just reject empty cells — a classifier that only checks "is it a non-empty string" would wrongly expose an unverified Drive link. Task 1 tests this with a real-shaped Drive URL string.
- A single-match "ok" result must never carry `phoneMobile`, `fullName`, `precio`, `numeroFactura`, `comentario`, or any image field — only `estado`/`marcaModelo`. Because this is enforced in two independent places (Task 3's `lookupRepair` and Task 5's `KcChat_executeOneRepairTool_`), each task tests it from a mock that deliberately includes the forbidden fields in its input, asserting they never reach the output — a test that only checks the *happy* shape wouldn't catch a layer silently forwarding extra fields.

---

### Task 1: Pure header mapping, date/price/image field parsing (Node)

**Files:**
- Create: `backend/apps-script/tests-node/repairLogic.js`
- Test: `backend/apps-script/tests-node/repairLogic.test.js`

**Interfaces:**
- Consumes: `normalizeText`, `parseNumericCell` via `require("./matchLogic")` — reuse, do not duplicate (DRY; `matchLogic.js` already exports both).
- Produces (used by Task 2 and Task 3, same file): `REPAIR_HEADER_FIELDS` (ordered array of `{key, header}` in the 18-field order from Global Constraints, with the real spreadsheet column index — 0-based: C=2, D=3, E=4, F=5, I=8, J=9, K=10, L=11, M=12, N=13, O=14, P=15, Q=16, R=17, S=18, T=19, U=20, AB=27 — stored only as a comment/reference, never used for lookup), `buildHeaderIndexMap(headerRow)`, `mapRepairRow(rowArray, headerIndexMap)`, `formatRepairDate(value)`, `classifyImageCell(value)`, `normalizePhone(value)`.
- Test file follows the exact harness already used by `matchLogic.test.js`/`aiChatLogic.test.js`: `const assert = require("assert");` + a local `pass`/`fail` counter + local `function test(name, fn) { try { fn(); pass++; console.log("PASS -", name); } catch (e) { fail++; console.log("FAIL -", name, "\n     ", e.message); } }`, ending with `console.log(\`\n${pass} pasaron, ${fail} fallaron\`); process.exit(fail > 0 ? 1 : 0);`. Do not invent a different assertion helper.

- [ ] **Step 1: Write the failing tests**

```js
const assert = require("assert");
const { normalizeText, parseNumericCell } = require("./matchLogic");

// Builds a 28-slot row (indices 0-27, covering A..AB) with the 18 real
// headers at their real positions and "x" everywhere else (A, B, G, H,
// V..AA — never read). `overrides` is {index: value} applied on top,
// so a test can blank out or duplicate one specific header.
function sampleHeaderRow(overrides) {
  const row = new Array(28).fill("x");
  row[2] = "Full Name"; row[3] = "Phone Mobile"; row[4] = "¿Marca/Modelo?"; row[5] = "Estado de la Reparacion:";
  row[8] = "¿Pantalla Rota?"; row[9] = "¿Equipo Encendido?"; row[10] = "Descripcion:"; row[11] = "Precio:";
  row[12] = "¿Condiciones del equipo?"; row[13] = "Numero de Factura:"; row[14] = "Fecha de Entrada"; row[15] = "Fecha de Modificacion";
  row[16] = "Imagen Frontal"; row[17] = "Imagen Trasera"; row[18] = "Imagen frontal de equipo listo."; row[19] = "Imagen trasera de equipo listo.";
  row[20] = "Comentario:"; row[27] = "Factura Paga";
  Object.entries(overrides || {}).forEach(([i, v]) => { row[Number(i)] = v; });
  return row;
}

test("mapea los 18 encabezados reales a sus indices", () => {
  const r = buildHeaderIndexMap(sampleHeaderRow());
  assert.strictEqual(r.ok, true);
  assert.strictEqual(r.map.fullName, 2);
  assert.strictEqual(r.map.facturaPaga, 27);
});

test("tolera espacios y mayusculas distintas en el encabezado", () => {
  const r = buildHeaderIndexMap(sampleHeaderRow({ 2: "  full name  ", 3: "PHONE MOBILE" }));
  assert.strictEqual(r.ok, true);
  assert.strictEqual(r.map.fullName, 2);
});

test("encabezado esperado ausente -> ok:false, reason:missing_header, field nombrado", () => {
  const r = buildHeaderIndexMap(sampleHeaderRow({ 2: "" })); // Full Name blanked out
  assert.strictEqual(r.ok, false);
  assert.strictEqual(r.reason, "missing_header");
  assert.strictEqual(r.field, "fullName");
});

test("encabezado duplicado -> ok:false, reason:duplicate_header", () => {
  const r = buildHeaderIndexMap(sampleHeaderRow({ 1: "Full Name" })); // column B also says "Full Name"
  assert.strictEqual(r.ok, false);
  assert.strictEqual(r.reason, "duplicate_header");
  assert.strictEqual(r.field, "fullName");
});

// mapRepairRow
function sampleRawRow() {
  const row = new Array(28).fill("");
  row[2] = "Juan Perez"; row[3] = "8092843485"; row[5] = "Recibido.";
  row[11] = "1500.50"; row[14] = new Date("2026-01-10T00:00:00Z");
  row[16] = "https://example.com/foto.jpg"; row[17] = "https://drive.google.com/file/d/ABC/view";
  return row;
}
test("mapea una fila completa en el orden de REPAIR_HEADER_FIELDS", () => {
  const map = buildHeaderIndexMap(sampleHeaderRow()).map;
  const row = mapRepairRow(sampleRawRow(), map);
  assert.deepStrictEqual(Object.keys(row), REPAIR_HEADER_FIELDS.map((f) => f.key));
  assert.strictEqual(row.precio.confirmed, true);
  assert.strictEqual(row.precio.value, 1500.5);
  assert.strictEqual(row.imagenFrontal.available, true);
  assert.strictEqual(row.imagenTrasera.available, false); // Drive share link, never assumed public
});
test("celdas vacias no lanzan excepcion y quedan null/vacias", () => {
  const map = buildHeaderIndexMap(sampleHeaderRow()).map;
  const row = mapRepairRow(new Array(28).fill(""), map);
  assert.strictEqual(row.fullName, "");
  assert.strictEqual(row.precio.confirmed, false);
  assert.strictEqual(row.fechaEntrada, null);
  assert.strictEqual(row.imagenFrontal.available, false);
});

// formatRepairDate
test("Date real -> string ISO", () => {
  assert.strictEqual(typeof formatRepairDate(new Date("2026-01-15T00:00:00Z")), "string");
});
test("celda vacia -> null", () => { assert.strictEqual(formatRepairDate(""), null); });
test("valor no-fecha -> null, no lanza excepcion", () => { assert.strictEqual(formatRepairDate("no es fecha"), null); });

// classifyImageCell
test("URL https directa a imagen -> available:true", () => {
  assert.strictEqual(classifyImageCell("https://example.com/foto.jpg").available, true);
});
test("celda vacia -> available:false", () => { assert.strictEqual(classifyImageCell("").available, false); });
test("enlace de Google Drive (share link) -> available:false, nunca se asume publico", () => {
  assert.strictEqual(classifyImageCell("https://drive.google.com/file/d/ABC123/view?usp=sharing").available, false);
});
test("ID de archivo de Drive sin URL -> available:false", () => { assert.strictEqual(classifyImageCell("1AbC-XyZ_fileId").available, false); });

// normalizePhone
test("quita espacios, guiones, parentesis y codigo de pais, deja solo digitos", () => {
  assert.strictEqual(normalizePhone("+1 (809) 284-3485"), "18092843485");
});
test("celda vacia -> cadena vacia", () => { assert.strictEqual(normalizePhone(""), ""); });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: fails with "function not defined" / module not found (file doesn't exist yet).

- [ ] **Step 3: Implement `backend/apps-script/tests-node/repairLogic.js`**

Implement exactly the functions listed in "Produces" above, plus `module.exports = { REPAIR_HEADER_FIELDS, buildHeaderIndexMap, mapRepairRow, formatRepairDate, classifyImageCell, normalizePhone };` (extended further in Tasks 2-3). `buildHeaderIndexMap` normalizes every header cell with `normalizeText` (required from `./matchLogic`) and matches against the normalized text of each `REPAIR_HEADER_FIELDS[i].header`; on a second column normalizing to a header already claimed by an earlier field, return `{ok:false, reason:"duplicate_header", field}` for that field. `classifyImageCell`'s only `available:true` path is a case-insensitive regex `^https:\/\/\S+\.(png|jpe?g|gif|webp)(\?\S*)?$`. `formatRepairDate` returns `value.toISOString()` when `value instanceof Date`, else `null`. `normalizePhone` is `String(value || "").replace(/\D/g, "")`. `mapRepairRow` uses `parseNumericCell` (required from `./matchLogic`) for the `precio` field only.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: all PASS, 0 failed.

- [ ] **Step 5: Commit**

```bash
git add backend/apps-script/tests-node/repairLogic.js backend/apps-script/tests-node/repairLogic.test.js
git commit -m "test: add pure header/date/image/phone parsing for repair backend"
```

---

### Task 2: Status authorization and filtering (Node, same module)

**Files:**
- Modify: `backend/apps-script/tests-node/repairLogic.js`
- Modify: `backend/apps-script/tests-node/repairLogic.test.js`

**Interfaces:**
- Consumes: `normalizeText` (local copy from Task 1), `mapRepairRow` output shape.
- Produces (used by Task 3 and Task 5): `AUTHORIZED_STATES` (the 5 canonical literal strings, in the exact order from Global Constraints), `canonicalAuthorizedState(rawState)` → `{authorized:true, canonical} | {authorized:false}`, `filterAuthorizedRepairs(rawRows)` → array of rows with `estado` replaced by the canonical label, unauthorized rows dropped.

- [ ] **Step 1: Write the failing tests**

```js
AUTHORIZED_STATES.forEach((canonical) => {
  test(`estado autorizado reconocido: "${canonical}"`, () => {
    assert.strictEqual(canonicalAuthorizedState(canonical).authorized, true);
    assert.strictEqual(canonicalAuthorizedState(canonical).canonical, canonical);
  });
  test(`variantes inocuas de "${canonical}" tambien se reconocen`, () => {
    const noisy = "  " + canonical.toUpperCase().replace(".", "") + "  ";
    assert.strictEqual(canonicalAuthorizedState(noisy).authorized, true);
    assert.strictEqual(canonicalAuthorizedState(noisy).canonical, canonical);
  });
});
test("estado vacio -> no autorizado", () => { assert.strictEqual(canonicalAuthorizedState("").authorized, false); });
test("estado desconocido -> no autorizado", () => { assert.strictEqual(canonicalAuthorizedState("Cancelado.").authorized, false); });
test("estado parecido pero distinto no colapsa con uno autorizado", () => {
  assert.strictEqual(canonicalAuthorizedState("Recibido en tienda.").authorized, false);
});

test("filterAuthorizedRepairs descarta estados no autorizados/vacios/desconocidos", () => {
  const rows = [
    { estado: "Recibido.", fullName: "A" },
    { estado: "Cancelado.", fullName: "B" },
    { estado: "", fullName: "C" },
    { estado: "equipo entregado", fullName: "D" }, // normaliza a "Equipo Entregado."
  ];
  const out = filterAuthorizedRepairs(rows);
  assert.deepStrictEqual(out.map((r) => r.fullName), ["A", "D"]);
  assert.strictEqual(out.find((r) => r.fullName === "D").estado, "Equipo Entregado.");
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: FAIL — `AUTHORIZED_STATES`/`canonicalAuthorizedState`/`filterAuthorizedRepairs` not defined.

- [ ] **Step 3: Implement in `backend/apps-script/tests-node/repairLogic.js`**

Add `AUTHORIZED_STATES` as the 5-item literal array from Global Constraints. `canonicalAuthorizedState(rawState)` normalizes `rawState` and each entry of `AUTHORIZED_STATES` with `normalizeText`, returns the first exact match's canonical literal, else `{authorized:false}`. `filterAuthorizedRepairs` maps each row through `canonicalAuthorizedState(row.estado)`, keeps only authorized ones with `estado` overwritten to the canonical string.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: all PASS, 0 failed.

- [ ] **Step 5: Commit**

```bash
git add backend/apps-script/tests-node/repairLogic.js backend/apps-script/tests-node/repairLogic.test.js
git commit -m "feat: add state authorization and filtering for repair records"
```

---

### Task 3: Identity search, disambiguation, and top-level lookup (Node, same module)

**Files:**
- Modify: `backend/apps-script/tests-node/repairLogic.js`
- Modify: `backend/apps-script/tests-node/repairLogic.test.js`

**Interfaces:**
- Consumes: `normalizePhone`, `normalizeText`, `filterAuthorizedRepairs` output shape (array of rows with `phoneMobile`, `fullName`, `numeroFactura`, `estado`, `marcaModelo` already canonical/mapped).
- Produces (used by Task 5): `searchRepairsByIdentity(authorizedRows, {phone, fullName})`, `disambiguateRepairsByInvoice(matches, numeroFactura)`, `buildDisclosedRecord(fullRecord)` → `{estado, marcaModelo}` only (the project's disclosure-policy allowlist — see Global Constraints), `lookupRepair(authorizedRows, {phone, fullName, numeroFactura})` → `{status:"missing_identifier"} | {status:"not_found"} | {status:"ambiguous"} | {status:"ok", record}` where `record` is always the output of `buildDisclosedRecord`, never the full matched row.

- [ ] **Step 1: Write the failing tests**

```js
function sampleAuthorizedRows() {
  return [
    { fullName: "Juan Perez", phoneMobile: "8092843485", numeroFactura: "F-001", estado: "Recibido.", marcaModelo: "HP Pavilion" },
    { fullName: "Juan Perez", phoneMobile: "8092843485", numeroFactura: "F-002", estado: "Equipo listo para retirar.", marcaModelo: "Samsung A06" },
    { fullName: "Maria Diaz", phoneMobile: "8292223485", numeroFactura: "F-003", estado: "Equipo Entregado.", marcaModelo: "iPhone 13" }, // shares a long digit run with the first two but is a different number
  ];
}

test("lookupRepair sin telefono -> missing_identifier", () => {
  assert.strictEqual(lookupRepair(sampleAuthorizedRows(), {}).status, "missing_identifier");
});
test("telefono sin coincidencias -> not_found", () => {
  assert.strictEqual(lookupRepair(sampleAuthorizedRows(), { phone: "8095550000" }).status, "not_found");
});
test("telefono que coincide por substring con otro NO hace match cruzado (aislamiento entre clientes)", () => {
  // "092843485" is a substring shared by both real numbers above when naively compared; exact match must still separate them.
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8292223485" });
  assert.strictEqual(r.status, "ok");
  assert.strictEqual(r.record.marcaModelo, "iPhone 13"); // Maria Diaz's device, never Juan Perez's data
});
test("consultar a un cliente nunca revela ni insinua datos de otro (aislamiento entre clientes)", () => {
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8092843485", numeroFactura: "F-001" });
  assert.strictEqual(r.status, "ok");
  assert.deepStrictEqual(Object.keys(r.record), ["estado", "marcaModelo"]);
  assert.strictEqual(JSON.stringify(r).includes("Maria"), false);
  assert.strictEqual(JSON.stringify(r).includes("iPhone 13"), false); // Maria's device never appears
});
test("telefono con varias coincidencias y sin factura -> ambiguous, sin exponer cuenta ni registros", () => {
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8092843485" });
  assert.strictEqual(r.status, "ambiguous");
  assert.deepStrictEqual(Object.keys(r), ["status"]); // nothing else on the object: no count, no candidates
});
test("telefono ambiguo + numero de factura correcto -> ok, el registro correcto (por estado/modelo, nunca por numeroFactura expuesto)", () => {
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8092843485", numeroFactura: "F-002" });
  assert.strictEqual(r.status, "ok");
  assert.strictEqual(r.record.estado, "Equipo listo para retirar.");
  assert.strictEqual(r.record.marcaModelo, "Samsung A06");
  assert.deepStrictEqual(Object.keys(r.record), ["estado", "marcaModelo"]); // nunca numeroFactura, fullName, etc.
});
test("telefono ambiguo + numero de factura que no coincide con ninguna -> not_found", () => {
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8092843485", numeroFactura: "F-999" });
  assert.strictEqual(r.status, "not_found");
});
test("nombre completo es opcional y solo refina, no sustituye al telefono", () => {
  const r = lookupRepair(sampleAuthorizedRows(), { phone: "8092843485", fullName: "Nombre Que No Coincide" });
  assert.strictEqual(r.status, "not_found"); // phone matches two rows but fullName filter (AND) leaves zero
});

test("un registro con estado no autorizado nunca llega a la busqueda (prueba obligatoria #8 de la seccion 9)", () => {
  // Si lookupRepair solo recibe lo que ya pasó por filterAuthorizedRepairs, un
  // registro no autorizado es estructuralmente invisible aunque su teléfono
  // coincida exactamente con el buscado.
  const raw = [
    { estado: "Cancelado.", fullName: "Oculto", phoneMobile: "8091111111", numeroFactura: "F-9", marcaModelo: "X" },
    { estado: "Recibido.", fullName: "Visible", phoneMobile: "8092222222", numeroFactura: "F-8", marcaModelo: "Y" },
  ];
  const authorized = filterAuthorizedRepairs(raw);
  const hidden = lookupRepair(authorized, { phone: "8091111111" });
  assert.strictEqual(hidden.status, "not_found"); // nunca "ok": el registro oculto no sobrevivió al filtro
});

test("buildDisclosedRecord nunca incluye campos personales aunque el registro de entrada los tenga", () => {
  const full = { estado: "Recibido.", marcaModelo: "HP", fullName: "X", phoneMobile: "809", precio: { value: 100, confirmed: true }, numeroFactura: "F-1", comentario: "texto", imagenFrontal: { available: true, url: "https://x.com/a.jpg" } };
  assert.deepStrictEqual(buildDisclosedRecord(full), { estado: "Recibido.", marcaModelo: "HP" });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: FAIL — `searchRepairsByIdentity`/`disambiguateRepairsByInvoice`/`lookupRepair` not defined.

- [ ] **Step 3: Implement in `backend/apps-script/tests-node/repairLogic.js`**

`searchRepairsByIdentity(rows, {phone, fullName})` filters rows where `normalizePhone(row.phoneMobile) === normalizePhone(phone)` (exact, never substring), AND, only if `fullName` is truthy, `normalizeText(row.fullName) === normalizeText(fullName)`. `disambiguateRepairsByInvoice(matches, numeroFactura)` filters by `normalizeText(row.numeroFactura) === normalizeText(numeroFactura)`. `buildDisclosedRecord(fullRecord)` returns `{ estado: fullRecord.estado, marcaModelo: fullRecord.marcaModelo }` — exactly these two keys, nothing else, regardless of what other keys `fullRecord` carries. `lookupRepair` follows the 5-step algorithm from the plan's design (missing phone → `missing_identifier`; 0 matches → `not_found`; >1 matches → try invoice disambiguation if provided, else `ambiguous`; exactly 1 → `{status:"ok", record: buildDisclosedRecord(matches[0])}` — never the raw matched row). The `ambiguous` return value must be a plain `{status:"ambiguous"}` object with no other keys. Update `module.exports` to add `AUTHORIZED_STATES, canonicalAuthorizedState, filterAuthorizedRepairs, searchRepairsByIdentity, disambiguateRepairsByInvoice, buildDisclosedRecord, lookupRepair` to the Task 1 list.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend/apps-script/tests-node && node repairLogic.test.js`
Expected: all PASS, 0 failed.

- [ ] **Step 5: Commit**

```bash
git add backend/apps-script/tests-node/repairLogic.js backend/apps-script/tests-node/repairLogic.test.js
git commit -m "feat: add identity search, invoice disambiguation, and lookupRepair"
```

---

### Task 4: Generalize the existing AI tool-calling loop (Node, backward-compatible)

**Files:**
- Modify: `backend/apps-script/tests-node/aiChatLogic.js`
- Modify: `backend/apps-script/tests-node/aiChatLogic.test.js` (add new cases only — do not remove or change existing ones)

**Interfaces:**
- Consumes: nothing new.
- Produces (used by Task 5): `KcChat_runAiChatLoop_(userMessage, deps)` now also accepts optional `deps.executeTool` (function, same signature as `KcChat_executeOneTool_`; defaults to `KcChat_executeOneTool_` when omitted) and optional `deps.resultKey` (string; defaults to `"products"`).

- [ ] **Step 1: Write the failing tests**

Append to the existing `backend/apps-script/tests-node/aiChatLogic.test.js` (same file, same `assert`/local `test()` harness, same local `toolCallMsg`/`finalMsg` helpers already defined near the top of that file — do not redefine them, do not remove any existing test above this point):

```js
test("deps.executeTool permite inyectar un executor distinto sin tocar el default", () => {
  let called = false;
  const customExecutor = () => { called = true; return { isError: false, content: "{}", products: [] }; };
  let call = 0;
  const callModel = (messages) => {
    call++;
    if (call === 1) return { ok: true, message: toolCallMsg("c1", "cualquier_nombre", {}) };
    return { ok: true, message: finalMsg("listo") };
  };
  KcChat_runAiChatLoop_("hola", { callModel, callTool: () => ({ ok: true, result: {} }), validateQueryFn: validateQuery, executeTool: customExecutor });
  assert.strictEqual(called, true);
});

test("deps.resultKey renombra la clave de resultados sin romper el default 'products'", () => {
  const r = KcChat_runAiChatLoop_("saludo simple", { callModel: () => ({ ok: true, message: finalMsg("hola") }), callTool: () => ({ ok: true, result: {} }), validateQueryFn: validateQuery, resultKey: "repairs" });
  assert.ok(Object.prototype.hasOwnProperty.call(r, "repairs"));
  assert.ok(!Object.prototype.hasOwnProperty.call(r, "products"));
});

// Regression: existing behavior when neither override is passed is untouched.
test("sin overrides, el comportamiento es igual al existente (clave 'products', KcChat_executeOneTool_ por defecto)", () => {
  const r = KcChat_runAiChatLoop_("saludo", { callModel: () => ({ ok: true, message: finalMsg("hola") }), callTool: () => ({ ok: true, result: {} }), validateQueryFn: validateQuery });
  assert.ok(Object.prototype.hasOwnProperty.call(r, "products"));
});
```

`validateQuery` above is the binding this test file already imports at its top (`const { searchProducts, calculateSubtotal, validateQuery } = require("./matchLogic");`) — reuse it, don't re-import.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend/apps-script/tests-node && node aiChatLogic.test.js`
Expected: the 3 new tests FAIL (no override support yet); the 12 pre-existing tests still PASS.

- [ ] **Step 3: Implement in `backend/apps-script/tests-node/aiChatLogic.js`**

Inside `KcChat_runAiChatLoop_`, at the top: `const executeOneTool = deps.executeTool || KcChat_executeOneTool_;` and `const resultKey = deps.resultKey || "products";`. Replace the direct call to `KcChat_executeOneTool_(...)` with `executeOneTool(...)`. Replace the two object literals that currently have a hardcoded `products: collectedProducts` key with `[resultKey]: collectedProducts` (computed property name).

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend/apps-script/tests-node && node aiChatLogic.test.js`
Expected: `15 pasaron, 0 fallaron` (12 original + 3 new).

- [ ] **Step 5: Commit**

```bash
git add backend/apps-script/tests-node/aiChatLogic.js backend/apps-script/tests-node/aiChatLogic.test.js
git commit -m "feat: make the tool-calling loop's executor and result key injectable"
```

---

### Task 5: Repair tool definition, system prompt, and tool executor (Node)

**Files:**
- Create: `backend/apps-script/tests-node/repairChatLogic.js`
- Test: `backend/apps-script/tests-node/repairChatLogic.test.js`

**Interfaces:**
- Consumes: `KcChat_runAiChatLoop_` from Task 4 (with `executeTool`/`resultKey` overrides), `lookupRepair` shape from Task 3 (this file doesn't import `repairLogic.js` directly — in these tests `deps.callTool` is a mock, exactly like the existing `aiChatLogic.test.js` pattern; the real wiring to `lookupRepair` happens only in the `.gs` port, Task 6).
- Produces: `TOOL_CONSULTAR_REPARACION`, `REPAIR_AUTHORIZED_TOOLS`, `CONSULTAR_REPARACION_TOOL` (OpenAI-style tool-def object: required `phone` string, optional `fullName` string, optional `numeroFactura` string), `REPAIR_SYSTEM_PROMPT`, `KcChat_executeOneRepairTool_(call, deps, validateQueryFn)`.

- [ ] **Step 1: Write the failing tests**

```js
const assert = require("assert");
const { KcChat_runAiChatLoop_ } = require("./aiChatLogic");
const { TOOL_CONSULTAR_REPARACION, REPAIR_SYSTEM_PROMPT, KcChat_executeOneRepairTool_ } = require("./repairChatLogic");

let pass = 0, fail = 0;
function test(name, fn) {
  try { fn(); pass++; console.log("PASS -", name); }
  catch (e) { fail++; console.log("FAIL -", name, "\n     ", e.message); }
}
function toolCallMsg(id, name, args) {
  return { finish_reason: "tool_calls", content: null, tool_calls: [{ id, type: "function", function: { name, arguments: JSON.stringify(args) } }] };
}
function finalMsg(text) { return { finish_reason: "stop", content: text, tool_calls: null }; }
const okValidate = (s) => ({ valid: true, value: s });

test("herramienta no autorizada (ej. nombre de una herramienta de inventario) nunca se ejecuta", () => {
  let called = false;
  const deps = { callTool: () => { called = true; return { ok: true, result: {} }; } };
  const out = KcChat_executeOneRepairTool_({ function: { name: "calculate_subtotal", arguments: "{}" } }, deps, okValidate);
  assert.strictEqual(called, false);
  assert.ok(JSON.parse(out.content).message.toLowerCase().includes("no autorizada"));
});

test("consulta ok: el contenido solo incluye estado y marcaModelo, aunque callTool devuelva mas campos", () => {
  // Defensa en profundidad: aunque deps.callTool (o un bug en lookupRepair) devolviera
  // de mas, el ejecutor reconstruye el record desde cero con su propia lista blanca —
  // nunca confia en el shape que llega.
  const record = { estado: "Recibido.", marcaModelo: "HP Pavilion", fullName: "Juan Perez", phoneMobile: "8092843485", precio: { value: 100, confirmed: true } };
  const deps = { callTool: () => ({ ok: true, result: { status: "ok", record } }) };
  const out = KcChat_executeOneRepairTool_({ function: { name: TOOL_CONSULTAR_REPARACION, arguments: JSON.stringify({ phone: "8092843485" }) } }, deps, okValidate);
  const parsed = JSON.parse(out.content);
  assert.deepStrictEqual(Object.keys(parsed.record), ["estado", "marcaModelo"]);
  assert.strictEqual(parsed.record.estado, "Recibido.");
  assert.strictEqual(out.content.includes("Juan Perez"), false);
  assert.strictEqual(out.content.includes("8092843485"), false);
  assert.strictEqual(out.repairs.length, 1);
  assert.deepStrictEqual(Object.keys(out.repairs[0]), ["estado", "marcaModelo"]);
});

test("consulta ambigua: el contenido NUNCA incluye conteo ni candidatos", () => {
  const deps = { callTool: () => ({ ok: true, result: { status: "ambiguous" } }) };
  const out = KcChat_executeOneRepairTool_({ function: { name: TOOL_CONSULTAR_REPARACION, arguments: JSON.stringify({ phone: "8092843485" }) } }, deps, okValidate);
  const parsed = JSON.parse(out.content);
  assert.deepStrictEqual(Object.keys(parsed), ["status"]);
  assert.deepStrictEqual(out.repairs, []);
});

test("consulta sin telefono (missing_identifier): el executor no inventa un telefono, deja que el modelo lo pida", () => {
  const deps = { callTool: () => ({ ok: true, result: { status: "missing_identifier" } }) };
  const out = KcChat_executeOneRepairTool_({ function: { name: TOOL_CONSULTAR_REPARACION, arguments: "{}" } }, deps, okValidate);
  assert.deepStrictEqual(out.repairs, []);
});

test("fallo al leer la hoja: no se inventa un registro, se informa error generico", () => {
  const deps = { callTool: () => ({ ok: false, reason: "sheet-unreachable" }) };
  const out = KcChat_executeOneRepairTool_({ function: { name: TOOL_CONSULTAR_REPARACION, arguments: JSON.stringify({ phone: "8092843485" }) } }, deps, okValidate);
  assert.strictEqual(out.isError, true);
  assert.deepStrictEqual(out.repairs, []);
});

test("flujo completo: telefono real -> consulta de reparacion -> respuesta ok", () => {
  let call = 0;
  const callModel = (messages) => {
    call++;
    if (call === 1) return { ok: true, message: toolCallMsg("c1", TOOL_CONSULTAR_REPARACION, { phone: "8092843485" }) };
    return { ok: true, message: finalMsg("Tu equipo esta en estado: Recibido.") };
  };
  const callTool = () => ({ ok: true, result: { status: "ok", record: { fullName: "Juan Perez", phoneMobile: "8092843485", estado: "Recibido.", marcaModelo: "HP Pavilion" } } });
  const r = KcChat_runAiChatLoop_("cual es el estado de mi reparacion", {
    callModel, callTool, executeTool: KcChat_executeOneRepairTool_, resultKey: "repairs", systemPrompt: REPAIR_SYSTEM_PROMPT, validateQueryFn: okValidate,
  });
  assert.strictEqual(r.status, "ok");
  assert.strictEqual(r.repairs.length, 1);
  assert.deepStrictEqual(r.repairs[0], { estado: "Recibido.", marcaModelo: "HP Pavilion" });
});

console.log(`\n${pass} pasaron, ${fail} fallaron`);
process.exit(fail > 0 ? 1 : 0);
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend/apps-script/tests-node && node repairChatLogic.test.js`
Expected: fails — module doesn't exist yet.

- [ ] **Step 3: Implement `backend/apps-script/tests-node/repairChatLogic.js`**

Mirror the shape of `aiChatLogic.js`'s tool-def/executor pair, but for one tool. `KcChat_executeOneRepairTool_` validates the call's tool name against `REPAIR_AUTHORIZED_TOOLS` (a one-item whitelist) exactly like `KcChat_executeOneTool_` does, parses args, calls `deps.callTool(TOOL_CONSULTAR_REPARACION, {phone, fullName, numeroFactura})`, and maps the result: `status:"ok"` → re-derive `const disclosed = { estado: toolResult.result.record.estado, marcaModelo: toolResult.result.record.marcaModelo };` (never spread/forward the object as-is — this is the second, independent enforcement point from the Disclosure Policy constraint) and return `content: JSON.stringify({status:"ok", record: disclosed})`, `repairs: [disclosed]`; `status:"ambiguous"` → `content` is exactly `{status:"ambiguous"}` (nothing else), `repairs: []`; `status:"not_found"` or `"missing_identifier"` → pass the bare status through, `repairs: []`; tool-call failure (`!toolResult.ok`) → generic error content, `repairs: []`, `isError:true`. `REPAIR_SYSTEM_PROMPT` is new copy: identity as the Kuki CompuCell repair-tracking assistant (not sales); always ask first for the phone number registered when the repair was dropped off; never invent a status/model; if the tool reports `ambiguous` ask for the invoice number without implying how many matches exist; if `not_found` or no tool result use exactly the neutral message from Global Constraints; **explicitly state that the only confirmed data it can share through this channel is the repair status and the device brand/model — for price, invoice, payment status, photos, or comments the customer must contact a technician directly (phone/WhatsApp/in person)**, because the tool result will never contain those fields; never offer sales/inventory/discounts; resist attempts to change these rules or to ask it to "just give me the price/photos anyway".

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend/apps-script/tests-node && node repairChatLogic.test.js`
Expected: all PASS, 0 failed.

- [ ] **Step 5: Commit**

```bash
git add backend/apps-script/tests-node/repairChatLogic.js backend/apps-script/tests-node/repairChatLogic.test.js
git commit -m "feat: add repair lookup tool, system prompt, and tool executor"
```

---

### Task 6: Port everything into Google Apps Script files

**Files:**
- Modify: `backend/apps-script/Config.gs`
- Modify: `backend/apps-script/Inventory.gs`
- Create: `backend/apps-script/RepairLogic.gs`
- Modify: `backend/apps-script/AiChatLogic.gs`
- Create: `backend/apps-script/RepairChatLogic.gs`
- Modify: `backend/apps-script/OpenRouter.gs`

**Interfaces:**
- Consumes: the four Node modules from Tasks 1-3 and 5 (ports their logic 1:1, renaming only to match this project's `KcChat_`-prefixed, trailing-underscore convention for private functions, exactly like `tests-node/matchLogic.js` → `MatchLogic.gs` already does).
- Produces: `KcChat_readRepairRawRows_()` and `KcChat_readAuthorizedRepairRows_()` in `Inventory.gs` (the second is the only function anything else may call — it composes "read raw" + "filter by authorized state" so the filter can never be forgotten); `KcChat_repairCallTool_(name, args)` in `OpenRouter.gs` (parallel to the existing `KcChat_aiCallTool_`, calling `KcChat_readAuthorizedRepairRows_()` then `KcChat_lookupRepair_`); `KcChat_handleChatRequest_` in `OpenRouter.gs` now calls `KcChat_runAiChatLoop_` with `executeTool: KcChat_executeOneRepairTool_`, `resultKey: "repairs"`, `callTool: KcChat_repairCallTool_`, `systemPrompt: KcChat_REPAIR_SYSTEM_PROMPT_`.

- [ ] **Step 1: Add config to `Config.gs`**

Add to `KcChat_CONFIG`: `REPAIR_SHEET_ID: "1BYdv9Bq28CJhV8vJsFr-nmMrkgIbz3Qe4ZenSaODm_s"`, `REPAIR_SHEET_NAME: "kuki CompuCell Base Datos"`. Add a new top-level constant `KcChat_REPAIR_HEADER_FIELDS_` (copy of Task 1's `REPAIR_HEADER_FIELDS`) and `KcChat_REPAIR_AUTHORIZED_STATES_` (copy of Task 2's `AUTHORIZED_STATES`). Do not touch `SHEET_ID`/`SHEET_NAME` (inventory) or any existing key.

- [ ] **Step 2: Create `backend/apps-script/RepairLogic.gs`**

Copy the implementations from `tests-node/repairLogic.js` (Tasks 1-3) verbatim, renaming the module-local `normalizeText`/`parseNumericCell` calls to reuse the existing `KcChat_normalizeText_`/`KcChat_parseNumericCell_` from `MatchLogic.gs` instead of duplicating them (this file already loads in the same Apps Script project, unlike the Node tests which can't `require` a `.gs` file), and prefixing all other function/constant names with `KcChat_` and a trailing underscore for internals (`KcChat_buildHeaderIndexMap_`, `KcChat_mapRepairRow_`, `KcChat_formatRepairDate_`, `KcChat_classifyImageCell_`, `KcChat_normalizePhone_`, `KcChat_canonicalAuthorizedState_`, `KcChat_filterAuthorizedRepairs_`, `KcChat_searchRepairsByIdentity_`, `KcChat_disambiguateRepairsByInvoice_`, `KcChat_lookupRepair_`), matching `KcChat_CONFIG.REPAIR_AUTHORIZED_STATES_`/`KcChat_REPAIR_HEADER_FIELDS_` from Step 1 instead of local constants.

- [ ] **Step 3: Add to `backend/apps-script/Inventory.gs`**

```
function KcChat_readRepairRawRows_() { /* open REPAIR_SHEET_ID/REPAIR_SHEET_NAME, getDataRange().getValues(),
  build header map via KcChat_buildHeaderIndexMap_(values[0]); throw KcChat_ConfigError_ naming only
  reason+field on failure; map each non-blank row (blank = fullName AND phoneMobile both empty) via
  KcChat_mapRepairRow_; return array */ }

function KcChat_readAuthorizedRepairRows_() {
  return KcChat_filterAuthorizedRepairs_(KcChat_readRepairRawRows_());
}
```

Follow the exact same try/catch-and-redact pattern already used by `KcChat_readInventoryRows_` in this file (never surface the real Google error message or the Sheet ID).

- [ ] **Step 4: Apply Task 4's generalization to `backend/apps-script/AiChatLogic.gs`**

Same two changes as Task 4 made in the Node mirror: `deps.executeTool || KcChat_executeOneTool_` and computed `[deps.resultKey || "products"]: collectedProducts`.

- [ ] **Step 5: Create `backend/apps-script/RepairChatLogic.gs`**

Copy `tests-node/repairChatLogic.js` (Task 5) verbatim with the project's naming convention (`KcChat_TOOL_CONSULTAR_REPARACION_`, `KcChat_REPAIR_AUTHORIZED_TOOLS_`, `KcChat_CONSULTAR_REPARACION_TOOL_`, `KcChat_REPAIR_SYSTEM_PROMPT_`, `KcChat_executeOneRepairTool_`).

- [ ] **Step 6: Modify `backend/apps-script/OpenRouter.gs`**

Add:
```
function KcChat_repairCallTool_(name, args) {
  try {
    const rows = KcChat_readAuthorizedRepairRows_();
    return { ok: true, result: KcChat_lookupRepair_(rows, { phone: args.phone, fullName: args.fullName, numeroFactura: args.numeroFactura }) };
  } catch (e) {
    return { ok: false, reason: e.message };
  }
}
```
Change `KcChat_handleChatRequest_` to pass `executeTool: KcChat_executeOneRepairTool_, resultKey: "repairs", callTool: KcChat_repairCallTool_, systemPrompt: KcChat_REPAIR_SYSTEM_PROMPT_` to `KcChat_runAiChatLoop_` instead of the current inventory `callTool`/implicit defaults. Leave `KcChat_callOpenRouter_`/`KcChat_callOpenRouterOnce_` untouched — they're generic HTTP plumbing, not inventory-specific.

- [ ] **Step 7: Verify no write calls were introduced**

Run: `grep -niE "setValue|setValues|appendRow|deleteRow|\.clear\(|insertRow|removeRow" backend/apps-script/RepairLogic.gs backend/apps-script/RepairChatLogic.gs backend/apps-script/Inventory.gs backend/apps-script/OpenRouter.gs backend/apps-script/Config.gs backend/apps-script/AiChatLogic.gs`
Expected: no matches outside of comments that *describe* the read-only guarantee (same as the Phase 1 audit's grep — compare against it).

- [ ] **Step 8: Commit**

```bash
git add backend/apps-script/Config.gs backend/apps-script/Inventory.gs backend/apps-script/RepairLogic.gs backend/apps-script/AiChatLogic.gs backend/apps-script/RepairChatLogic.gs backend/apps-script/OpenRouter.gs
git commit -m "feat: wire the repair lookup backend into the Apps Script chat endpoint"
```

---

### Task 7: Extend `Tests.gs` (manual Apps Script test runner)

**Files:**
- Modify: `backend/apps-script/Tests.gs`

**Interfaces:**
- Consumes: every `KcChat_*` function from Task 6.
- Produces: new `run(...)` entries inside `KcChat_runAllTests`, following the existing `run("name", fn)` + `KcChat_assertEqual_`/`KcChat_assertTrue_` pattern already in this file. Do not remove or reorder any existing `run(...)` line.

- [ ] **Step 1: Add test functions mirroring Tasks 1, 2, 3, and 5's Node cases**

Port the Node test bodies from `repairLogic.test.js` and `repairChatLogic.test.js` into `KcChat_test*` functions using `KcChat_assertEqual_`/`KcChat_assertTrue_` (same translation `matchLogic.test.js` → the existing `KcChat_test*` functions in this file already demonstrates). Register each with a new `run(...)` line inside `KcChat_runAllTests`, placed after the existing AI-chat test block and before the final integration test.

- [ ] **Step 2: Add one real-sheet read-only integration test**

```
function KcChat_testRepairSheetIsNeverModified() {
  KcChat_assertConfigured_(); // extend this if it doesn't yet check REPAIR_SHEET_ID/NAME
  const before = KcChat_snapshotRepairSheet_();
  KcChat_lookupRepair_(KcChat_readAuthorizedRepairRows_(), { phone: "0000000000" });
  KcChat_lookupRepair_(KcChat_readAuthorizedRepairRows_(), {});
  const after = KcChat_snapshotRepairSheet_();
  KcChat_assertEqual_(after, before, "La hoja de reparaciones cambió después de usar el backend de solo lectura");
}
function KcChat_snapshotRepairSheet_() {
  return SpreadsheetApp.openById(KcChat_CONFIG.REPAIR_SHEET_ID).getSheetByName(KcChat_CONFIG.REPAIR_SHEET_NAME).getDataRange().getValues();
}
```
Register it with `run(...)` in `KcChat_runAllTests`, same place as the existing `KcChat_testInventoryIsNeverModified`.

- [ ] **Step 3: Note what cannot run from this session**

This task's Step 2 test, and any test touching `SpreadsheetApp`, cannot be executed here — there's no Google account access in this environment. Record this explicitly in Task 8's documentation step; do not claim it passed.

- [ ] **Step 4: Commit**

```bash
git add backend/apps-script/Tests.gs
git commit -m "test: mirror repair backend tests inside the Apps Script test runner"
```

---

### Task 8: Final verification sweep and phase documentation

**Files:**
- Create: `docs/repair-agent-backend-phase-2.md`

**Interfaces:**
- Consumes: the results of every prior task.
- Produces: the Phase 2 report the spec's section 10 asks for.

- [ ] **Step 1: Re-run every Node test file and record exact output**

Run: `cd backend/apps-script/tests-node && node matchLogic.test.js && node aiChatLogic.test.js && node repairLogic.test.js && node repairChatLogic.test.js`
Expected: `matchLogic.test.js` → `30 pasaron, 0 fallaron` (unchanged); `aiChatLogic.test.js` → `15 pasaron, 0 fallaron` (12 original + 3 from Task 4); `repairLogic.test.js` and `repairChatLogic.test.js` → all PASS, 0 failed, with the exact counts from Tasks 1-3 and 5.

- [ ] **Step 2: Confirm no out-of-scope files changed**

`backend/apps-script/` is gitignored (see "Process Adaptation" above), so `git diff` cannot see changes there — use it only for what it CAN see, then check the gitignored tree separately:
1. Run `git status --short`. It must show nothing outside `docs/` (and the plan file already present under `docs/superpowers/plans/`) — in particular, zero output for `frontend/`, `.github/`, or any deployment config. Any such line aborts this task.
2. Diff `backend/apps-script/Code.gs` and `backend/apps-script/appsscript.json` against the copies saved in Task 1's `task-1-before/` snapshot (or, if a later task happened to snapshot them too, the earliest snapshot available) — both must be byte-identical to before Task 1 started. This confirms rate limiting/routing untouched and no new OAuth scopes were requested.
3. List `backend/apps-script/` (e.g. `find backend/apps-script -type f`) and confirm the full file set is exactly the pre-existing files plus: `RepairLogic.gs`, `RepairChatLogic.gs`, `tests-node/repairLogic.js`, `tests-node/repairLogic.test.js`, `tests-node/repairChatLogic.js`, `tests-node/repairChatLogic.test.js` — no other new file.

- [ ] **Step 2b: Re-run the final write-operation grep across the whole backend, not just the new files**

Run: `grep -niE "setValue|setValues|appendRow|deleteRow|\.clear\(|insertRow|removeRow" backend/apps-script/*.gs backend/apps-script/consolidated/*.gs`
Expected: identical to the Phase 1 audit's result — matches only inside comments describing the read-only guarantee, none inside the new `Tests.gs` additions or any other file.

- [ ] **Step 3: Write `docs/repair-agent-backend-phase-2.md`**

Include, as the spec's section 10 requires: files modified/created (the exact list from Tasks 1-7); new/modified functions (the "Produces" lines from each task); the 18-field header map and its exact source headers; the state-authorization rule and the 5 canonical labels (note that the Phase 2 spec text briefly listed a sixth, `Pieza en Transito.`, that an earlier draft of this plan included after a clarifying question, and that the user explicitly withdrew it in their final approval, confirming no explicit authorization exists to add any state beyond the Phase 1 audit's original 5 — this is the authoritative, final list); how multiple matches are handled (ask for invoice number, never expose count); the privacy decision — phone number is used only as a **search key**, never as proof of identity (the user explicitly rejected treating name+phone as sufficient identity verification after reviewing the first draft of this plan); the record released once a match is resolved is therefore limited to `estado` + `marcaModelo` only, enforced independently in both `lookupRepair` and `KcChat_executeOneRepairTool_`; the residual risk that remains even with this reduced disclosure (anyone who knows/guesses a customer's registered phone number can learn that customer's repair status and device model, though nothing more sensitive) and what a stronger mechanism would need before expanding disclosure (e.g., an OTP sent to that phone via WhatsApp/SMS) exactly as the spec's section 7 asks for when full exposure isn't gated by real verification; the real image-field format — **unresolved**, because this session has no access to the live Sheet to inspect actual Q/R/S/T cell values, so `KcChat_classifyImageCell_`'s conservative allowlist (only a direct `https://...image-extension` URL) is a placeholder that must be revisited once someone with Sheet access reports the real format; **permissions note** — opening a second Sheet by ID (`REPAIR_SHEET_ID`) does not require any new Apps Script OAuth scope beyond what `SpreadsheetApp` already requires for the existing inventory sheet (confirmed: `appsscript.json` has zero diff, per Step 2) — it only requires that the Google account the script executes as (today: "Ejecutar como: Yo", i.e. the script owner) has at least viewer access shared on that specific sheet in Google's own sharing settings, which is outside this repo and outside this session's visibility; **logging note** — no new code calls `Logger.log` with any repair record field or with the OpenRouter API key (confirms section 7's "no registrar en logs" requirement by construction, not by a runtime check — grep `backend/apps-script/RepairLogic.gs backend/apps-script/RepairChatLogic.gs backend/apps-script/Inventory.gs backend/apps-script/OpenRouter.gs` for `Logger.log` and list every call site, if any); tests executed and their exact pass counts from Step 1; tests that could not run (the real-Sheets integration test from Task 7, and anything needing a real OpenRouter call) and why; and the steps needed before the frontend can be connected (Phase 3 must read the new `repairs` field instead of `products`, build a new card component per the Phase 1 audit's section 8, and someone with Google access must confirm headers/image format and run `KcChat_runAllTests` for real).

- [ ] **Step 4: Commit**

```bash
git add docs/repair-agent-backend-phase-2.md
git commit -m "docs: add Phase 2 repair backend report"
```
