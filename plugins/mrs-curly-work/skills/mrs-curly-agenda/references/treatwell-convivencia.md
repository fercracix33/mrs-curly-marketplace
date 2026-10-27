# Convivencia con Treatwell: huecos y registro en los dos sistemas

> Solo para cuando `treatwell_status` indica que el conector no está conectado. Con el conector, usa las herramientas normales de agenda.

Mientras Treatwell Pro siga siendo la agenda responsable de Mrs Curly, una hora solo existe si está libre en Treatwell. poliHair es la copia donde se trabaja la ficha, la conversación y el seguimiento, y recibe la cita después. Esta guía explica cómo leer los huecos de Treatwell en segundos y en qué orden registrar.

## Cuándo aplica

- Siempre que vayas a ofrecer, confirmar, mover o cancelar una hora mientras dure la convivencia.
- Solo desde un cliente con control de navegador (por ejemplo Claude en el escritorio con Chrome, o Codex) y con la sesión de Treatwell Pro ya iniciada en ese navegador. Desde ChatGPT en el móvil no hay navegador: di que no puedes comprobar Treatwell y pide que se verifique allí antes de confirmar.
- Nunca escribas contraseñas ni códigos de verificación. Si Treatwell pide iniciar sesión, para y pídeselo a la usuaria.

## Paso 1. Abrir la agenda

Abre una pestaña nueva en `https://pro.treatwell.es/agenda`. Si aparece la agenda con las columnas de profesionales, la sesión está activa. La agenda acepta la fecha en la URL con el formato `/agenda/AAAA-MM-DD`; úsala para ir a un día concreto en lugar de pulsar en la tira de fechas, que se desplaza sola.

Trampas conocidas de ese panel: la traducción automática de Chrome altera los textos, el zoom desplaza los clics por coordenadas, las capturas de pantalla pueden tardar o fallar y aparecen encuestas emergentes que se cierran con «Pregúntame más tarde». Lee datos por el script, no por capturas.

## Paso 2. Capturar la sesión en memoria

El panel autentica sus peticiones con encabezados que viven en memoria. Ejecuta este bloque una vez por pestaña con la herramienta de JavaScript. Instala un gancho sobre `fetch` y fuerza una navegación interna de la agenda para que el panel emita una petición autenticada. Los encabezados quedan en `window.__phAuth`; no los muestres, no los copies a la conversación y no los guardes en ningún sitio.

```javascript
// Instala un gancho sobre fetch y fuerza una navegación interna de la agenda
// para que el panel emita una petición autenticada. Los encabezados quedan en
// window.__phAuth, solo en memoria de la pestaña. No se devuelven ni se registran.
if (!window.__phHook) {
  window.__phHook = true;
  var __phOrigFetch = window.fetch;
  window.fetch = function (input, init) {
    try {
      var url = typeof input === "string" ? input : input.url;
      var src = (init && init.headers) || (input && input.headers) || {};
      var h = {};
      if (typeof src.forEach === "function") src.forEach(function (v, k) { h[String(k).toLowerCase()] = v; });
      else Object.keys(src).forEach(function (k) { h[k.toLowerCase()] = src[k]; });
      if (/api\.uala\.com/.test(url) && (h.authorization || h["x-client-auth"]))
        window.__phAuth = { authorization: h.authorization, "x-client-auth": h["x-client-auth"] };
    } catch (e) {}
    return __phOrigFetch.apply(this, arguments);
  };
}
history.pushState({}, "", "/agenda/" + new Date(Date.now() + 86400000).toISOString().slice(0, 10));
window.dispatchEvent(new PopStateEvent("popstate", { state: {} }));
// Esperar unos 3 segundos y comprobar que `!!window.__phAuth` es true.
```

Espera unos tres segundos y comprueba que `!!window.__phAuth` devuelve `true`. Si es `false`, repite la navegación a otro día una sola vez. Si sigue sin capturarse, la sesión no está activa: para y avisa.

## Paso 3. Leer los huecos

Define la función ejecutando el bloque siguiente en la misma pestaña. Después llámala con los parámetros del caso. Solo hace peticiones de lectura a las colecciones que el propio panel carga: citas del rango, profesionales, asignaciones, tratamientos, grupos, bloqueos, horarios especiales, aperturas extraordinarias y la ficha del salón. No crea, modifica ni borra nada.

```javascript
window.__polihairTreatwellHuecos = async function (input) {
  var TZ = "Europe/Madrid";
  var ACCOUNT = "67426";
  var API = "https://api.uala.com/api/v1/venues/" + ACCOUNT;
  var COLLECTIONS = [
    "appointments",
    "staff_members",
    "staff_member_treatments",
    "venue_treatments",
    "venue_treatment_groups",
    "vacancies",
    "custom_time_tables",
    "extra_openings",
  ];
  var IGNORED_STATES = { deleted: 1, discarded: 1, canceled: 1 };
  var opts = Object.assign({ days: 7, duration: null, service: null, staff: null, stepMin: 15, maxOptions: 6 }, input || {});
  if (!/^\d{4}-\d{2}-\d{2}$/.test(opts.from)) throw new Error("from debe ser AAAA-MM-DD");
  if (!(opts.days >= 1 && opts.days <= 60)) throw new Error("days entre 1 y 60");
  if (!opts.service && !(opts.duration >= 5 && opts.duration <= 600)) throw new Error("Indica service o duration en minutos");
  var auth = window.__phAuth;
  if (!auth || !(auth.authorization || auth["x-client-auth"])) throw new Error("Sin encabezados de sesión (window.__phAuth). Ejecuta antes la captura.");

  // --- utilidades ---
  var norm = function (s) { return String(s || "").normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().replace(/\s+/g, " ").trim(); };
  var pad = function (n) { return (n < 10 ? "0" : "") + n; };
  var offsetOf = function (dateKey) {
    var parts = new Intl.DateTimeFormat("en-US", { timeZone: TZ, timeZoneName: "longOffset" }).formatToParts(new Date(dateKey + "T12:00:00Z"));
    var m = /([+-])(\d{2}):(\d{2})/.exec(parts.find(function (p) { return p.type === "timeZoneName"; }).value);
    return m ? m[1] + m[2] + ":" + m[3] : "+00:00";
  };
  var wall = function (dateKey, hhmm) { return Date.parse(dateKey + "T" + (hhmm.length === 5 ? hhmm + ":00" : hhmm) + offsetOf(dateKey)); };
  var addDays = function (dateKey, n) { var d = new Date(dateKey + "T00:00:00Z"); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10); };
  var dowOf = function (dateKey) { return new Date(dateKey + "T00:00:00Z").getUTCDay(); }; // 0 = domingo, igual que time_table.day
  var fmtTime = function (ms) {
    var p = new Intl.DateTimeFormat("es-ES", { timeZone: TZ, hour: "2-digit", minute: "2-digit", hour12: false }).formatToParts(new Date(ms));
    return p.find(function (x) { return x.type === "hour"; }).value + ":" + p.find(function (x) { return x.type === "minute"; }).value;
  };
  var dayName = function (dateKey) { return new Intl.DateTimeFormat("es-ES", { timeZone: TZ, weekday: "short", day: "numeric", month: "short" }).format(new Date(dateKey + "T12:00:00Z")); };
  var fromKey = opts.from, lastKey = addDays(fromKey, opts.days - 1);
  var fromIso = fromKey + "T00:00:00" + offsetOf(fromKey), toIso = lastKey + "T23:59:59" + offsetOf(lastKey);

  // --- lectura: solo GET ---
  var headers = { accept: "application/json", authorization: auth.authorization, "x-client-auth": auth["x-client-auth"] };
  var get = async function (url) {
    var res = await fetch(url, { method: "GET", headers: headers, credentials: "omit", redirect: "error" });
    if (res.status === 401 || res.status === 403) throw new Error("Sesión caducada (" + res.status + ")");
    if (!res.ok) throw new Error("Treatwell respondió " + res.status + " en " + url);
    var body = await res.json();
    if (body.success !== true) throw new Error("Respuesta no válida en " + url);
    return body.data;
  };
  var readCollection = async function (collection) {
    var url = new URL(API + "/" + collection + ".json");
    if (collection === "appointments") { url.searchParams.set("from_time", fromIso); url.searchParams.set("to_time", toIso); }
    var data = await get(url.href);
    var rows = data[collection], count = data[collection + "_count"];
    if (!Array.isArray(rows) || (count != null && rows.length !== count)) throw new Error("Respuesta incompleta en " + collection);
    return rows;
  };
  var t0 = performance.now();
  var src = {};
  var results = await Promise.all([get(API + ".json")].concat(COLLECTIONS.map(async function (c) { src[c] = await readCollection(c); })));
  var venue = results[0].venue || results[0];
  var readMs = Math.round(performance.now() - t0);

  // --- servicio y profesionales capaces ---
  var staffAll = src.staff_members.map(function (s) { return { id: String(s.id), name: (s.full_name || ((s.first_name || "") + " " + (s.last_name || ""))).trim(), timeTable: s.time_table || [] }; });
  var wanted = staffAll, serviceInfo = null;
  var durationByStaff = {}, extraByStaff = {};
  if (opts.service) {
    var names = String(opts.service).split("/").map(norm), gName = names[0], vName = names[1];
    var group = src.venue_treatment_groups.find(function (g) { return norm(g.name) === gName; }) || src.venue_treatment_groups.find(function (g) { return norm(g.name).indexOf(gName) >= 0; });
    if (!group) throw new Error("Grupo de servicio no encontrado: " + opts.service);
    var variants = src.venue_treatments.filter(function (t) { return String(t.venue_treatment_group_id) === String(group.id); });
    var treatment = vName ? (variants.find(function (t) { return norm(t.name) === vName; }) || variants.find(function (t) { return norm(t.name).indexOf(vName) >= 0; })) : (variants.length === 1 ? variants[0] : null);
    if (!treatment) throw new Error("Variante no encontrada en " + group.name + ": " + variants.map(function (t) { return t.name; }).join(", "));
    var extraSec = (treatment.extra_time_after != null ? treatment.extra_time_after : group.extra_time_after) || 0;
    var applyExtra = treatment.always_apply_extra_time_after || group.always_apply_extra_time_after;
    src.staff_member_treatments.forEach(function (r) {
      if (String(r.venue_treatment_id) !== String(treatment.id) || r.state === "disabled") return;
      durationByStaff[String(r.staff_member_id)] = Math.round((r.duration || treatment.duration) / 60);
      extraByStaff[String(r.staff_member_id)] = applyExtra ? Math.round(extraSec / 60) : 0;
    });
    wanted = staffAll.filter(function (s) { return durationByStaff[s.id] != null; });
    serviceInfo = { group: group.name, variant: treatment.name, durationMin: Math.round(treatment.duration / 60), extraAfterMin: applyExtra ? Math.round(extraSec / 60) : 0, capable: wanted.map(function (s) { return s.name; }) };
  }
  if (opts.staff) wanted = wanted.filter(function (s) { return norm(s.name).indexOf(norm(opts.staff)) >= 0; });
  if (!wanted.length) throw new Error("Ninguna profesional disponible para ese servicio o nombre");

  // --- horarios ---
  var customByStaffDate = {};
  src.custom_time_tables.forEach(function (r) { customByStaffDate[r.staff_member_id + "|" + r.date] = r; });
  var extraByDate = {};
  src.extra_openings.forEach(function (r) { extraByDate[r.date] = r; });
  var toIntervals = function (dateKey, row) {
    if (!row || !row.start_time || !row.end_time) return [];
    var s = wall(dateKey, row.start_time), e = wall(dateKey, row.end_time);
    if (row.start_break && row.end_break) {
      var bs = wall(dateKey, row.start_break), be = wall(dateKey, row.end_break);
      return [[s, bs], [be, e]].filter(function (iv) { return iv[1] > iv[0]; });
    }
    return [[s, e]];
  };
  var shiftsFor = function (staff, dateKey) {
    var extra = extraByDate[dateKey];
    if (extra && !extra.start_time && !extra.end_time) return { intervals: [], source: "cierre extraordinario del salón" };
    var custom = customByStaffDate[staff.id + "|" + dateKey];
    if (custom) return { intervals: toIntervals(dateKey, custom), source: "horario especial de la profesional" };
    var weekly = staff.timeTable.find(function (d) { return d.day === dowOf(dateKey); });
    if (weekly && weekly.start_time) return { intervals: toIntervals(dateKey, weekly), source: "horario semanal de la profesional" };
    if (extra && extra.start_time) return { intervals: toIntervals(dateKey, extra), source: "apertura extraordinaria del salón" };
    var venueDay = (venue.time_table || []).find(function (d) { return d.day === dowOf(dateKey); });
    if (venueDay && venueDay.start_time) return { intervals: toIntervals(dateKey, venueDay), source: "horario del salón (la ficha de la profesional no define ese día; confirmar)" };
    return { intervals: [], source: "sin horario ese día" };
  };

  // --- ocupación: citas y bloqueos, sin datos de clientas ---
  var busyByStaff = {};
  var push = function (staffId, a, b, label) { (busyByStaff[staffId] = busyByStaff[staffId] || []).push({ a: a, b: b, label: label }); };
  var appointmentsCounted = 0;
  src.appointments.forEach(function (r) {
    if (IGNORED_STATES[r.state]) return;
    var emb = (r.data && r.data.staff_member_treatment) || {};
    var durSec = r.custom_duration != null ? r.custom_duration : (emb.total_duration != null ? emb.total_duration : (emb.duration || 0));
    var a = Date.parse(r.time), b = a + (durSec + (r.extra_time_staff_member_after || 0)) * 1000;
    push(String(r.staff_member_id), a, Math.max(b, a + 60000), "cita · " + (emb.name || "servicio"));
    appointmentsCounted++;
  });
  var blocksCounted = 0;
  src.vacancies.forEach(function (r) {
    var a, b;
    if (r.time == null && r.typology === "vacation") { a = wall(r.from_date.slice(0, 10), r.from_date.slice(11, 19)); b = wall(r.to_date.slice(0, 10), r.to_date.slice(11, 19)) + 1000; }
    else { a = Date.parse(r.time); b = a + (r.duration || 0) * 1000; }
    if (b <= Date.parse(fromIso) || a >= Date.parse(toIso)) return;
    push(String(r.staff_member_id), a, b, "bloqueo · " + (r.typology || "sin tipo") + (r.notes ? " · " + r.notes : ""));
    blocksCounted++;
  });

  // --- huecos ---
  var step = opts.stepMin * 60000, nowMs = Date.now();
  var days = [], options = [];
  for (var i = 0; i < opts.days; i++) {
    var dateKey = addDays(fromKey, i);
    var dayOut = { date: dateKey, label: dayName(dateKey), staff: [] };
    wanted.forEach(function (st) {
      var needMin = (opts.service ? durationByStaff[st.id] : opts.duration) + (extraByStaff[st.id] || 0);
      var needMs = needMin * 60000;
      var shift = shiftsFor(st, dateKey);
      var dayStart = wall(dateKey, "00:00"), dayEnd = wall(dateKey, "23:59");
      var busy = (busyByStaff[st.id] || []).filter(function (x) { return x.b > dayStart && x.a < dayEnd; }).sort(function (x, y) { return x.a - y.a; });
      var free = [];
      shift.intervals.forEach(function (iv) {
        var cursor = iv[0], e = iv[1];
        busy.forEach(function (x) {
          if (x.b <= cursor || x.a >= e) return;
          if (x.a > cursor) free.push([cursor, Math.min(x.a, e)]);
          cursor = Math.max(cursor, x.b);
        });
        if (cursor < e) free.push([cursor, e]);
      });
      dayOut.staff.push({
        name: st.name, needMin: needMin, source: shift.source,
        shift: shift.intervals.map(function (iv) { return fmtTime(iv[0]) + "–" + fmtTime(iv[1]); }).join(", ") || "no trabaja",
        busy: busy.map(function (x) { return fmtTime(x.a) + "–" + fmtTime(x.b) + " " + x.label; }),
        free: free.map(function (iv) { return fmtTime(iv[0]) + "–" + fmtTime(iv[1]) + " (" + Math.round((iv[1] - iv[0]) / 60000) + " min)"; }),
      });
      free.forEach(function (iv) {
        if (iv[1] - iv[0] < needMs || iv[1] <= nowMs) return;
        var start = Math.ceil(Math.max(iv[0], nowMs) / step) * step;
        while (start + needMs <= iv[1] && options.length < 500) {
          options.push({ date: dateKey, label: dayName(dateKey), staff: st.name, start: fmtTime(start), end: fmtTime(start + needMs), ms: start });
          start += step;
        }
      });
    });
    days.push(dayOut);
  }
  options.sort(function (x, y) { return x.ms - y.ms; });
  var seen = {}, firstOptions = [];
  for (var k = 0; k < options.length && firstOptions.length < opts.maxOptions; k++) {
    var o = options[k], key = o.date + "|" + o.staff;
    if (seen[key]) continue;
    seen[key] = true;
    firstOptions.push({ date: o.date, label: o.label, staff: o.staff, start: o.start, end: o.end });
  }
  return {
    readOnly: true, account: ACCOUNT, from: fromKey, to: lastKey,
    service: serviceInfo, durationMin: opts.service ? null : opts.duration,
    staffConsidered: wanted.map(function (s) { return s.name; }), readMs: readMs,
    counts: { appointments: appointmentsCounted, blocks: blocksCounted, staff: staffAll.length },
    firstOptions: firstOptions, totalCandidateStarts: options.length, days: days,
  };
};
```

Ejemplo de llamada:

```javascript
const r = await window.__polihairTreatwellHuecos({
  from: "2026-10-22",
  days: 10,
  service: "Corte en seco + Definición / Densidad media",
  staff: null,
  stepMin: 15,
  maxOptions: 6,
});
JSON.stringify({ service: r.service, readMs: r.readMs, firstOptions: r.firstOptions, days: r.days });
```

Parámetros:

| Campo | Significado |
| --- | --- |
| `from` | Primer día a revisar, en Europe/Madrid. |
| `days` | Días consecutivos a revisar, de 1 a 60. |
| `service` | «Grupo / Variante» tal como aparecen en Treatwell. Fija la duración por profesional y filtra quién puede hacerlo. Sin acentos ni mayúsculas exactas. |
| `duration` | Alternativa a `service`: minutos necesarios sin filtrar por capacidad. |
| `staff` | Nombre parcial de una profesional, o `null` para todas. |
| `stepMin` | Rejilla de inicios para las opciones. |
| `maxOptions` | Máximo de primeras opciones devueltas, una por día y profesional. |

## Cómo leer el resultado

- `service.capable`: profesionales que tienen asignado ese servicio. Si una profesional no aparece, no la ofrezcas aunque tenga la tarde libre.
- `firstOptions`: primera hora válida por día y profesional, ya ordenadas. Ofrece dos o tres, no la lista entera.
- `days[].staff[]`: por día y profesional, el turno aplicado (`shift`), la ocupación (`busy`, sin nombres de clientas) y los huecos libres (`free`) con su duración.
- `days[].staff[].source` indica de dónde sale el turno. Si dice «horario del salón (la ficha de la profesional no define ese día; confirmar)», abre ese día en la agenda con la URL y confirma a ojo que la columna no está rayada antes de ofrecer la hora.
- `readMs` suele estar entre uno y tres segundos. Si el resultado llega con error de «Sesión caducada», vuelve al paso 2 o pide iniciar sesión.

Las opciones ya incluyen la duración de la profesional para esa variante y el tiempo extra posterior cuando Treatwell lo aplica siempre. No añadas márgenes por tu cuenta ni los quites.

## Paso 4. Registrar en los dos sistemas, en este orden

1. Con la hora acordada y la autorización de la usuaria, crea la cita en Treatwell desde su interfaz: abre el día con la URL, pulsa el hueco en la columna de la profesional, elige clienta, servicio y hora, y guarda. Comprueba que la tarjeta aparece en la columna y hora correctas.
2. Después refléjala en poliHair con `booking_create`, con la misma profesional, variante, fecha y hora, y `origin: "treatwell"` para que quede claro de dónde viene. Si la reserva nace de un chat, incluye `conversationId`. No envíes confirmaciones desde poliHair sin autorización expresa: durante la convivencia los recordatorios los manda Treatwell.
3. Si poliHair rechaza la creación por solapamiento, no fuerces nada: la cita ya existe en poliHair o hay otra en ese tramo. Léela con `calendar_get_day` y resuélvelo con la usuaria.
4. Para mover o cancelar, el mismo orden: primero Treatwell, después `booking_reschedule` o `booking_cancel` en poliHair sobre la cita equivalente, localizada por clienta, día y hora.

Si el paso 1 se hizo y el paso 2 falla, díselo a la usuaria tal cual: la cita existe en Treatwell y falta en poliHair. No des por hecho que la volverás a crear sin avisar.

## Paso 5. Preparar una escritura sin enviarla

Antes de crear, mover o eliminar en la interfaz, puedes preparar la operación con el borrador. Resuelve la clienta por nombre o teléfono, comprueba que la profesional hace el servicio, que la hora cae dentro de su turno, que es futura y que no se solapa con citas ni bloqueos, y devuelve las peticiones que harían falta. **No envía nada y no puede enviar nada**: su única llamada de red es de lectura con el método fijado en el código. Las formas de alta de clienta, creación, cambio de hora y eliminación se observaron en una secuencia manual real el 21 de octubre de 2026; cada petición indica si fue observada o solo deducida del código del panel.

Acciones: `client_create` (alta de clienta, comprueba duplicados por nombre y teléfono), `create`, `reschedule`, `discard` (lo que hace el botón eliminar del panel) y `cancel` (no observada).

Define la función ejecutando este bloque en la misma pestaña, después de la captura del paso 2:

```javascript
window.__polihairTreatwellBorrador = async function (input) {
  var TZ = "Europe/Madrid";
  var ACCOUNT = "67426";
  var API = "https://api.uala.com/api/v1/venues/" + ACCOUNT;
  var IGNORED_STATES = { deleted: 1, discarded: 1, canceled: 1 };
  var opts = Object.assign({ action: "create", client: null, clientId: null, phone: null, email: null, service: null, staff: null, date: null, time: null, notes: "", appointmentId: null, newDate: null, newTime: null, newStaff: null }, input || {});
  var auth = window.__phAuth;
  if (!auth || !(auth.authorization || auth["x-client-auth"])) throw new Error("Sin encabezados de sesión (window.__phAuth). Ejecuta antes la captura.");
  if (["create", "reschedule", "cancel", "discard", "client_create"].indexOf(opts.action) < 0) throw new Error("action debe ser create, reschedule, cancel, discard o client_create");

  // --- utilidades ---
  var norm = function (s) { return String(s || "").normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().replace(/\s+/g, " ").trim(); };
  var offsetOf = function (dateKey) {
    var parts = new Intl.DateTimeFormat("en-US", { timeZone: TZ, timeZoneName: "longOffset" }).formatToParts(new Date(dateKey + "T12:00:00Z"));
    var m = /([+-])(\d{2}):(\d{2})/.exec(parts.find(function (p) { return p.type === "timeZoneName"; }).value);
    return m ? m[1] + m[2] + ":" + m[3] : "+00:00";
  };
  var isoAt = function (dateKey, hhmm) { return dateKey + "T" + (hhmm.length === 5 ? hhmm + ":00" : hhmm) + offsetOf(dateKey); };
  var wall = function (dateKey, hhmm) { return Date.parse(isoAt(dateKey, hhmm)); };
  var dowOf = function (dateKey) { return new Date(dateKey + "T00:00:00Z").getUTCDay(); };
  var fmtTime = function (ms) {
    var p = new Intl.DateTimeFormat("es-ES", { timeZone: TZ, hour: "2-digit", minute: "2-digit", hour12: false }).formatToParts(new Date(ms));
    return p.find(function (x) { return x.type === "hour"; }).value + ":" + p.find(function (x) { return x.type === "minute"; }).value;
  };
  var dateOk = function (d) { return /^\d{4}-\d{2}-\d{2}$/.test(d || ""); };
  var timeOk = function (t) { return /^\d{2}:\d{2}$/.test(t || ""); };

  // --- única llamada de red: GET, método fijado ---
  var headers = { accept: "application/json", authorization: auth.authorization, "x-client-auth": auth["x-client-auth"] };
  var get = async function (path, params) {
    var url = new URL(API + path);
    Object.keys(params || {}).forEach(function (k) { url.searchParams.set(k, params[k]); });
    var res = await fetch(url.href, { method: "GET", headers: headers, credentials: "omit", redirect: "error" });
    if (res.status === 401 || res.status === 403) throw new Error("Sesión caducada (" + res.status + ")");
    if (!res.ok) throw new Error("Treatwell respondió " + res.status + " en " + path);
    var body = await res.json();
    if (body.success !== true) throw new Error("Respuesta no válida en " + path);
    return body.data;
  };

  var checks = [];
  var check = function (name, ok, detail) { checks.push({ name: name, ok: !!ok, detail: detail || "" }); return !!ok; };
  var result = { dryRun: true, sent: false, canSend: false, action: opts.action, account: ACCOUNT, resolved: {}, checks: checks, requests: [], summary: "" };

  // --- alta de clienta: solo comprueba duplicados y devuelve la petición ---
  if (opts.action === "client_create") {
    var fullName = String(opts.client || "").trim();
    if (!check("nombre indicado", fullName.length >= 3, fullName || "vacío")) { result.summary = "No se puede preparar: falta el nombre."; return result; }
    var dupes = (await get("/customers/search.json", { q: fullName })).customers || [];
    var sameName = dupes.filter(function (c) { return norm(c.full_name) === norm(fullName); });
    check("sin ficha con el mismo nombre", sameName.length === 0, sameName.length ? sameName.length + " ficha(s) ya existen con ese nombre; usa clientId" : "ninguna coincidencia exacta");
    if (opts.phone) {
      var byPhone = (await get("/customers/search.json", { q: String(opts.phone) })).customers || [];
      check("sin ficha con el mismo teléfono", byPhone.length === 0, byPhone.length ? byPhone.length + " ficha(s) con ese teléfono" : "ninguna");
    }
    var customerBody = { full_name: fullName };
    if (opts.phone) customerBody.phone = String(opts.phone);
    if (opts.email) customerBody.email = String(opts.email);
    result.resolved.client = { name: fullName, hasPhone: !!opts.phone, hasEmail: !!opts.email };
    result.requests.push({ step: 1, method: "POST", url: API + "/customers.json", body: { customer: customerBody }, note: "observada el 21/10/2026 con full_name; phone y email no observados. Treatwell activa notificaciones por defecto." });
    result.canSend = checks.every(function (c) { return c.ok; });
    result.summary = (result.canSend ? "Prepararía " : "NO prepararía ") + "el alta de la clienta " + fullName + ". No se ha enviado nada.";
    return result;
  }

  // --- día y hora objetivo ---
  var isEnd = opts.action === "cancel" || opts.action === "discard";
  var targetDate = opts.action === "reschedule" ? opts.newDate : opts.date;
  var targetTime = opts.action === "reschedule" ? opts.newTime : opts.time;
  var targetStaffName = opts.action === "reschedule" ? (opts.newStaff || opts.staff) : opts.staff;
  if (!isEnd) {
    if (!dateOk(targetDate) || !timeOk(targetTime)) throw new Error("date/time (o newDate/newTime) en formato AAAA-MM-DD y HH:MM");
  }
  var lookupDate = opts.date;
  if (opts.action !== "create" && !opts.appointmentId && !dateOk(lookupDate)) throw new Error("Para localizar la cita indica appointmentId o date (y time) de la cita actual");

  // --- lecturas ---
  var datesToRead = [];
  if (dateOk(lookupDate)) datesToRead.push(lookupDate);
  if (dateOk(targetDate) && datesToRead.indexOf(targetDate) < 0) datesToRead.push(targetDate);
  var reads = await Promise.all([
    get(".json"),
    get("/staff_members.json"),
    get("/staff_member_treatments.json"),
    get("/venue_treatments.json"),
    get("/venue_treatment_groups.json"),
    get("/vacancies.json"),
    get("/custom_time_tables.json"),
    get("/extra_openings.json"),
  ].concat(datesToRead.map(function (d) { return get("/appointments.json", { from_time: isoAt(d, "00:00"), to_time: isoAt(d, "23:59:59") }); })));
  var venue = reads[0].venue || reads[0];
  var staffAll = reads[1].staff_members.map(function (s) { return { id: String(s.id), name: (s.full_name || ((s.first_name || "") + " " + (s.last_name || ""))).trim(), timeTable: s.time_table || [] }; });
  var smt = reads[2].staff_member_treatments, vts = reads[3].venue_treatments, groups = reads[4].venue_treatment_groups;
  var vacancies = reads[5].vacancies, customTT = reads[6].custom_time_tables, extraOpen = reads[7].extra_openings;
  var appointmentsByDate = {};
  datesToRead.forEach(function (d, i) { appointmentsByDate[d] = reads[8 + i].appointments; });

  // --- cita existente (mover / cancelar) ---
  var existing = null;
  if (opts.action !== "create") {
    var pool = appointmentsByDate[lookupDate] || [];
    if (opts.appointmentId) existing = pool.find(function (a) { return String(a.id) === String(opts.appointmentId); }) || null;
    else {
      var cands = pool.filter(function (a) {
        if (IGNORED_STATES[a.state]) return false;
        if (timeOk(opts.time) && Date.parse(a.time) !== wall(lookupDate, opts.time)) return false;
        if (opts.client && norm(a.customer_full_name).indexOf(norm(opts.client)) < 0) return false;
        if (opts.staff) { var st = staffAll.find(function (s) { return String(s.id) === String(a.staff_member_id); }); if (!st || norm(st.name).indexOf(norm(opts.staff)) < 0) return false; }
        return true;
      });
      if (cands.length === 1) existing = cands[0];
      else {
        result.needsDecision = "cita";
        result.candidates = cands.map(function (a) { var emb = (a.data && a.data.staff_member_treatment) || {}; return { appointmentId: String(a.id), client: a.customer_full_name, time: fmtTime(Date.parse(a.time)), service: emb.name || null, state: a.state }; });
        check("cita localizada", false, cands.length === 0 ? "ninguna cita coincide ese día" : cands.length + " citas coinciden; indica appointmentId");
        result.summary = "No se puede preparar: la cita no queda identificada de forma única.";
        return result;
      }
    }
    if (!existing) { check("cita localizada", false, "no existe con ese id ese día"); result.summary = "No se puede preparar: cita no encontrada."; return result; }
    var embx = (existing.data && existing.data.staff_member_treatment) || {};
    result.resolved.appointment = { id: String(existing.id), client: existing.customer_full_name, customerId: String(existing.customer_id), staffId: String(existing.staff_member_id), time: fmtTime(Date.parse(existing.time)), date: lookupDate, service: embx.name || null, durationMin: Math.round(((existing.custom_duration != null ? existing.custom_duration : (embx.total_duration || embx.duration || 0)) + (existing.extra_time_staff_member_after || 0)) / 60), state: existing.state };
    check("cita localizada", true, result.resolved.appointment.client + " · " + result.resolved.appointment.time + " · " + (result.resolved.appointment.service || "sin servicio"));
    check("estado modificable", !IGNORED_STATES[existing.state], "estado " + existing.state);
  }

  // --- cancelar o eliminar: solo una petición ---
  if (isEnd) {
    if (opts.action === "discard") result.requests.push({ step: 1, method: "PUT", url: API + "/appointments/" + existing.id + "/discard.json", body: {}, note: "observada el 21/10/2026: es lo que hace el botón eliminar del panel (estado discarded)" });
    else result.requests.push({ step: 1, method: "PUT", url: API + "/appointments/" + existing.id + "/cancel.json", body: {}, note: "NO observada; deducida del código del panel. El botón eliminar usa discard.json" });
    result.canSend = checks.every(function (c) { return c.ok; });
    var verb = opts.action === "discard" ? "Eliminaría" : "Cancelaría";
    result.summary = (result.canSend ? verb : "NO " + verb.toLowerCase()) + " la cita de " + result.resolved.appointment.client + " el " + lookupDate + " a las " + result.resolved.appointment.time + ". No se ha enviado nada.";
    return result;
  }

  // --- clienta (crear) ---
  var customer = null;
  if (opts.action === "create") {
    if (opts.clientId) {
      customer = { id: String(opts.clientId), name: "(id indicado)" };
    } else {
      if (!opts.client) throw new Error("Indica client (nombre o teléfono) o clientId");
      var found = (await get("/customers/search.json", { q: String(opts.client) })).customers || [];
      var exact = found.filter(function (c) { return norm(c.full_name) === norm(opts.client); });
      var pick = exact.length === 1 ? exact : found;
      if (pick.length === 1) customer = { id: String(pick[0].id), name: pick[0].full_name, hasPhone: !!pick[0].phone };
      else {
        result.needsDecision = "clienta";
        result.candidates = found.slice(0, 10).map(function (c) { return { clientId: String(c.id), name: c.full_name, hasPhone: !!c.phone, hasEmail: !!c.email }; });
        check("clienta única", false, found.length === 0 ? "sin coincidencias; crear ficha es otra decisión" : found.length + " coincidencias; indica clientId");
        result.summary = "No se puede preparar: la clienta no queda identificada de forma única.";
        return result;
      }
    }
    result.resolved.client = customer;
    check("clienta única", true, customer.name);
  } else {
    customer = { id: String(existing.customer_id), name: existing.customer_full_name };
    result.resolved.client = customer;
  }

  // --- profesional ---
  var staff = null;
  if (targetStaffName) {
    var sm = staffAll.filter(function (s) { return norm(s.name).indexOf(norm(targetStaffName)) >= 0; });
    if (sm.length !== 1) { check("profesional única", false, sm.map(function (s) { return s.name; }).join(", ") || "sin coincidencia"); result.summary = "No se puede preparar: profesional no identificada."; return result; }
    staff = sm[0];
  } else if (existing) {
    staff = staffAll.find(function (s) { return String(s.id) === String(existing.staff_member_id); });
  }
  if (!staff) throw new Error("Indica staff");
  result.resolved.staff = { id: staff.id, name: staff.name };
  check("profesional única", true, staff.name);

  // --- servicio y duración ---
  var treatment = null, assignment = null, needMin = 0;
  if (opts.action === "create") {
    if (!opts.service) throw new Error("Indica service como Grupo / Variante");
    var names = String(opts.service).split("/").map(norm), gName = names[0], vName = names[1];
    var group = groups.find(function (g) { return norm(g.name) === gName; }) || groups.find(function (g) { return norm(g.name).indexOf(gName) >= 0; });
    if (!group) { check("servicio", false, "grupo no encontrado"); result.summary = "No se puede preparar: servicio no encontrado."; return result; }
    var variants = vts.filter(function (t) { return String(t.venue_treatment_group_id) === String(group.id); });
    treatment = vName ? (variants.find(function (t) { return norm(t.name) === vName; }) || variants.find(function (t) { return norm(t.name).indexOf(vName) >= 0; })) : (variants.length === 1 ? variants[0] : null);
    if (!treatment) { check("servicio", false, "variantes: " + variants.map(function (t) { return t.name; }).join(", ")); result.summary = "No se puede preparar: variante no identificada."; return result; }
    assignment = smt.find(function (r) { return String(r.venue_treatment_id) === String(treatment.id) && String(r.staff_member_id) === staff.id && r.state !== "disabled"; }) || null;
    check("servicio", true, group.name + " / " + treatment.name);
    if (!check("profesional capacitada", !!assignment, assignment ? "asignación " + assignment.id : staff.name + " no tiene asignado este servicio")) { result.summary = "No se puede preparar: la profesional no hace ese servicio."; return result; }
    var extraSec = (treatment.extra_time_after != null ? treatment.extra_time_after : group.extra_time_after) || 0;
    var applyExtra = treatment.always_apply_extra_time_after || group.always_apply_extra_time_after;
    needMin = Math.round((assignment.duration || treatment.duration) / 60) + (applyExtra ? Math.round(extraSec / 60) : 0);
    result.resolved.service = { group: group.name, variant: treatment.name, treatmentId: String(treatment.id), assignmentId: String(assignment.id), durationMin: needMin, priceEuros: assignment.price != null ? assignment.price : treatment.price };
  } else {
    needMin = result.resolved.appointment.durationMin || 0;
    if (opts.newStaff) {
      var tid = existing.staff_member_treatment_id;
      var a0 = smt.find(function (r) { return String(r.id) === String(tid); });
      var vtid = a0 ? a0.venue_treatment_id : ((existing.data && existing.data.staff_member_treatment) || {}).venue_treatment_id;
      var a1 = smt.find(function (r) { return String(r.venue_treatment_id) === String(vtid) && String(r.staff_member_id) === staff.id && r.state !== "disabled"; });
      check("profesional capacitada", !!a1, a1 ? "asignación " + a1.id : "la nueva profesional no tiene asignado el servicio de la cita");
      if (a1 && a1.duration) needMin = Math.round(a1.duration / 60) + Math.round((existing.extra_time_staff_member_after || 0) / 60);
      result.resolved.newAssignmentId = a1 ? String(a1.id) : null;
    }
  }

  // --- turno y ocupación del día objetivo ---
  var startMs = wall(targetDate, targetTime), endMs = startMs + needMin * 60000;
  var toIntervals = function (dateKey, row) {
    if (!row || !row.start_time || !row.end_time) return [];
    var s = wall(dateKey, row.start_time), e = wall(dateKey, row.end_time);
    if (row.start_break && row.end_break) { var bs = wall(dateKey, row.start_break), be = wall(dateKey, row.end_break); return [[s, bs], [be, e]].filter(function (iv) { return iv[1] > iv[0]; }); }
    return [[s, e]];
  };
  var shift = (function () {
    var extra = extraOpen.find(function (r) { return r.date === targetDate; });
    if (extra && !extra.start_time && !extra.end_time) return { intervals: [], source: "cierre extraordinario del salón" };
    var custom = customTT.find(function (r) { return String(r.staff_member_id) === staff.id && r.date === targetDate; });
    if (custom) return { intervals: toIntervals(targetDate, custom), source: "horario especial de la profesional" };
    var weekly = staff.timeTable.find(function (d) { return d.day === dowOf(targetDate); });
    if (weekly && weekly.start_time) return { intervals: toIntervals(targetDate, weekly), source: "horario semanal de la profesional" };
    if (extra && extra.start_time) return { intervals: toIntervals(targetDate, extra), source: "apertura extraordinaria del salón" };
    var venueDay = (venue.time_table || []).find(function (d) { return d.day === dowOf(targetDate); });
    if (venueDay && venueDay.start_time) return { intervals: toIntervals(targetDate, venueDay), source: "horario del salón (la ficha de la profesional no define ese día; confirmar)" };
    return { intervals: [], source: "sin horario ese día" };
  })();
  var inShift = shift.intervals.some(function (iv) { return startMs >= iv[0] && endMs <= iv[1]; });
  check("dentro del turno", inShift, shift.source + " · " + (shift.intervals.map(function (iv) { return fmtTime(iv[0]) + "–" + fmtTime(iv[1]); }).join(", ") || "no trabaja"));
  check("hora futura", startMs > Date.now(), fmtTime(startMs) + " del " + targetDate);
  var busy = [];
  (appointmentsByDate[targetDate] || []).forEach(function (a) {
    if (IGNORED_STATES[a.state] || String(a.staff_member_id) !== staff.id) return;
    if (existing && String(a.id) === String(existing.id)) return;
    var emb = (a.data && a.data.staff_member_treatment) || {};
    var durSec = a.custom_duration != null ? a.custom_duration : (emb.total_duration != null ? emb.total_duration : (emb.duration || 0));
    var s = Date.parse(a.time), e = Math.max(s + (durSec + (a.extra_time_staff_member_after || 0)) * 1000, s + 60000);
    busy.push({ s: s, e: e, label: "cita · " + (emb.name || "servicio") });
  });
  vacancies.forEach(function (r) {
    if (String(r.staff_member_id) !== staff.id) return;
    var s, e;
    if (r.time == null && r.typology === "vacation") { s = wall(r.from_date.slice(0, 10), r.from_date.slice(11, 19)); e = wall(r.to_date.slice(0, 10), r.to_date.slice(11, 19)) + 1000; }
    else { s = Date.parse(r.time); e = s + (r.duration || 0) * 1000; }
    busy.push({ s: s, e: e, label: "bloqueo · " + (r.typology || "sin tipo") });
  });
  var clashes = busy.filter(function (b) { return b.s < endMs && b.e > startMs; });
  check("sin solapamiento", clashes.length === 0, clashes.map(function (b) { return fmtTime(b.s) + "–" + fmtTime(b.e) + " " + b.label; }).join("; ") || "libre de " + fmtTime(startMs) + " a " + fmtTime(endMs));
  result.resolved.slot = { date: targetDate, start: fmtTime(startMs), end: fmtTime(endMs), durationMin: needMin, timeIso: isoAt(targetDate, targetTime) };

  // --- peticiones que harían falta (no se envían) ---
  if (opts.action === "create") {
    result.requests.push({ step: 1, method: "POST", url: API + "/appointments.json", body: { appointment: { staff_member_id: Number(staff.id), time: isoAt(targetDate, targetTime) } }, note: "observada el 21/10/2026: crea un hueco en estado requested sin clienta ni servicio; el id lo devuelve la respuesta" });
    result.requests.push({ step: 2, method: "PUT", url: API + "/appointments/{id}/book.json", body: { appointment: { id: "{id}", time: isoAt(targetDate, targetTime), customer_id: Number(customer.id), staff_member_id: Number(staff.id), staff_member_treatment_id: Number(assignment.id), customer_full_name: customer.name, workstation_id: null, customers_count: 1 }, next: true }, note: "observada el 21/10/2026: pasa la cita a booked con clienta y servicio. El panel no envía notas ni cancellation_protection en este paso" });
  } else {
    var body = { appointment: { id: Number(existing.id), time: targetDate + "T" + targetTime + ":00", staff_member_id: Number(staff.id), staff_member_treatment_id: Number(result.resolved.newAssignmentId || existing.staff_member_treatment_id) }, next: true };
    result.requests.push({ step: 1, method: "PUT", url: API + "/appointments/" + existing.id + ".json", body: body, note: "observada el 21/10/2026 para un cambio de hora: la hora va SIN desfase horario y se interpreta en la zona del salón. Cambio de profesional no observado" });
  }
  result.canSend = checks.every(function (c) { return c.ok; });
  var who = customer.name + " con " + staff.name;
  result.summary = (result.canSend ? "Prepararía " : "NO prepararía ") + (opts.action === "create" ? "la creación de " + result.resolved.service.group + " / " + result.resolved.service.variant + " para " : "el cambio de la cita de ") + who + " el " + targetDate + " de " + result.resolved.slot.start + " a " + result.resolved.slot.end + ". No se ha enviado nada.";
  return result;
};
```

Ejemplos:

```javascript
// Crear
const b = await window.__polihairTreatwellBorrador({
  action: "create",
  client: "Nombre Apellido",            // o clientId: "123456"
  service: "Corte en seco + Definición / Densidad media",
  staff: "Loli",
  date: "2026-10-26",
  time: "15:00",
  notes: "",
});
JSON.stringify({ summary: b.summary, canSend: b.canSend, checks: b.checks, resolved: b.resolved, requests: b.requests, candidates: b.candidates });

// Mover: localiza la cita por clienta, fecha y hora actuales
await window.__polihairTreatwellBorrador({ action: "reschedule", client: "Nombre Apellido", date: "2026-10-26", time: "15:00", newDate: "2026-10-27", newTime: "16:00", newStaff: null });

// Eliminar (botón eliminar del panel): por appointmentId o por clienta, fecha y hora
await window.__polihairTreatwellBorrador({ action: "discard", client: "Nombre Apellido", date: "2026-10-26", time: "15:00" });

// Alta de clienta nueva, solo con lo que la clienta haya dado
await window.__polihairTreatwellBorrador({ action: "client_create", client: "Nombre Apellido", phone: null, email: null });
```

Cómo usar el resultado:

- `checks`: cada comprobación con `ok` y detalle. Si alguna falla, `canSend` es `false` y no se debe registrar nada hasta resolverla con la usuaria.
- `needsDecision` con `candidates`: varias clientas o varias citas coinciden. Pregunta y repite con `clientId` o `appointmentId`. Nunca elijas por tu cuenta.
- `resolved`: clienta, profesional, servicio con duración y precio, y la franja exacta. Es lo que hay que reproducir en la interfaz de Treatwell al registrar.
- `requests`: las peticiones que el panel haría, con una nota que dice si esa forma fue observada en la prueba real o solo deducida. Son informativas; no las envíes con ninguna herramienta. Si en el futuro se autoriza un envío, hace falta una decisión explícita de la propietaria y un cambio del ADR.
- Tras registrar en la interfaz, vuelve a ejecutar el script de huecos del paso 3 para ese día y confirma que la cita aparece en la columna y hora esperadas.

## Puesta al día eventual

Cuando se pida «pasa a poliHair las citas de Treatwell de tal semana», acota la tarea por semana y profesional, lee las citas con el script (campo `busy` de cada día y, si hace falta, el detalle de la cita en la interfaz para la clienta) resuelve todas las clientas de la semana con una sola llamada a `clients_resolve` (teléfonos o nombres completos), crea las fichas que falten y después crea en poliHair solo las citas que falten según `calendar_get_day`, con `operations_batch` si son muchas. Un repaso repetido no duplica: poliHair rechaza dos citas confirmadas de la misma profesional en el mismo tramo.

## Límites

- Es una integración de lectura sobre la API interna que usa el panel. Treatwell puede cambiarla sin aviso; si las respuestas dejan de validar, el script falla con un mensaje y no hay que inventar huecos.
- La ventana entre leer y registrar no es cero: una reserva del marketplace puede entrar en medio. Al crear en Treatwell, su propia agenda avisa del solapamiento; respétalo.
- El script no muestra nombres de clientas en la ocupación. Para identificar a una clienta usa la interfaz o `clients_search` en poliHair.
