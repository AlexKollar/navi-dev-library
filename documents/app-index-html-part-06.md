# app/index.html — part 6 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 9259–11338 of app/index.html.

    peers: (b.peers || []).map(function (p) { var t = p.tunnel || {}; return { name: p.name || "Unnamed peer", nodeId: p.nodeId || "", short: String(p.nodeId || "").slice(0, 12), canBorrow: p.canBorrow !== false, reachable: null, status: "unknown", lastProbeError: "", lastProbeReason: "", lastProbeAgeSec: null, balanceSec: 0, lastTs: 0, publicEndpoint: p.publicEndpoint || "", tunnel: { enabled: !!t.enabled, sshHost: t.sshHost || "", sshPort: t.sshPort || 227, sshUser: t.sshUser || "", localPort: t.localPort || null }, hasTunnelKey: !!p.tunnelPublicKey, tunnelKeyAuthorized: false, tunnelState: "off", tunnelSentence: "", tunnelSuggestion: p.tunnelSuggestion || { sshHost: "" }, relayCarryTo: p.relayCarryTo || [], relayAcceptFrom: p.relayAcceptFrom || [], relayWillCarryTo: p.relayWillCarryTo || [] }; })
  };
}
function meshDotColor(m) {
  // Build .87: the payload's summary state drives the dot, in
  // the status column's established dot language only — grey
  // off / no peers, green reachable, amber borrowing/lending or
  // any warning (an unreachable OR a dropped peer; the row's
  // line carries which, in words). No new dot color is invented
  // for Mesh: the warning surfaces through the same language
  // every other Servers row speaks. Payloads from before .87
  // (no state field) fall through to the older derivation.
  if (!m.enabled) return "#9fb3c8";
  if (m.state === "warning") return "#f5b544";
  if (m.state === "no-peers") return "#9fb3c8";
  if (m.borrowing || m.lendingNow) return "#f5b544";
  if (m.peers && m.peers.length && m.peers.every(function (p) { return p.reachable === false; })) return "#f5b544";
  return "#34d399";
}
function meshFmtSec(sec) {
  var s = Math.round(sec || 0);
  return (s > 0 ? "+" : s < 0 ? "−" : "") + Math.abs(s) + "s";
}
function meshAgeText(sec) {
  // Build 7 (#87): the age of a peer's last probe, the way the
  // roster speaks it — "42s", "3m", "2h" — inside "(last checked
  // … ago)". Null (never probed) renders nothing.
  if (sec == null) return "";
  var s = Math.max(0, Math.round(sec));
  if (s < 60) return s + "s";
  if (s < 3600) return Math.round(s / 60) + "m";
  return Math.round(s / 3600) + "h";
}
function meshUnreachableText(p) {
  // Build 7 (#87): a failed check keeps its reason and its age —
  // "unreachable — unauthorized (last checked 42s ago)" — and
  // never decays back to "not checked yet"; that phrase is
  // reserved for a peer no probe has ever completed for.
  var t = "unreachable";
  if (p.lastProbeError) t += " — " + p.lastProbeError;
  var age = meshAgeText(p.lastProbeAgeSec);
  if (age) t += " (last checked " + age + " ago)";
  return t;
}
function meshPillHtml(mesh) {
  // The Mesh attribution pill itself (Build 3), BARE since
  // Build 8 (#88): pillRow() owns the one row under a reply and
  // puts this pill first in it, ahead of any source pills.
  //
  // Build 2 (#203): when the answer travelled inside a
  // federation — the bridge records the fedinet the borrow was
  // authorized under on the payload — the pill turns the
  // federation green and reads the whole recorded route:
  // "Fedinet > Answered by Name", with the carrier as the
  // middle segment when a relay carried it ("Fedinet >
  // Carrier > Answered by Name"). It renders the recorded
  // route only: no federation stamp, no green, and the
  // pairwise pill below keeps its exact old shape.
  if (mesh.fedinet) {
    var mid = mesh.viaRelay
      ? "<span class='fedseg fedseg-mid'> &gt; " + esc(mesh.viaRelay) + "</span><span class='fedseg fedseg-elide' aria-hidden='true'> &gt; \u2026</span>"
      : "";
    var label = "Answered by " + (mesh.node || "a paired machine") + " via the " + mesh.fedinet + " fedinet" + (mesh.viaRelay ? ", carried by " + mesh.viaRelay : "");
    return "<span class='meshpill fedpill' aria-label='" + esc(label) + "'><svg aria-hidden='true' focusable='false' width='11' height='11' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'>" + meshGlyphSvg() + "</svg>" + esc(mesh.fedinet) + mid + " &gt; Answered by " + esc(mesh.node || "a paired machine") + "</span>";
  }
  return "<span class='meshpill' aria-label='Answered by " + esc(mesh.node || "a paired machine") + " via Apex | Mesh'><svg aria-hidden='true' focusable='false' width='11' height='11' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'>" + meshGlyphSvg() + "</svg>via " + esc(mesh.node || "a paired machine") + "</span>";
}
function pillRow(mesh, sources) {
  // Build 8 (#88): ONE row under a reply carries every pill —
  // the Mesh attribution FIRST, then the source pills. Before,
  // each kind rendered its own wrapping .srcs row, so a reply
  // that was both borrowed and pack-informed stacked two rows
  // with the Mesh pill on the bottom — the field report that
  // filed #88. The row never wraps (flex-nowrap): on a phone the
  // pills shrink and ellipsize inside it instead of pushing the
  // page sideways or stacking.
  var pills = "";
  if (mesh && mesh.servedBy === "mesh") pills += meshPillHtml(mesh);
  pills += srcPills(sources);
  if (!pills) return "";
  var hasMesh = !!(mesh && mesh.servedBy === "mesh");
  var hasSrc = !!(sources && sources.length);
  var label = hasMesh && hasSrc ? "Answered by a paired machine — sources" : hasMesh ? "Answered by a paired machine" : "Sources";
  return "<div class='srcs pillrow' aria-label='" + label + "'>" + pills + "</div>";
}
function meshLineHtml(mesh) {
  // The quiet line under a reply Mesh touched (build .85): who
  // answered, or why this machine answered instead. Same register
  // as the source pills — metadata, never part of the reply.
  // Build 3: a mesh-served reply carries its attribution as a
  // PILL — Hexlink glyph + "via <name>" in Mesh Cyan, one glance,
  // no prose line — rendered by pillRow() since Build 8, never
  // from here. The fallback lines below stay plain text: a
  // dropped or busy lender is a notice, not an attribution, and
  // local replies carry nothing at all — absence means local.
  if (!mesh) return "";
  if (mesh.servedBy === "mesh") {
    // v0.4.2e (#231): the automatic precedence preferred the
    // federation and a Mesh machine answered instead — the
    // fallback names itself under the reply, beside the pill.
    if (mesh.autoFellFrom === "federation") return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc((mesh.autoFedinet || "The federation") + " had no member answering — this reply came over the Mesh instead") + "</div>";
    // 0.2.0e Build 7 (#154): the pill names the machine that
    // ANSWERED; the road it took gets its own quiet line.
    // Build 2 (#203): a federation answer's pill carries the
    // carrier itself, so the separate line would say the same
    // thing twice — it stands down for fedinet routes only.
    if (mesh.viaRelay && !mesh.fedinet) return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>via relay " + esc(mesh.viaRelay) + "</div>";
    return "";
  }
  if (mesh.notice) return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(mesh.notice) + "</div>";
  if (mesh.fellBack && mesh.path === "relay") {
    // 0.2.0e Build 7 (#154): the relay road was tried and the
    // failure keeps its stage's name — never a bare code.
    var bn = mesh.node || "The paired machine";
    var rn = mesh.viaRelay || "the relay";
    var line = mesh.reason === "no-consent"
      ? (mesh.missing === "asker" ? "This machine's own relay consent for the road to " + bn + " is not ticked"
        : mesh.missing === "lender" ? bn + " has not ticked that it accepts relayed envelopes from " + rn
        : rn + " has not ticked that it will carry between this machine and " + bn)
      : mesh.reason === "relay-dark" ? rn + " could not be reached to carry the envelope"
      : mesh.reason === "dest-dark" ? bn + " could not be reached through " + rn
      : mesh.reason === "dest-refused" ? bn + " refused the relayed envelope — " + rn + "'s road to it did not check out"
      : mesh.reason === "dest-unknown" ? rn + " is not paired with " + bn + ", so it cannot carry there"
      : mesh.reason === "hop-limit" ? "The envelope could not be carried — a relayed envelope is never re-relayed"
      : "";
    if (line) return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(line + " — answered locally") + "</div>";
  }
  if (mesh.fellBack) {
    // v0.4.2e (#231): every road the automatic precedence
    // tried came up empty — the line names the road that was
    // preferred, so the local answer never poses as the
    // precedence's first choice. (The relay branch above
    // keeps its own stage line: it is the more actionable
    // fact when a carrier was involved.)
    if (mesh.autoFellFrom === "federation" && mesh.reason !== "lender-error") {
      return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc("The automatic route tried " + (mesh.autoFedinet || "the federation") + " first and fell back — answered locally") + "</div>";
    }
    // v0.4.1e (#228): the chat's ROUTE found no lender in its
    // field — the line blames the route, not a machine that
    // was never asked.
    if (mesh.reason === "route-unavailable") {
      var rl = mesh.routeMode === "federation"
        ? "No member of " + (mesh.routeName || "that federation") + " is in this chat's route right now"
        : "No paired machine is in this chat's Mesh route right now";
      return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc(rl + " — answered locally") + "</div>";
    }
    var why = mesh.reason === "busy" ? " was busy"
      : mesh.reason === "cap" ? " is at its lending cap for the hour"
      : mesh.reason === "idle" ? " is not idle yet"
      : mesh.reason === "lending-off" ? " is not lending right now"
      : mesh.reason === "fingerprint" ? " presented a certificate that does not match its card"
      : mesh.reason === "unauthorized" ? " did not accept this machine's signature"
      : mesh.reason === "tls" ? " answered, but the secure handshake failed"
      : mesh.reason === "timeout" ? " did not answer in time"
      : mesh.reason === "too-large" ? " could not fit this chat in its context"
      : mesh.reason === "unreachable" ? " could not be reached"
      : "";
    return "<div style='color:var(--dim);font-size:11px;margin-top:2px'>" + esc((mesh.node || "The paired machine") + why + " — answered locally") + "</div>";
  }
  return "";
}
// -- Federation (0.2.0e Build 8, #154) ------------------------------
// Its own page (Alex's ruling, 2026-11-13 — not a Mesh tab): the
// fedinets this node keeps, their signed rosters, the gateway
// cards, the trust toggles and ticks, founding, borrow caps, and
// the trust audit log. DHD serves this same page as the remote
// admin surface; the bridge tiers the acts (read + revoke open,
// manage behind the Badge step-up, crown acts under the full
// ceremony), and refusals come back as sentences the page shows
// ON THE CARD that was acted on.
//
// The switch gate, structurally: no control on this page ever
// flips locally. A press sends the act; the bridge answers with
// the state it actually stored; the page re-renders from that
// answer alone. Flip, stored state, and reload read one truth
// because there is only the one truth to read.
var federationState = null;
var fedMsg = { error: "", errorFor: "", sentence: "" };
var fedConfirm = null;
var federationError = "";
var federationFetching = false;

function federationAct(body, forId) {
  if (CONFIG.backend !== "live") { fedMsg.error = "Federation runs through the live bridge."; fedMsg.errorFor = forId || ""; render(); return; }
  fetch("/api/federation", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.fedinets) federationState = j;
    fedMsg.error = j && j.ok ? "" : ((j && j.error) || "The bridge did not answer that act.");
    fedMsg.errorFor = j && j.ok ? "" : (forId || "");
    fedMsg.sentence = (j && j.sentence) || "";
    if (view === "Federation") render();
    if (view === "FediChat") renderFedichat();
  }).catch(function () {
    fedMsg.error = "The bridge did not answer that act."; fedMsg.errorFor = forId || "";
    if (view === "Federation") render();
  });
}
function federationErrorHtml() {
  // 0.2.0e Build 10 (#195): the answer that did not set state,
  // painted ONCE — the bridge's own sentence, and a manual
  // retry. Nothing here fetches on its own; the strobe this
  // replaces repainted the page as fast as the door answered.
  return "<div class='card'><b>Federation state did not load</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>The bridge's answer did not carry this node's federation state: " + esc(federationError) + "</p>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>The page shows that answer once and waits — it does not retry on its own. If the bridge is an older build than this page, updating it is the cure; otherwise check that the bridge is running, then retry.</p>" +
    "<div class='row' style='margin-top:8px'><button class='act' id='fed-retry' style='min-height:44px'>Retry</button></div></div>";
}
function federationFetchState() {
  // 0.2.0e Build 10 (#195): the ONE fetch of the federation
  // state — fired by the view's enter hook in goView (and by
  // the error card's manual Retry), guarded in flight, and
  // never called from render or from the wiring pass. Any
  // answer that does not set state lands in federationError
  // and is painted exactly once by federationHtml.
  if (federationFetching) return;
  if (CONFIG.backend !== "live") return;
  federationFetching = true;
  federationError = "";
  fetch("/api/federation", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ act: "state" }) }).then(function (r) { return r.json(); }).then(function (j) {
    federationFetching = false;
    if (j && j.fedinets) { federationState = j; federationError = ""; }
    else federationError = (j && j.error) || "the answer carried no federation state.";
    if (view === "Federation") render();
  }).catch(function () {
    federationFetching = false;
    federationError = "the bridge did not answer — is it running?";
    if (view === "Federation") render();
  });
}
function fedBadge() { return CONFIG.operatorBadge || ""; }
function fedSwitch(on, attrs, label) {
  return "<button class='switch' role='switch' aria-checked='" + (on ? "true" : "false") + "' " + attrs + " aria-label='" + esc(label) + "'><span class='knob'></span></button>";
}
function fedConfirmHtml(fedId) {
  if (!fedConfirm || fedConfirm.fedinetId !== fedId) return "";
  return "<div class='card' style='margin-top:10px;border-color:#f5b544'><b>A crown act — speak its name.</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>Type the words <span class='mono' style='color:var(--ink)'>" + esc(fedConfirm.label) + "</span> to confirm. " +
    (federationState && federationState.viaDhd ? "Through the DHD door your Operator Badge rides with the act as the step-up; " : "") +
    "the act lands in the audit log either way.</p>" +
    "<div class='row' style='margin-top:6px'><input class='grow' id='fed-confirm-in' type='text' placeholder='" + esc(fedConfirm.label) + "' aria-label='Type the confirm words'><button class='act' data-fed-confirm-go='1' style='min-height:44px'>Confirm</button><button class='ghost' data-fed-confirm-cancel='1' style='min-height:44px'>Cancel</button></div></div>";
}
// -- The Dossier (v0.5.6e, #260): one dossier, many doors --
// Casework -> The Dossier, the people-finding app. Tabs
// across the top — Person / Username / Email / Phone /
// Photo — are keys into the SAME dossier object; the bridge
// runs the lookups (public, passive, keyless; every result
// cited; absence named), the page is the view, and Navi
// runs the same machinery from chat (her runs land in the
// history below like the operator's). Markdown is RENDERED
// here (mdToHtml, the #265 escape-first road), never raw.
var dzState = null, dzFetching = false, dzTab = "person";
var dzMsg = "", dzMsgOk = false;
// v0.5.9e (#280): the delete arm — the element id whose
// confirm strip stands open, and the optional reason typed
// into it. The confirm names the blast radius before it
// fires; the reason, if given, rides the custody lines.
var dzDelArm = "", dzDelReason = "";
var DZ_KIND_ORDER = ["person", "username", "email", "phone", "photo", "courts", "business", "license", "archive", "address", "record"];
function dzElementLabel(e) { return e.kind === "record" ? "the record" : e.kind + ": " + e.key; }
var DZ_TABS = [["person", "Person"], ["username", "Username"], ["email", "Email"], ["phone", "Phone"], ["photo", "Photo"], ["courts", "Courts"], ["business", "Business"], ["license", "License"], ["archive", "Archive"], ["address", "Address"]];
// v0.5.15e (#290): the query builder's composed plan —
// shown in the open, fired only on the confirm press.
var dzPlan = null;

// SIEM -> Seraph, the SIEM suite (#335; the floor, #153).
// Round one is standard Apex, manageable whole from DHD: the
// seat's role, the registry, per-sensor dashboards and the SSH
// tab all live on this one view. State rides GET
// /api/seraph/state; every act rides POST /api/seraph/act.
var srState = null, srFetching = false, srSensor = "", srTab = "dash";
var srMsg = "", srMsgOk = true, srSshTimer = null, srSshSince = 0, srSshChat = "";
function srGlyph(size) {
  return "<svg aria-hidden='true' width='" + size + "' height='" + size + "' viewBox='0 0 48 48'><path fill='#e9a13b' d='M11 35.5C6.2 30.6 4.4 23.8 6.1 16.4c1.3 2.8 3 4.7 5.3 5.9-1-3.1-1.1-6.4-.4-9.8 2.5 4.2 4.2 10.4 4.6 17z'/><path fill='#e9a13b' transform='translate(48 0) scale(-1 1)' d='M11 35.5C6.2 30.6 4.4 23.8 6.1 16.4c1.3 2.8 3 4.7 5.3 5.9-1-3.1-1.1-6.4-.4-9.8 2.5 4.2 4.2 10.4 4.6 17z'/><circle cx='24' cy='24' r='14.5' fill='none' stroke='#14b8a6' stroke-width='4'/><circle cx='24' cy='24' r='8.6' fill='none' stroke='#14b8a6' stroke-width='3.6'/><path fill='#e9a13b' d='M24 17.2c2.3 3.4 4.4 5.5 4.4 8.4a4.4 4.4 0 1 1-8.8 0c0-2.9 2.1-5 4.4-8.4z'/></svg>";
}
function srCeremonial(size) {
  // The winged eye — Seraph's ceremonial mark, for full-size
  // surfaces only (sensor page heroes, the sensor edition's
  // splash when it comes). Six wings, one eye, gold on teal.
  var w = "";
  [-1, 1].forEach(function (s) {
    [0, 1, 2].forEach(function (i) {
      var a = -38 + i * 30;
      w += "<path fill='#14b8a6' opacity='" + (0.95 - i * 0.22) + "' transform='translate(60 40) scale(" + s + " 1) rotate(" + a + ")' d='M6 -4C20 -14 34 -16 44 -12C36 -4 22 0 8 2Z'/>";
    });
  });
  return "<svg aria-hidden='true' width='" + size + "' height='" + Math.round(size * 0.66) + "' viewBox='0 0 120 80'>" + w +
    "<path fill='none' stroke='#e9a13b' stroke-width='3.4' d='M22 40C34 20 46 12 60 12s26 8 38 28c-12 20-24 28-38 28S34 60 22 40Z'/>" +
    "<circle cx='60' cy='40' r='12.5' fill='#e9a13b'/><circle cx='60' cy='40' r='5.6' fill='#0b0f14'/></svg>";
}
function srWhen(ts) { return ts ? new Date(ts * 1000).toLocaleString() : "never"; }
function refreshSeraph(rerender) {
  if (CONFIG.backend !== "live" || srFetching) return;
  srFetching = true;
  fetch("/api/seraph/state")
    .then(function (r) { return r.json(); })
    .then(function (j) { srFetching = false; if (j && j.ok) srState = j; if (rerender !== false && view === "Seraph") render(); })
    .catch(function () { srFetching = false; });
}
function seraphAct(action, body, done) {
  if (CONFIG.backend !== "live") { srMsg = "Seraph runs through the live bridge."; srMsgOk = false; render(); return; }
  var payload = { action: action };
  Object.keys(body || {}).forEach(function (k) { payload[k] = body[k]; });
  fetch("/api/seraph/act", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (j && j.ok === false) { srMsg = j.error || "The bridge did not answer that act."; srMsgOk = false; }
      else { srMsg = ""; if (done) done(j); }
      refreshSeraph(true);
    })
    .catch(function () { srMsg = "The bridge did not answer that act."; srMsgOk = false; if (view === "Seraph") render(); });
}
function seraphHtml() {
  var h = "<div id='seraph-root'>";
  h += "<div class='row' style='align-items:center;gap:10px'>" + srGlyph(34) + "<h3 style='margin:0'>Seraph · the watch floor</h3></div>";
  h += "<p style='color:var(--dim)'>The SIEM suite: sensors watching their segments, detections in Navi's voice, evidence that files itself into cases. A sensor sees the network it is plugged into — no more, and it says so.</p>";
  if (srMsg) h += "<p style='color:" + (srMsgOk ? "var(--cy)" : "#f57c7c") + "'>" + esc(srMsg) + "</p>";
  if (CONFIG.backend !== "live") { h += "<div class='card'>Seraph runs through the live bridge — switch the backend to Live in Settings. The shape below is the suite's.</div></div>"; return h; }
  if (!srState) { if (!srFetching) refreshSeraph(true); h += "<div class='card'>Calling the watch floor…</div></div>"; return h; }
  if (srSensor) { h += srSensorHtml(); h += "</div>"; return h; }
  var st = srState.settings || {};
  // This seat.
  h += "<div class='card'><b>This seat</b> <span style='color:var(--dim);font-size:12px'>— Seraph is a configuration of Apex: this seat's role is yours to set.</span>";
  h += "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'><label>Role <select id='sr-role' aria-label='This seat's Seraph role'><option value='console'" + (st.role === "console" ? " selected" : "") + ">Console — watch other sensors</option><option value='sensor'" + (st.role === "sensor" ? " selected" : "") + ">Sensor — watch this segment</option><option value='both'" + (st.role === "both" ? " selected" : "") + ">Both</option></select></label>";
  h += "<label>Retention (days) <input id='sr-ret' type='number' min='1' max='365' value='" + esc(String(st.retentionDays || 30)) + "' style='width:70px' aria-label='Retention in days'></label>";
  h += "<label>Sleep window <input id='sr-sleep1' type='text' value='" + esc(st.sleepStart || "01:00") + "' style='width:58px' aria-label='Sleep window start'> – <input id='sr-sleep2' type='text' value='" + esc(st.sleepEnd || "05:00") + "' style='width:58px' aria-label='Sleep window end'></label>";
  h += "<label>Console node id <input id='sr-console' type='text' value='" + esc(st.consoleNode || "") + "' placeholder='(reports stay home)' style='width:170px' aria-label='Console node id for reports'></label></div>";
  h += "<div class='row' style='margin-top:8px;gap:8px'><button class='act' id='sr-role-save'>Save seat</button>" + (st.role !== "console" ? "<button class='ghost' id='sr-collect'>Collect now</button>" : "") + "<button class='ghost' id='sr-refresh'>Refresh</button></div></div>";
  // Sensors.
  h += "<div class='card'><b>Sensors</b> <span style='color:var(--dim);font-size:12px'>— registered here, each watching its own segment. PIA registers as the first remote sensor when she ships.</span>";
  var sensors = srState.sensors || [];
  if (!sensors.length) h += "<p style='color:var(--dim)'>No sensors yet. Give this seat the sensor role above, or register a remote sensor below.</p>";
  sensors.forEach(function (s) {
    var snap = s.lastSnapshot;
    h += "<div class='card' style='margin-top:8px'><div class='row' style='gap:8px;align-items:center'><b class='grow'>" + esc(s.name) + "</b><span style='color:var(--dim);font-size:12px'>" + esc(s.kind) + (s.state === "revoked" ? " · REVOKED" : "") + "</span></div>";
    h += "<div style='color:var(--dim);font-size:12px'>Last seen " + srWhen(s.lastSeen) + (snap ? " · " + snap.assets + " hosts · " + snap.listeners + " listeners" : " · no picture yet") + (s.openDetections ? " · <b style='color:#f5b544'>" + s.openDetections + " open detection(s)</b>" : "") + (s.vantage ? "<br>Vantage: " + esc(s.vantage) : "") + "</div>";
    h += "<div class='row' style='margin-top:6px;gap:8px'><button class='act' data-sropen='" + esc(s.id) + "'>Open</button>" + (s.state !== "revoked" && s.kind !== "self" ? "<button class='ghost' data-srrevoke='" + esc(s.id) + "'>Revoke</button>" : "") + "</div></div>";
  });
  h += "<div class='row' style='margin-top:10px;flex-wrap:wrap;gap:8px'><input id='sr-reg-name' type='text' placeholder='Sensor name — e.g. PIA · Branch office' aria-label='Sensor name'><input id='sr-reg-addr' type='text' placeholder='Address (for SSH)' aria-label='Sensor address'><input id='sr-reg-user' type='text' placeholder='SSH user' aria-label='SSH user'><input id='sr-reg-node' type='text' placeholder='Mesh node id' aria-label='Mesh node id'><input id='sr-reg-vantage' type='text' placeholder='Vantage — the segment it watches' aria-label='Sensor vantage'></div><button class='ghost' id='sr-register' style='margin-top:8px'>Register sensor</button></div>";
  // Watchlist.
  var wl = srState.watchlist || [];
  h += "<div class='card'><b>Watchlist</b> <span style='color:var(--dim);font-size:12px'>— matched on the sensor itself; the list never leaves this machine (the Wire's declared list matches too). Only matches travel.</span><div style='margin-top:6px'>" +
    (wl.length ? wl.map(function (t) { return "<span class='mono' style='margin-right:10px'>" + esc(t) + " <button class='ghost' data-srwlremove='" + esc(t) + "' aria-label='Remove " + esc(t) + "'>×</button></span>"; }).join("") : "<span style='color:var(--dim)'>Nothing watched yet.</span>") +
    "</div><div class='row' style='margin-top:8px;gap:8px'><input id='sr-wl-add' type='text' placeholder='IP or name to watch' aria-label='Watchlist term'><button class='ghost' id='sr-wl-addbtn'>Watch it</button></div></div>";
  // Detections.
  h += srDetectionsHtml((srState.detections || []), "");
  h += "</div>";
  return h;
}
function srDetectionsHtml(dets, sensorId) {
  var names = {};
  (srState.sensors || []).forEach(function (s) { names[s.id] = s.name; });
  var list = dets.filter(function (d) { return !sensorId || d.sensorId === sensorId; });
  var h = "<div class='card'><b>Detections</b>";
  if (!list.length) { h += "<p style='color:var(--dim)'>Nothing yet — the floor is quiet.</p></div>"; return h; }
  list.slice(0, 50).forEach(function (d) {
    h += "<div class='card' style='margin-top:8px'><div class='row' style='gap:8px'><b class='grow'>" + esc(d.title) + "</b><span style='color:var(--dim);font-size:12px'>" + esc(d.state) + "</span></div>";
    h += "<div style='color:var(--dim);font-size:12px'>" + esc(names[d.sensorId] || d.sensorId) + " · " + srWhen(d.ts) + " · " + esc(d.kind) + "</div>";
    h += "<p style='margin:6px 0'>" + esc(d.detail) + "</p>";
    if (d.state === "open") h += "<div class='row' style='gap:8px'><button class='act' data-srfile='" + esc(d.id) + "'>File as case</button><button class='ghost' data-srdismiss='" + esc(d.id) + "'>Dismiss</button></div>";
    if (d.state === "filed" && d.caseSlug) h += "<div style='color:var(--dim);font-size:12px'>Filed as case <span class='mono'>" + esc(d.caseSlug) + "</span></div>";
    h += "</div>";
  });
  h += "</div>";
  return h;
}
function srSensorHtml() {
  var s = null;
  (srState.sensors || []).forEach(function (x) { if (x.id === srSensor) s = x; });
  if (!s) { srSensor = ""; return "<div class='card'>That sensor is gone. <button class='ghost' id='sr-back'>Back</button></div>"; }
  var h = "<div class='row' style='gap:10px;align-items:center'><button class='ghost' id='sr-back'>← Sensors</button>" + srCeremonial(64) + "<div><b>" + esc(s.name) + "</b><div style='color:var(--dim);font-size:12px'>" + esc(s.kind) + " · last seen " + srWhen(s.lastSeen) + (s.state === "revoked" ? " · REVOKED (tombstone)" : "") + "</div></div></div>";
  h += "<div class='tabbar' role='tablist' aria-label='Sensor sections' style='margin:10px 0'><button class='" + (srTab === "dash" ? "act" : "ghost") + "' data-srtab='dash'>Dashboard</button><button class='" + (srTab === "ssh" ? "act" : "ghost") + "' data-srtab='ssh'>SSH</button></div>";
  if (srTab === "ssh") { h += srSshHtml(s); return h; }
  var snap = s.lastSnapshot;
  if (!snap) { h += "<div class='card'>No picture yet — this sensor has not reported. When it collects, its segment appears here.</div>"; return h; }
  h += "<div class='card'><b>The picture</b> <span style='color:var(--dim);font-size:12px'>as of " + srWhen(snap.ts) + "</span><div class='row' style='gap:16px;margin-top:6px;flex-wrap:wrap'><span><b>" + snap.assets + "</b> hosts</span><span><b>" + snap.conns + "</b> connections</span><span><b>" + snap.listeners + "</b> listeners</span></div>";
  if (snap.gaps && snap.gaps.length) h += "<p style='color:#f5b544;font-size:12px'>Vantage gaps: " + snap.gaps.map(esc).join("; ") + "</p>";
  if (snap.remotes && snap.remotes.length) { h += "<p style='color:var(--dim);font-size:12px'>Top talkers: " + snap.remotes.slice(0, 8).map(function (r) { return esc(r.ip) + ":" + r.port + " (" + r.count + ")"; }).join(" · ") + "</p>"; }
  h += "</div>";
  var inv = snap.assetList || [];
  h += "<div class='card'><b>Inventory</b>" + (inv.length ? inv.map(function (a) { return "<div class='row' style='gap:8px;font-size:13px'><span class='grow'>" + esc(a.name || a.ip) + "</span><span class='mono' style='color:var(--dim)'>" + esc(a.ip) + "</span><span class='mono' style='color:var(--dim)'>" + esc(a.mac) + "</span></div>"; }).join("") : "<p style='color:var(--dim)'>No hosts in the last picture.</p>") + "</div>";
  var lis = snap.listenerList || [];
  h += "<div class='card'><b>Listening services</b>" + (lis.length ? lis.map(function (l) { return "<div class='mono' style='font-size:13px'>" + esc(l.ip) + ":" + l.port + "/" + esc(l.proto) + "</div>"; }).join("") : "<p style='color:var(--dim)'>None seen.</p>") + "</div>";
  h += srDetectionsHtml((srState.detections || []), s.id);
  return h;
}
function srSshHtml(s) {
  var h = "<div class='card'><b>SSH — " + esc(s.name) + "</b><p style='color:var(--dim);font-size:13px'>Your own key, your own session, opened by hand. Seraph keeps no standing session and stores no credential; the session runs in this seat's shell machinery, and closing the tab ends it.</p>";
  if (s.kind === "self") { h += "<p>This sensor is this seat itself — its shell is the one in Chat's CLI mode.</p></div>"; return h; }
  if (!s.address) { h += "<p>No address on this sensor's record — register one to open a session.</p></div>"; return h; }
  h += "<div class='row' style='gap:8px'><button class='act' id='sr-ssh-open'>Open SSH session</button><button class='ghost' id='sr-ssh-close'>Close</button></div>";
  h += "<pre id='sr-term' class='mono' style='margin-top:10px;background:#0b0f14;color:#c9e8e2;border-radius:8px;padding:10px;min-height:220px;max-height:420px;overflow:auto;font-size:12px;white-space:pre-wrap'></pre>";
  h += "<div class='row' style='gap:8px;margin-top:8px'><input class='grow mono' id='sr-ssh-line' type='text' placeholder='Type a line, Enter sends it' aria-label='SSH input line'><button class='ghost' id='sr-ssh-send'>Send</button></div></div>";
  return h;
}
function srPollSsh() {
  if (view !== "Seraph" || srTab !== "ssh" || !srSshChat) return;
  fetch("/api/shell/session/state?chatId=" + encodeURIComponent(srSshChat) + "&since=" + srSshSince)
    .then(function (r) { return r.json(); })
    .then(function (j) {
      var term = document.getElementById("sr-term");
      if (term && j) {
        if (typeof j.seq === "number") srSshSince = j.seq;
        (j.events || []).forEach(function (e) {
          var t = (e && (e.data || e.text)) || "";
          if (t) term.textContent += t;
        });
        term.scrollTop = term.scrollHeight;
        if (j.ended) term.textContent += "\n[session ended" + (j.closeReason ? ": " + j.closeReason : "") + "]\n";
      }
      if (view === "Seraph" && srTab === "ssh" && srSshChat && !(j && j.ended)) srSshTimer = setTimeout(srPollSsh, 1200);
    })
    .catch(function () { if (view === "Seraph" && srTab === "ssh") srSshTimer = setTimeout(srPollSsh, 2500); });
}
function bindSeraph() {
  function on(id, ev, fn) { var el = document.getElementById(id); if (el) el.addEventListener(ev, fn); }
  on("sr-refresh", "click", function () { refreshSeraph(true); });
  on("sr-role-save", "click", function () {
    seraphAct("role", { role: document.getElementById("sr-role").value, retentionDays: document.getElementById("sr-ret").value, sleepStart: document.getElementById("sr-sleep1").value, sleepEnd: document.getElementById("sr-sleep2").value, consoleNode: document.getElementById("sr-console").value });
  });
  on("sr-collect", "click", function () { seraphAct("collect", {}); });
  on("sr-register", "click", function () {
    seraphAct("register", { name: document.getElementById("sr-reg-name").value, address: document.getElementById("sr-reg-addr").value, sshUser: document.getElementById("sr-reg-user").value, nodeId: document.getElementById("sr-reg-node").value, vantage: document.getElementById("sr-reg-vantage").value });
  });
  on("sr-wl-addbtn", "click", function () { seraphAct("watchlist", { term: document.getElementById("sr-wl-add").value }); });
  on("sr-back", "click", function () { srSensor = ""; srTab = "dash"; render(); });
  document.querySelectorAll("[data-sropen]").forEach(function (b) { b.addEventListener("click", function () { srSensor = b.getAttribute("data-sropen"); srTab = "dash"; render(); }); });
  document.querySelectorAll("[data-srrevoke]").forEach(function (b) { b.addEventListener("click", function () { seraphAct("revoke", { id: b.getAttribute("data-srrevoke") }); }); });
  document.querySelectorAll("[data-srfile]").forEach(function (b) { b.addEventListener("click", function () { seraphAct("detection", { id: b.getAttribute("data-srfile"), act: "file" }); }); });
  document.querySelectorAll("[data-srdismiss]").forEach(function (b) { b.addEventListener("click", function () { seraphAct("detection", { id: b.getAttribute("data-srdismiss"), act: "dismiss" }); }); });
  document.querySelectorAll("[data-srwlremove]").forEach(function (b) { b.addEventListener("click", function () { seraphAct("watchlist", { term: b.getAttribute("data-srwlremove"), remove: true }); }); });
  document.querySelectorAll("[data-srtab]").forEach(function (b) { b.addEventListener("click", function () { srTab = b.getAttribute("data-srtab"); if (srTab !== "ssh" && srSshTimer) { clearTimeout(srSshTimer); srSshTimer = null; } render(); }); });
  on("sr-ssh-open", "click", function () {
    var s = null; (srState.sensors || []).forEach(function (x) { if (x.id === srSensor) s = x; });
    if (!s) return;
    srSshChat = "seraph-ssh-" + s.id; srSshSince = 0;
    fetch("/api/shell/session/open", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId: srSshChat, cols: 100, rows: 30 }) })
      .then(function (r) { return r.json(); })
      .then(function () {
        var cmd = "ssh -o StrictHostKeyChecking=accept-new " + (s.sshUser ? s.sshUser + "@" : "") + s.address + "\n";
        return fetch("/api/shell/session/input", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId: srSshChat, text: cmd }) });
      })
      .then(function () { var term = document.getElementById("sr-term"); if (term) term.textContent = ""; srPollSsh(); })
      .catch(function () { srMsg = "The SSH session would not open."; srMsgOk = false; render(); });
  });
  on("sr-ssh-close", "click", function () {
    if (srSshTimer) { clearTimeout(srSshTimer); srSshTimer = null; }
    if (srSshChat) fetch("/api/shell/session/close", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId: srSshChat }) }).catch(function () {});
    srSshChat = "";
  });
  function srSendLine() {
    var inp = document.getElementById("sr-ssh-line");
    if (!inp || !srSshChat) return;
    var text = inp.value + "\n"; inp.value = "";
    fetch("/api/shell/session/input", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId: srSshChat, text: text }) }).catch(function () {});
  }
  on("sr-ssh-send", "click", srSendLine);
  var line = document.getElementById("sr-ssh-line");
  if (line) line.addEventListener("keydown", function (e) { if (e.key === "Enter") { e.preventDefault(); srSendLine(); } });
}
function refreshDossier(rerender) {
  if (CONFIG.backend !== "live" || dzFetching) return;
  dzFetching = true;
  fetch("/api/dossier", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ action: "state" }) })
    .then(function (r) { return r.json(); })
    .then(function (j) { dzFetching = false; if (j && j.dossiers) dzState = j; if (rerender !== false && view === "Dossier") render(); })
    .catch(function () { dzFetching = false; });
}
function dossierAct(action, body) {
  if (CONFIG.backend !== "live") { dzMsg = "The Dossier runs through the live bridge."; dzMsgOk = false; render(); return; }
  var payload = { action: action };
  Object.keys(body || {}).forEach(function (k) { payload[k] = body[k]; });
  fetch("/api/dossier", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (j && j.dossiers) dzState = j;
      if (j && j.ok === false) { dzMsg = j.error || "The bridge did not answer that act."; dzMsgOk = false; }
      else if (action === "open-case" && j.caseSlug) { dzMsg = "OSINT case opened: " + j.caseSlug + " — the dossier stands as its subject."; dzMsgOk = true; }
      else if (action === "file" && j.filed) { dzMsg = "Filed into the case at " + j.filed.path + "."; dzMsgOk = true; }
      else if (action === "delete-element" && j.deleted) { dzMsg = "Deleted " + j.deleted.label + " — from this dossier" + (j.deleted.casesCleaned && j.deleted.casesCleaned.length ? ", from " + j.deleted.casesCleaned.length + " case(s), and from the Vault formations" : "") + ". The cases' custody records carry the line."; dzMsgOk = true; dzDelArm = ""; dzDelReason = ""; }
      else if (action === "merge") { dzMsg = "Merged — one dossier, every key."; dzMsgOk = true; }
      else if (action === "plan" && j.plan) { dzPlan = j.plan; dzMsg = "The plan is composed — nothing has run. The confirm below is the fire."; dzMsgOk = true; }
      else if (action === "run-plan" && j.planRan) { dzMsg = "The plan ran: " + j.planRan.join(" · "); dzMsgOk = true; dzPlan = null; }
      else if (action === "pivot" && j.pivoted) { dzMsg = "The proposed door ran — its section stands on the dossier."; dzMsgOk = true; }
      else { dzMsg = ""; }
      if (view === "Dossier") render();
    })
    .catch(function () { dzMsg = "The bridge did not answer that act."; dzMsgOk = false; if (view === "Dossier") render(); });
}
function dossierHtml() {
  var h = "<h3>The Dossier</h3>";
  h += "<p style='color:var(--dim)'>Apex's people-finding app — the full kit (v0.5.15e). Every tab is a key into the <b>same dossier</b> — runs attach to the open dossier, merges are yours to make, and every result is cited to its source. Public sources, passive and keyless; absence is named — <i>no public record found</i> — never dressed up; a hit is a lead, never a conclusion; credential material is never sought (breach <i>presence</i> only); no face recognition, ever. The shelves: the original five keys, Courts (federal dockets + opinions read keyless; PACER and the state portals stand as named pointers), Business (EDGAR, OpenCorporates, 990s, federal awards), License (callsigns and N-numbers read directly; name-keyed boards stand as pointers), Archive (the Wayback index; scholarship), and Address — the civil spine. Navi runs the same machinery from chat — say <span class='mono'>dossier: courts …</span>, <span class='mono'>court records for …</span>, or <span class='mono'>dossier plan for …</span> — and her runs land in the history like yours.</p>";
  if (CONFIG.backend !== "live") return h + "<div class='card'>The Dossier runs through the live bridge — switch the backend to Live in Settings.</div>";
  if (!dzState && !dzFetching) refreshDossier(false);
  if (!casesState && !casesFetching) refreshCases();
  if (!dzState) return h + "<div class='card'>Opening the dossier…</div>";
  if (dzMsg) h += "<div class='card' style='border-color:" + (dzMsgOk ? "var(--green)" : "#f87171") + "'>" + esc(dzMsg) + "</div>";
  // Tabs across the top — the strip scrolls on a phone.
  h += "<div class='dz-tabs' role='tablist' aria-label='Dossier keys'>"
    + DZ_TABS.map(function (t) { return "<button role='tab' class='" + (dzTab === t[0] ? "act" : "ghost") + "' data-dztab='" + t[0] + "' aria-label='" + t[1] + " tab'>" + t[1] + "</button>"; }).join("")
    + "</div>";
  var open = dzState.open;
  // The run form for the active tab.
  h += "<div class='card dz-run'>";
  if (dzTab === "person") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Full name — e.g. Jane Q Public' aria-label='Person name'><input id='dz-location' type='text' placeholder='Location (optional) — City, ST' aria-label='Location (optional)'></div>"
      + "<div class='row' style='margin-top:8px'><input class='grow' id='dz-aliases' type='text' placeholder='Registry anchors (optional) — aliases, comma-separated' aria-label='Aliases (optional)'><input id='dz-domain' type='text' placeholder='Held domain (optional)' aria-label='Held domain (optional)'></div>"
      + "<p style='color:var(--dim);font-size:12px'>The registry check is the anchor: it stands at the registries' own doors (NSOPW + the state registry), cited — and beside it, the sanctions check: the OFAC lists (SDN + consolidated) are read from the Treasury's own publication, a match named as a lead, a miss stated as a miss. Anchors you bring back from a record — its listed aliases, a held domain — are what enrichment fans out from; a same-name guess never inherits a stranger's record.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the person lookup'>Look up the person</button>";
  } else if (dzTab === "username") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Handle — e.g. shadowfox' aria-label='Username handle'></div>"
      + "<p style='color:var(--dim);font-size:12px'>The handle is swept across the platform corpus (44 platforms) by polite profile checks: hits cited per platform, misses listed as checked-clean, doors that do not answer named as such.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the username sweep'>Sweep the handle</button>";
  } else if (dzTab === "email") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Email address — e.g. jane@example.com' aria-label='Email address'></div>"
      + "<p style='color:var(--dim);font-size:12px'>The address's public posture — MX, SPF, DMARC read from DNS — plus breach <i>presence</i> at the source. Credential content never enters the kit.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the email lookup'>Look up the address</button>";
  } else if (dzTab === "phone") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Phone number — e.g. +1 215 555 0134' aria-label='Phone number'></div>"
      + "<p style='color:var(--dim);font-size:12px'>Facts from the number's own structure and public sources, plus cited search pointers in its common formats. The kit cites sources; it does not launder aggregator claims.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the phone lookup'>Look up the number</button>";
  } else if (dzTab === "photo") {
    h += "<p style='color:var(--dim);font-size:12px'>Hand the bridge an image and it reads the embedded metadata (EXIF) — <b>fully local</b>: nothing is uploaded anywhere, and no face recognition is performed, ever.</p>"
      + "<input id='dz-photo-file' type='file' accept='image/jpeg,image/png' aria-label='Choose an image to read'>"
      + "<div id='dz-photo-name' style='color:var(--dim);font-size:12px;margin-top:6px'>No file chosen.</div>"
      + "<button class='act' id='dz-run' style='margin-top:8px' aria-label='Read the photo metadata'>Read the metadata</button>";
  } else if (dzTab === "courts") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Party name — e.g. Jane Q Public' aria-label='Court party name'><input id='dz-location' type='text' placeholder='State (optional) — e.g. PA' aria-label='State (optional)'></div>"
      + "<p style='color:var(--dim);font-size:12px'>Federal dockets and opinions are read keyless through CourtListener (the RECAP archive). PACER is credentialed and per-page — it stands as a named pointer, as do the state portals (session-driven forms). A docket names a party; it does not characterize them.</p>"
      + "<button class='act' id='dz-run' aria-label='Search the court records'>Search the dockets</button>";
  } else if (dzTab === "business") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Entity name — e.g. Acme Corp' aria-label='Business name'></div>"
      + "<p style='color:var(--dim);font-size:12px'>SEC EDGAR (filer match, company facts, recent filings, full-text), OpenCorporates registrations and their officers of record, ProPublica's nonprofit (990) filings, and USAspending federal awards. The FEC and USPTO's TESS stand as named pointers; Pennsylvania's entity search is form-driven and stands at the Commonwealth's door.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the business lookup'>Look up the business</button>";
  } else if (dzTab === "license") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Callsign, N-number, or a name — e.g. W1AW, N12345, Jane Public' aria-label='License key'></div>"
      + "<p style='color:var(--dim);font-size:12px'>What has a keyless machine shape, this kit reads: amateur callsigns (the FCC ULS mirror) and aircraft N-numbers (the FAA registry). Professional licenses are kept by name behind the boards' own session forms — PA PALS, ULS name search and the FAA airmen inquiry stand as named pointers, and the run says so.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the license check'>Check the license</button>";
  } else if (dzTab === "archive") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='A URL or domain — or a name, for scholarship' aria-label='Archive key'></div>"
      + "<p style='color:var(--dim);font-size:12px'>A URL reads the Wayback Machine's snapshot index (count, first, latest — each cited). A name reads the scholarship record: OpenAlex author records and Crossref works. A snapshot is the page as the archive saw it — a record of the page, not of the truth of its claims.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the archive lookup'>Check the archive</button>";
  } else if (dzTab === "address") {
    h += "<div class='row'><input class='grow' id='dz-value' type='text' placeholder='Street address — e.g. 123 Main St, Pennsburg, PA' aria-label='Address'></div>"
      + "<p style='color:var(--dim);font-size:12px'>The civil spine: the address as the gazetteer states it (county, state), the county's own property door named, and the voter note for the state — a pointer where the state publishes a status check, an honest absence where the file is restricted by law. An address anchors records; it never convicts a place of its occupants.</p>"
      + "<button class='act' id='dz-run' aria-label='Run the address lookup'>Look up the address</button>";
  }
  h += "</div>";
  // The query builder (v0.5.15e, #290): compose first.
  h += "<div class='card'><b>Compose a run</b><p style='color:var(--dim);font-size:12px'>The shelves a starting key fans out to — composed in the open, with each shelf's doors and cost, BEFORE anything fires. The confirm is the fire.</p>"
    + "<div class='row'><select id='dz-plan-key' aria-label='Starting key type'><option value='person'>Person</option><option value='business'>Business</option><option value='address'>Address</option><option value='username'>Username</option></select><input class='grow' id='dz-plan-value' type='text' placeholder='The starting key — a name, an entity, an address, a handle' aria-label='Starting key value'><button class='ghost' id='dz-plan-go' aria-label='Compose the run'>Compose the run</button></div>";
  if (dzPlan) {
    h += "<div style='margin-top:8px'><b>" + esc(dzPlan.keyType) + ": " + esc(dzPlan.value) + "</b> <span style='color:var(--dim);font-size:12px'>— " + dzPlan.totalRequests + " request(s) across " + dzPlan.steps.length + " step(s)</span>"
      + dzPlan.steps.map(function (st, i) { return "<div style='font-size:13px;margin-top:4px'>" + (i + 1) + ". <b>" + esc(st.shelf) + "</b> <span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(st.run) + " · ~" + st.requests + " req</span> — " + esc(st.note) + "</div>"; }).join("")
      + "<p style='color:var(--dim);font-size:12px;margin-top:4px'>After: " + esc(dzPlan.after) + ".</p>"
      + "<button class='act' id='dz-plan-run' aria-label='Run the composed plan'>Run the plan</button></div>";
  }
  h += "</div>";
  // The open dossier.
  if (open) {
    h += "<div class='card'><div class='row'><b class='grow' style='font-size:15px'>" + esc(open.title) + "</b><span style='color:var(--dim);font-size:12px'>opened " + esc(open.created) + " · updated " + esc(open.updated) + "</span></div>";
    if (open.keys && open.keys.length) h += "<p style='margin-top:6px'>" + open.keys.map(function (k) { return "<span class='ic mono' style='margin-right:6px'>" + esc(k.type) + ": " + esc(k.value) + "</span>"; }).join("") + "</p>";
    // The built dossier (v0.5.9e, #280): ONE rendered
    // document made of its elements — grouped by kind, each
    // element carrying its provenance (key, door, sources,
    // when) and its typed case links by name, each rendered
    // through the same escape-first renderer as Case Files
    // and the Vault: the document reads rendered, and HTML
    // a source filed stays inert text. A dossier whose store
    // predates elements reads as the blob it always was.
    if (open.elements && open.elements.length) {
      h += "<div class='dz-elements' style='margin-top:8px'>";
      DZ_KIND_ORDER.forEach(function (kind) {
        var group = open.elements.filter(function (e) { return e.kind === kind; });
        if (!group.length) return;
        h += "<h4 style='margin:12px 0 4px'>" + esc(kind === "record" ? "Record" : kind.charAt(0).toUpperCase() + kind.slice(1)) + "</h4>";
        group.forEach(function (e) {
          h += "<div class='dz-element' style='border-top:1px solid var(--line);padding:8px 0'>";
          h += "<div class='row'><b class='grow' style='font-size:13px'>" + esc(dzElementLabel(e)) + "</b><span style='color:var(--dim);font-size:12px'>" + esc(e.at || "") + (e.door ? " · " + esc(e.door) + " door" : "") + "</span>"
            + (dzDelArm === e.id ? "" : "<button class='ghost' data-dzdel='" + esc(e.id) + "' aria-label='Delete element " + esc(dzElementLabel(e)) + " everywhere'>Delete</button>")
            + "</div>";
          if (e.sources && e.sources.length) h += "<div style='color:var(--dim);font-size:12px'>" + e.sources.map(function (s) { return esc(s.name) + " (" + esc(s.verdict) + ")"; }).join(" · ") + "</div>";
          if (e.cases && e.cases.length) h += "<div style='font-size:12px'>Filed into: " + e.cases.map(function (c) { return "<b>" + esc(c.name) + "</b>"; }).join(", ") + "</div>";
          h += "<div class='md'>" + mdToHtml(e.markdown || "") + "</div>";
          if (dzDelArm === e.id) {
            h += "<div class='card' style='margin-top:6px;border-color:#f87171'><p style='font-size:13px'>Delete <b>" + esc(dzElementLabel(e)) + "</b> everywhere? It leaves this dossier"
              + (e.cases && e.cases.length ? ", the case(s) it stands in — " + e.cases.map(function (c) { return esc(c.name); }).join(", ") + " —" : "")
              + " and the Vault formations. The cases keep a custody line naming where it was deleted from; nothing else remains.</p>"
              + "<div class='row' style='margin-top:6px'><input class='grow' id='dz-del-reason' type='text' placeholder='Reason (optional) — irrelevant / repeat / incorrect…' aria-label='Reason for deleting (optional)' value='" + esc(dzDelReason) + "'><button class='ghost' data-dzdelok='" + esc(e.id) + "' style='color:#f87171'>Delete everywhere</button><button class='ghost' data-dzdelno='1'>Keep it</button></div></div>";
          }
          h += "</div>";
        });
      });
      h += "</div>";
    } else if (open.markdown) h += "<div class='md' style='margin-top:8px'>" + mdToHtml(open.markdown) + "</div>";
    else h += "<p style='color:var(--dim)'>No runs yet — every key above opens this same dossier.</p>";
    if (open.mergedFrom && open.mergedFrom.length) h += "<p style='color:var(--dim);font-size:12px'>Merged in: " + open.mergedFrom.map(function (m) { return esc(m.title); }).join(", ") + ".</p>";
    h += "<p style='color:var(--dim);font-size:12px'>" + (open.links ? open.links.length : 0) + " typed link(s) on this dossier — keys, sources, merges, case inclusions, stored as data.</p>";
    // Proposed next doors (v0.5.15e, #290): one hop,
    // proposed — the press is the go; nothing auto-runs.
    if (dzState.pivots && dzState.pivots.length) {
      h += "<div style='margin-top:6px'><span style='color:var(--dim);font-size:12px'>Proposed next doors — one hop, your call:</span><div class='row' style='margin-top:4px'>"
        + dzState.pivots.map(function (pv) { return "<button class='ghost' data-dzpivot='" + esc(pv.keyType) + "|" + esc(pv.value) + "' aria-label='Run the proposed " + esc(pv.keyType) + " door for " + esc(pv.value) + "'>" + esc(pv.keyType) + ": " + esc(pv.value) + "</button>"; }).join("")
        + "</div><p style='color:var(--dim);font-size:12px'>" + dzState.pivots.map(function (pv) { return esc(pv.keyType) + " " + esc(pv.value) + " — " + esc(pv.via); }).join(" · ") + "</p></div>";
    }
    // Case citizenship, both ways.
    h += "<div class='row' style='margin-top:8px'>";
    if (open.caseSlug) h += "<span style='color:var(--dim)'>Stands as the OSINT case <b>" + esc(open.caseSlug) + "</b>.</span>";
    else h += "<button class='ghost' id='dz-open-case' aria-label='Open an OSINT case from this dossier'>Open OSINT case</button>";
    h += "<select id='dz-case-sel' aria-label='Case to file into' style='margin-left:8px'>" + (casesState || []).map(function (c) { return "<option value='" + esc(c.slug) + "'>" + esc(c.name || c.slug) + "</option>"; }).join("") + "</select>"
      + "<button class='ghost' id='dz-file-dossier' aria-label='File this dossier into the chosen case'>File dossier into case</button></div>";
    h += "</div>";
  } else {
    h += "<div class='card'>No dossier open — run any key and one opens, or start one below.</div>";
  }
  // Dossiers + merge.
  h += "<div class='card'><div class='row'><b class='grow'>Dossiers</b><input id='dz-new-title' type='text' placeholder='New dossier title' aria-label='New dossier title'><button class='ghost' id='dz-new' aria-label='Start a new dossier'>New dossier</button></div>";
  (dzState.dossiers || []).forEach(function (d) {
    h += "<div class='row' style='margin-top:6px'><span class='grow'>" + esc(d.title) + " <span style='color:var(--dim);font-size:12px'>· " + d.runs + " run(s)" + (d.caseSlug ? " · case " + esc(d.caseSlug) : "") + "</span></span>"
      + (open && d.id === open.id ? "<span style='color:var(--dim)'>open</span>" : "<button class='ghost' data-dzopen='" + esc(d.id) + "' aria-label='Open dossier " + esc(d.title) + "'>Open</button>")
      + (open && d.id !== open.id ? "<button class='ghost' data-dzmerge='" + esc(d.id) + "' aria-label='Merge " + esc(d.title) + " into the open dossier'>Merge into open</button>" : "")
      + "</div>";
  });
  h += "</div>";
  // Run history — both doors, the audit trail.
  h += "<div class='card'><b>Run history</b><p style='color:var(--dim);font-size:12px'>Every run, either door — its key, its sources, its verdicts. The audit trail a practitioner would sign.</p>";
  (dzState.runs || []).forEach(function (r) {
    h += "<div style='border-top:1px solid var(--line);padding:6px 0'><div class='row'><span class='grow' style='font-size:13px'><b>" + esc(r.keyType) + "</b> · " + esc(r.key) + " <span style='color:var(--dim)'>— " + esc(r.summary) + "</span></span><span style='color:var(--dim);font-size:12px'>" + esc(r.at) + " · " + esc(r.door) + " door</span></div>"
      + "<div style='color:var(--dim);font-size:12px'>" + (r.sources || []).map(function (s) { return esc(s.name) + " (" + esc(s.verdict) + ")"; }).join(" · ") + "</div>"
      + (open && r.dossierId === open.id ? "<button class='ghost' data-dzfile-run='" + esc(r.id) + "' aria-label='File this run into the chosen case'>File element into case</button>" : "")
      + "</div>";
  });
  if (!(dzState.runs || []).length) h += "<p style='color:var(--dim)'>No runs yet.</p>";
  h += "</div>";
  return h;
}
function wireDossier() {
  document.querySelectorAll("[data-dztab]").forEach(function (b) { b.onclick = function () { dzTab = b.getAttribute("data-dztab"); render(); }; });
  document.querySelectorAll("[data-dzopen]").forEach(function (b) { b.onclick = function () { dossierAct("open", { dossierId: b.getAttribute("data-dzopen") }); }; });
  document.querySelectorAll("[data-dzmerge]").forEach(function (b) { b.onclick = function () { if (dzState && dzState.open) dossierAct("merge", { keepId: dzState.open.id, dropId: b.getAttribute("data-dzmerge") }); }; });
  document.querySelectorAll("[data-dzfile-run]").forEach(function (b) { b.onclick = function () { var sel = document.getElementById("dz-case-sel"); if (sel && sel.value && dzState && dzState.open) dossierAct("file", { dossierId: dzState.open.id, runId: b.getAttribute("data-dzfile-run"), "case": sel.value }); }; });
  // The element delete (#280): arm, keep, fire. The reason
  // is read at fire time; it is optional and never required.
  document.querySelectorAll("[data-dzdel]").forEach(function (b) { b.onclick = function () { dzDelArm = b.getAttribute("data-dzdel"); dzDelReason = ""; render(); }; });
  document.querySelectorAll("[data-dzdelno]").forEach(function (b) { b.onclick = function () { dzDelArm = ""; dzDelReason = ""; render(); }; });
  document.querySelectorAll("[data-dzdelok]").forEach(function (b) { b.onclick = function () { var r = document.getElementById("dz-del-reason"); dossierAct("delete-element", { elementId: b.getAttribute("data-dzdelok"), from: "dossier", reason: r ? r.value.trim() : "" }); }; });
  document.querySelectorAll("[data-dzpivot]").forEach(function (b) { b.onclick = function () { var parts = (b.getAttribute("data-dzpivot") || "").split("|"); if (parts.length === 2) dossierAct("pivot", { keyType: parts[0], value: parts[1] }); }; });
  var pg = document.getElementById("dz-plan-go");
  if (pg) pg.onclick = function () { var k = document.getElementById("dz-plan-key"); var v = document.getElementById("dz-plan-value"); dossierAct("plan", { keyType: k ? k.value : "person", value: v ? v.value : "" }); };
  var pr = document.getElementById("dz-plan-run");
  if (pr) pr.onclick = function () { if (dzPlan) dossierAct("run-plan", { keyType: dzPlan.keyType, value: dzPlan.value, confirm: true }); };
  var runBtn = document.getElementById("dz-run");
  if (runBtn) runBtn.onclick = function () {
    if (dzTab === "photo") {
      var f = document.getElementById("dz-photo-file");
      var file = f && f.files && f.files[0];
      if (!file) { dzMsg = "Choose an image first — the bridge reads the file you hand it, locally."; dzMsgOk = false; render(); return; }
      var reader = new FileReader();
      reader.onload = function () {
        var b64 = String(reader.result || "").split(",")[1] || "";
        dossierAct("photo", { value: file.name, dataBase64: b64 });
      };
      reader.readAsDataURL(file);
      return;
    }
    var v = document.getElementById("dz-value");
    var body = { keyType: dzTab, value: v ? v.value : "" };
    if (dzTab === "courts") {
      var cloc = document.getElementById("dz-location");
      body.location = cloc ? cloc.value : "";
    }
    if (dzTab === "person") {
      var loc = document.getElementById("dz-location");
      var al = document.getElementById("dz-aliases");
      var dm = document.getElementById("dz-domain");
      body.location = loc ? loc.value : "";
      body.anchors = { aliases: (al && al.value ? al.value.split(",").map(function (s) { return s.trim(); }).filter(Boolean) : []), domain: dm ? dm.value.trim() : "", locality: body.location };
    }
    dossierAct("run", body);
  };
  var pf = document.getElementById("dz-photo-file");
  if (pf) pf.onchange = function () { var n = document.getElementById("dz-photo-name"); if (n) n.textContent = pf.files && pf.files[0] ? pf.files[0].name + " — read locally when you run it; nothing is uploaded." : "No file chosen."; };
  var oc = document.getElementById("dz-open-case");
  if (oc) oc.onclick = function () { if (dzState && dzState.open) dossierAct("open-case", { dossierId: dzState.open.id }); };
  var fd = document.getElementById("dz-file-dossier");
  if (fd) fd.onclick = function () { var sel = document.getElementById("dz-case-sel"); if (sel && sel.value && dzState && dzState.open) dossierAct("file", { dossierId: dzState.open.id, "case": sel.value }); };
  var nw = document.getElementById("dz-new");
  if (nw) nw.onclick = function () { var t = document.getElementById("dz-new-title"); dossierAct("new", { title: t ? t.value : "" }); };
}

// -- FediChat (v0.5.3e, #267): one signed room per fedinet --
// The operators talk; their nodes carry the messages; their
// Navis are in the room. State comes from the bridge
// (POST /api/fedichat) — the log, the roster, the chain's
// verdict and the room's settings all live bridge-side; this
// page is a view, like every other surface.
var fcState = null, fcFetching = false, fcFed = "", fcMsg = "", fcCompleteIdx = 0;
var fcTrayOpen = false, fcWinOpen = false;   // the room's overlays (#270): the members tray and the About & settings window; markup exists only while open, portaled to <body> like the wire reader
var fcRoomId = "", fcCreateOpen = false, fcQuote = null, fcJump = "", fcSearchResults = null, fcPresenceAt = 0;   // Build 2 (v0.5.7e): the selected room, the create overlay, a pending quote, a jump target, search results, the presence throttle
document.addEventListener("keydown", function (e) {
  if (e.key === "Escape" && (fcTrayOpen || fcWinOpen || fcCreateOpen)) { fcTrayOpen = false; fcWinOpen = false; fcCreateOpen = false; if (view === "FediChat") render(); }
});
function refreshFedichat(rerender) {
  if (CONFIG.backend !== "live" || fcFetching) return;
  fcFetching = true;
  fetch("/api/fedichat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ action: "state" }) })
    .then(function (r) { return r.json(); })
    .then(function (j) { fcFetching = false; fcState = j; if (rerender !== false && view === "FediChat") render(); })
    .catch(function () { fcFetching = false; });
}
function renderFedichat() { if (view === "FediChat") render(); }
function fedichatAct(action, extra, cb) {
  var body = { action: action };
  if (extra) for (var k in extra) body[k] = extra[k];
  fetch("/api/fedichat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      fcState = j;
      fcMsg = j && j.ok === false ? (j.error || "") : "";
      renderFedichat();
      if (cb) cb(j);
    })
    .catch(function () { fcMsg = "The bridge did not answer that."; renderFedichat(); });
}
function fedichatSearch(q, room) {
  // The citable log's search door: its answer is results,
  // not room state, so it never rides fedichatAct (which
  // would mistake the answer for the state).
  fetch("/api/fedichat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ action: "search", fed: room.fedId, room: room.roomId || "", q: q }) })
    .then(function (r) { return r.json(); })
    .then(function (j) { fcSearchResults = (j && j.results) || []; renderFedichat(); })
    .catch(function () { fcSearchResults = []; renderFedichat(); });
}
function fedichatPresence(state, room) {
  // Ephemeral gossip: fire-and-forget, throttled by the
  // caller. The answer is a bare ok — never room state.
  fetch("/api/fedichat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ action: "presence", fed: room.fedId, room: room.roomId || "", state: state }) }).catch(function () {});
}
function fedichatRoom() {
  if (!fcState || !fcState.rooms || !fcState.rooms.length) return null;
  for (var i = 0; i < fcState.rooms.length; i++) if (fcState.rooms[i].fedId === fcFed && (fcState.rooms[i].roomId || "") === fcRoomId) return fcState.rooms[i];
  for (var i = 0; i < fcState.rooms.length; i++) if (fcState.rooms[i].fedId === fcFed) return fcState.rooms[i];
  return fcState.rooms[0];
}
function fcTtlLabel(sec) {
  if (!sec) return "";
  if (sec % 86400 === 0) return (sec / 86400) + "d";
  if (sec % 3600 === 0) return (sec / 3600) + "h";
  return Math.round(sec / 60) + "m";
}
function fcDisplayName(m) {
  // The room's display grammar (v0.5.4e, #267 amendment):
  // the OPERATOR name is who a member is in the room; the
  // node name is the machine underneath, shown secondarily.
  // Members and messages both carry opName, resolved by the
  // bridge from the author's signed presence lines.
  return String((m && (m.opName || m.authorName || m.name)) || "");
}
function fedichatUserMatches(prefix, members) {
  // Completion's one source: the signed roster as the room
  // holds it. Expelled or departed members are not on the
  // roster, so they never complete. Since v0.5.4e the
  // OPERATOR names the room holds complete too — the source
  // grows, it never swaps: node names and roster numbers
  // keep working. Pure — the bench drives this function
  // directly.
  var p = String(prefix || "").toLowerCase();
  return (members || []).filter(function (m) {
    return String(m.name || "").toLowerCase().indexOf(p) === 0
      || String(m.opName || "").toLowerCase().indexOf(p) === 0
      || String(m.n) === String(prefix || "");
  });
}
function fedichatTime(ts) {
  var d = new Date((ts || 0) * 1000);
  function p2(x) { return (x < 10 ? "0" : "") + x; }
  return p2(d.getHours()) + ":" + p2(d.getMinutes());
}
function fedichatHtml() {
  var h = "<h3>FediChat</h3>";
  h += "<p style='color:var(--dim)'>The fedinet's signed rooms — the operators talk, their nodes carry the messages, and their Navis are in the room. Every room's log is append-only and hash-chained on its member nodes; admission is the signed roster, an invite room's member set is its own boundary, and there is no server and no hub. <span class='mono'>/navi</span> asks your Navi, <span class='mono'>/navi -&lt;user&gt;</span> asks a member's Navi, <span class='mono'>/navinavi -&lt;user&gt;</span> sets your Navi to ask theirs — and <span class='mono'>/me</span> acts.</p>";
  if (CONFIG.backend !== "live") return h + "<div class='card'>FediChat runs through the live bridge — switch the backend to Live in Settings. The rooms themselves live on the member nodes, not on any server.</div>";
  if (!fcState && !fcFetching) refreshFedichat(false);
  if (!fcState) return h + "<div class='card'>Opening the room…</div>";
  if (fcMsg) h += "<div class='card' style='border-color:#f87171;color:#fca5a5'>" + esc(fcMsg) + "</div>";
  if (!fcState.rooms || !fcState.rooms.length) return h + "<div class='card'>No room yet — this node is not on any fedinet's roster. The roster is the room's admission: join a fedinet from the Federation page and its room opens here.</div>";
  var room = fedichatRoom();
  var st = fcState.settings || {};
  // The room as a chat surface (v0.5.5e, #270), grown by
  // Build 2 (v0.5.7e): the frame fills the view, the lines
  // scroll inside it, the composer is docked at the foot.
  // The members rail lives in the tray at the upper left
  // (fedichatTrayHtml — now the room switcher and the door
  // to new rooms), the room's facts and settings in the
  // floating About & settings window (fedichatWinHtml).
  var said = {};
  room.messages.forEach(function (m) { said[m.author] = true; });
  var byId = {};
  room.messages.forEach(function (m) { byId[m.id] = m; });
  var lines = room.messages.map(function (m) {
    var disp = fcDisplayName(m);
    var nodeBit = (m.opName && m.authorName && m.opName !== m.authorName) ? " · " + esc(m.authorName) : "";
    var who = "<b>" + esc(disp) + "</b> <span style='color:var(--dim)'>· № " + m.n + nodeBit + "</span>";
    var when = "<span style='color:var(--dim);font-size:11px'>" + fedichatTime(m.ts) + "</span>";
    var idAttr = " id='fc-line-" + esc(m.id) + "'";
    var cls = "fc-line" + (fcJump && fcJump === m.id ? " hl" : "");
    var quoteBit = "";
    if (m.quote && m.quote.id) {
      var qm = byId[m.quote.id];
      if (m.quoteState === "verified" && qm) quoteBit = "<div class='fc-quote'>❝ <b>" + esc(m.quote.name || "") + "</b>: " + esc(String(qm.body || "").slice(0, 140)) + " <span style='font-size:11px'>· line " + esc(m.quote.id.slice(0, 8)) + " · ✓ verified against the chain</span></div>";
      else quoteBit = "<div class='fc-quote fc-quote-bad'>❝ <b>" + esc(m.quote.name || "") + "</b> · line " + esc(m.quote.id.slice(0, 8)) + " · quote unverifiable — the quoted line is not held here, or no longer matches its id; it is flagged, never silently trusted</div>";
    }
    var meta = "<div class='fc-linemeta'><button type='button' class='ghost fc-mini' data-fcquote='" + esc(m.id) + "'>Quote</button></div>";
    if (m.kind === "me") return "<div class='" + cls + "'" + idAttr + " style='font-style:italic;color:var(--dim)'>* " + esc(disp) + " " + esc(m.body) + " " + when + "</div>";
    if (m.kind === "navi-a" || m.kind === "navi-na") return "<div class='" + cls + "'" + idAttr + "><span class='chip' style='font-size:10px'>NAVI</span> " + when + quoteBit + "<div style='white-space:pre-wrap'><b>Navi(" + esc(disp) + "):</b> " + esc(m.body) + "</div>" + meta + "</div>";
    if (m.kind === "navi-q") {
      return "<div class='" + cls + "'" + idAttr + ">" + who + " " + when + " <span class='chip' style='font-size:10px'>ASKS NAVI</span>" + quoteBit + "<div style='white-space:pre-wrap'>" + esc(m.body) + "</div>" + meta + "</div>";
    }
    if (m.kind === "navi-nq") {
      return "<div class='" + cls + "'" + idAttr + "><b>Navi(" + esc(disp) + ")</b> " + when + " <span class='chip' style='font-size:10px'>ASKS NAVI</span>" + quoteBit + "<div style='white-space:pre-wrap'>" + esc(m.body) + "</div>" + meta + "</div>";
    }
    if (m.kind === "media") {
      var src = "/api/fedichat/media?fed=" + encodeURIComponent(room.fedId) + "&room=" + encodeURIComponent(room.roomId || "") + "&sha=" + encodeURIComponent((m.media || {}).sha || "");
      return "<div class='" + cls + "'" + idAttr + ">" + who + " " + when + quoteBit + "<div class='fc-media'><img src='" + src + "' alt='" + esc(m.body || "image") + "'></div><div style='color:var(--dim);font-size:12px'>" + esc(m.body || "image") + "</div>" + meta + "</div>";
    }
    if (m.kind === "referral") {
      var cases = (fcState.cases || []);
      var fileBit = cases.length ? "<div class='row' style='margin-top:6px'><select id='fc-file-case-" + esc(m.id) + "' aria-label='Case to file this lead into'>" + cases.map(function (c) { return "<option value='" + esc(c.slug) + "'>" + esc(c.title) + "</option>"; }).join("") + "</select><button type='button' class='ghost' data-fcfile='" + esc(m.id) + "' style='min-height:44px'>File into the case</button></div>" : "<div style='color:var(--dim);font-size:12px;margin-top:4px'>No case stands on this node to file it into yet.</div>";
      return "<div class='" + cls + "'" + idAttr + "><div class='fc-lead'><span class='chip' style='font-size:10px'>LEAD</span> " + when + " <b>" + esc((m.ref || {}).title || "Navi referral") + "</b><div style='white-space:pre-wrap'>" + esc(m.body) + "</div><div style='color:var(--dim);font-size:12px'>A referral from Navi(" + esc(disp) + "), out of a Navi-to-Navi exchange — a lead, never a filing. It files only by your act, through the case door.</div>" + fileBit + "</div>" + meta + "</div>";
    }
    return "<div class='" + cls + "'" + idAttr + ">" + who + " " + when + quoteBit + "<div style='white-space:pre-wrap'>" + esc(m.body) + "</div>" + meta + "</div>";
  }).join("");
  var seats = room.members.filter(function (m) { return !said[m.nodeId]; }).map(function (m) {
    var nodeBit = (m.opName && m.name && m.opName !== m.name) ? " · " + esc(m.name) : "";
    return "<div class='fc-line' style='color:var(--dim)'>— " + esc(fcDisplayName(m)) + " · № " + m.n + nodeBit + " is in the room; no lines from them are held here yet.</div>";
  }).join("");
  var members = room.members.map(function (m) {
    var nodeBit = (m.opName && m.name && m.opName !== m.name) ? " · " + esc(m.name) : "";
    return "<div style='padding:3px 0'><b>" + esc(fcDisplayName(m)) + "</b> <span style='color:var(--dim)'>· № " + m.n + nodeBit + (m.self ? " · this node" : "") + "</span></div>";
  }).join("");
  var presenceBits = [];
  var pres = room.presence || {};
  room.members.forEach(function (m) { if (pres[m.nodeId]) presenceBits.push(esc(fcDisplayName(m)) + " is " + esc(pres[m.nodeId]) + "…"); });
  (room.naviThinking || []).forEach(function (nid) {
    var mm = null;
    room.members.forEach(function (x) { if (x.nodeId === nid) mm = x; });
    if (mm) presenceBits.push("Navi (" + esc(fcDisplayName(mm)) + ") is thinking…");
  });
  var facts = "";
  if (room.kind === "invite") facts += " <span class='chip' style='font-size:10px'>INVITE</span>";
  if (room.kind === "dm") facts += " <span class='chip' style='font-size:10px'>DM</span>";
  if (room.ttlSeconds) facts += " <span class='chip' style='font-size:10px'>VANISHING · " + esc(fcTtlLabel(room.ttlSeconds)) + "</span>";
  if (room.dissolved) facts += " <span class='chip' style='font-size:10px;color:#f87171'>DISSOLVED</span>";
  if (room.caseSlug) facts += " <button type='button' class='ghost fc-mini' data-fccase='" + esc(room.caseSlug) + "'>Case: " + esc(room.caseTitle || room.caseSlug) + "</button>";
  h = "";
  if (fcMsg) h += "<div class='card' style='border-color:#f87171;color:#fca5a5'>" + esc(fcMsg) + "</div>";
  h += "<div class='fc-room'>"
    + "<div class='fc-topbar'><button class='ghost' id='fc-users' aria-label='Room members and channels'><svg width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round' style='vertical-align:-3px'><path d='M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2'/><circle cx='9' cy='7' r='4'/><path d='M22 21v-2a4 4 0 0 0-3-3.87'/><path d='M16 3.13a4 4 0 0 1 0 7.75'/></svg> Members</button>"
    + "<b class='fc-room-name'>" + esc(room.room) + "</b><span style='color:var(--dim)'>" + esc(room.name) + "</span>" + facts
    + "<button class='ghost' id='fc-settings' style='margin-left:auto' aria-label='About this room and its settings'>⚙ Settings</button></div>"
    + "<div class='card' id='fc-log'>" + (lines || "<div style='color:var(--dim)'>The room is quiet — no lines held yet. The first line opens it.</div>") + seats + "</div>"
    + "<div class='fc-presence' id='fc-presence'>" + presenceBits.join(" · ") + "</div>"
    + "<div class='fc-searchrow'><input class='grow' id='fc-search' type='text' autocomplete='off' placeholder='Search this room's log' aria-label='Search this room's log'><button class='ghost' id='fc-search-go' type='button'>Search</button></div>"
    + "<div id='fc-results'>" + (fcSearchResults ? (fcSearchResults.length ? fcSearchResults.map(function (r) { return "<div style='padding:2px 0'><button type='button' class='ghost fc-mini' data-fcjump='" + esc(r.id) + "'>" + esc(r.opName || r.authorName || "") + " · " + fedichatTime(r.ts) + " — " + esc(r.snippet || "") + "</button></div>"; }).join("") : "<div style='color:var(--dim);font-size:12px'>No held line matches.</div>") : "") + "</div>"
    + "<div id='fc-complete' class='fc-complete'></div>"
    + (fcQuote ? "<div class='fc-quotechip' style='color:var(--dim);font-size:12px'>Quoting a held line — it rides your next post, proven against the chain. <button type='button' class='ghost fc-mini' data-fcquote-clear='1'>Drop the quote</button></div>" : "")
    + "<form id='fc-form' class='row fc-composer'><input class='grow' id='fc-input' type='text' autocomplete='off' placeholder='Say it in " + esc(room.room) + " — /navi asks Navi, /navi -member asks theirs, /navinavi -member sets yours to ask theirs, /me acts' aria-label='FediChat message'><button type='button' class='ghost' id='fc-attach' aria-label='Attach an image'>📎</button><input type='file' id='fc-file' accept='image/jpeg,image/png,image/gif,image/webp' style='display:none' aria-label='Image file'><button class='act' type='submit'>Post</button></form>"
    + "<div class='fc-hint' style='color:var(--dim);font-size:12px;margin-top:4px'>Enter posts; Tab completes <span class='mono'>/navi -&lt;user&gt;</span> and <span class='mono'>/navinavi -&lt;user&gt;</span> against the roster; 📎 posts an image (2 MiB cap); Quote hangs a held line on yours; search finds a line and jumps to it.</div>"
    + "<button class='ghost fc-settings-phone' id='fc-settings-m'>⚙ Room settings &amp; about</button>"
    + "</div>";
  if (fcTrayOpen || fcWinOpen || fcCreateOpen) h += "<div id='fc-shade'></div>";
  if (fcTrayOpen) h += fedichatTrayHtml(room, members);
  if (fcWinOpen) h += fedichatWinHtml(room, st);
  if (fcCreateOpen) h += fedichatCreateHtml(room);
  return h;
}
function fedichatTrayHtml(room, members) {
  // The members tray (#270, grown by Build 2): the room's
  // name, the ROOM LIST — every room this node belongs to,
  // the switcher that replaced the in-page tabbar — the door
  // to a new room, and the users, the old rail's rows
  // verbatim, each with its Message door into a DM.
  // Rendered only while open and portaled to <body> by
  // render(), like the wire reader.
  var h = "<div id='fc-tray' role='dialog' aria-modal='true' aria-label='Room members and channels'>";
  h += "<div class='fc-ov-head'><div class='fc-ov-title'>" + esc(room.room) + "</div><button type='button' class='fc-ov-x' id='fc-tray-x' aria-label='Close the members tray'>×</button></div>";
  h += "<div style='color:var(--dim);font-size:12px'>" + esc(room.name) + " · signed roster · " + room.members.length + " members</div>";
  h += "<div style='margin-top:14px'><b>Rooms</b></div>" + fcState.rooms.map(function (r) {
    var val = r.fedId + ((r.roomId || "") ? "|" + r.roomId : "");
    var tag = r.kind === "invite" ? " · invite" : (r.kind === "dm" ? " · dm" : "");
    return "<div style='padding:3px 0'><button class='" + (r.fedId === room.fedId && (r.roomId || "") === (room.roomId || "") ? "act" : "ghost") + "' data-fcroom='" + esc(val) + "' style='width:100%;text-align:left;min-height:44px'>" + esc(r.room) + tag + "</button></div>";
  }).join("");
  h += "<div style='padding:3px 0'><button class='ghost' data-fccreate='1' style='width:100%;text-align:left;min-height:44px'>+ New room…</button></div>";
  h += "<div style='margin-top:14px'><b>Members</b></div>" + room.members.map(function (m) {
    var nodeBit = (m.opName && m.name && m.opName !== m.name) ? " · " + esc(m.name) : "";
    return "<div class='row' style='padding:3px 0'><span class='grow'><b>" + esc(fcDisplayName(m)) + "</b> <span style='color:var(--dim)'>· № " + m.n + nodeBit + (m.self ? " · this node" : "") + "</span></span>" + (m.self ? "" : "<button type='button' class='ghost fc-mini' data-fcdm='" + esc(m.nodeId) + "'>Message</button>") + "</div>";
  }).join("");
  h += "</div>";
  return h;
}
function fedichatWinHtml(room, st) {
  // The About & settings window (#270): the explanation and
  // the header card's facts, verbatim in substance — the
  // room's name and roster, the chain's verdict, the
  // carriage sentence, the retention state, the last round,
  // Sync now — above the settings controls, unchanged and
  // switch-gated in their home. Build 2 (v0.5.7e) grows the
  // facts: the room's kind, its vanishing posture in plain
  // words, its case tie, and — for the room's creator — the
  // member-set controls and the dissolution door, every one
  // a creator-signed revision at the bridge. Rendered only
  // while open; portaled to <body> by render().
  var h = "<div id='fc-win' role='dialog' aria-modal='true' aria-label='About this room and its settings'>";
  h += "<div class='fc-ov-head'><div class='fc-ov-title'>About " + esc(room.room) + "</div><button type='button' class='fc-ov-x' id='fc-win-x' aria-label='Close room settings'>×</button></div>";
  h += "<p style='color:var(--dim)'>The fedinet's signed rooms — the operators talk, their nodes carry the messages, and their Navis are in the room. Every room's log is append-only and hash-chained on its member nodes; admission is the signed roster, and there is no server and no hub. <span class='mono'>/navi</span> asks your Navi, <span class='mono'>/navi -&lt;user&gt;</span> asks a member's Navi — questions and answers post in the open — and <span class='mono'>/me</span> acts.</p>";
  h += "<div class='row'><b style='font-size:16px'>" + esc(room.room) + "</b><span style='color:var(--dim)'>" + esc(room.name) + " · signed roster · " + room.members.length + " members · " + room.held + " lines held</span><button class='ghost' id='fc-sync'>Sync now</button></div>";
  if (room.chain && room.chain.ok) h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Chain ✓ verified — history as held is tamper-evident.</div>";
  else if (room.chain) h += "<div style='color:#fca5a5;font-size:12px;margin-top:4px'>Chain: " + esc(room.chain.reason || "broken") + " — named, never silently healed.</div>";
  if (room.kind === "general") h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>The general room — the fedinet's own room. Every member of the roster is a member of it.</div>";
  if (room.kind === "invite") h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>An invite room — its member set is its boundary: only member nodes are offered its lines, hold its log, or see that it stands. Membership changes are the creator's, signed and auditable.</div>";
  if (room.kind === "dm") h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>A direct message — a room of exactly two members; only their nodes hold its log.</div>";
  if (room.ttlSeconds) h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Vanishing: lines older than " + esc(fcTtlLabel(room.ttlSeconds)) + " are pruned on every honoring node, in the retention shape — the newest older line kept as the chain's floor. <b>Vanishing is honored by every member node; it is not a proof of deletion.</b></div>";
  if (room.dissolved) h += "<div style='color:#fca5a5;font-size:12px;margin-top:4px'>This room was dissolved by its creator — honoring nodes pruned it to its floor, and no new lines land in it.</div>";
  if (room.caseSlug) h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Tied to a case: <button type='button' class='ghost fc-mini' data-fccase='" + esc(room.caseSlug) + "'>" + esc(room.caseTitle || room.caseSlug) + "</button> — the case carries the link back.</div>";
  if (st.relay) h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Relay carriage is ON for this node: when a member is reached through a relay, the carrier can read the lines it carries. Turn it off below and the room only travels direct.</div>";
  else h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Relay carriage is OFF for this node: the room travels direct only — an unreachable member waits for a direct road.</div>";
  h += "<div style='color:var(--dim);font-size:12px'>Retention: " + (st.retention === "30d" ? "30 days — older lines prune from this node's log, with the newest older line kept as the chain's floor" : "keep — this node holds the room's whole history") + ".</div>";
  if (room.syncNotes && room.syncNotes.length) h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Last round: " + room.syncNotes.map(function (n) { return esc(n); }).join(" · ") + "</div>";
  // The creator's controls: the member set and the case
  // tie are creator-signed revisions; dissolution likewise.
  if (room.isCreator && room.kind === "invite" && !room.dissolved) {
    var roster = room.members;
    fcState.rooms.forEach(function (r) { if (r.fedId === room.fedId && r.kind === "general") roster = r.members; });
    var inRoom = {};
    room.members.forEach(function (m) { inRoom[m.nodeId] = true; });
    h += "<div style='margin-top:14px'><b>The room's members — your room, your set</b><div style='color:var(--dim);font-size:12px'>Ticked members are offered the room and hold its log; unticking a member stops new lines reaching their node (what they already hold stays held — history is not rewritten).</div>"
      + roster.map(function (m) { return "<label class='row' style='padding:3px 0'><input type='checkbox' data-fcmember='" + esc(m.nodeId) + "'" + (inRoom[m.nodeId] ? " checked" : "") + (m.self ? " disabled" : "") + "> <span><b>" + esc(fcDisplayName(m)) + "</b> <span style='color:var(--dim)'>· № " + m.n + (m.self ? " · this node" : "") + "</span></span></label>"; }).join("")
      + "<div class='row' style='margin-top:6px'><button class='ghost' data-fcsavemembers='1' style='min-height:44px'>Save the member set</button></div></div>";
    var cases = (fcState.cases || []);
    if (cases.length) h += "<div style='margin-top:10px'><b>Tie to a case</b><div class='row' style='margin-top:4px'><select id='fc-tie-case' aria-label='Case to tie this room to'><option value=''>— no case —</option>" + cases.map(function (c) { return "<option value='" + esc(c.slug) + "'" + (room.caseSlug === c.slug ? " selected" : "") + ">" + esc(c.title) + "</option>"; }).join("") + "</select><button class='ghost' data-fctie='1' style='min-height:44px'>Tie it</button></div></div>";
    h += "<div style='margin-top:10px'><b>Dissolve the room</b><div style='color:var(--dim);font-size:12px'>A signed dissolution: honoring nodes prune the room to its floor and mark it dissolved. It cannot be undone.</div><div class='row' style='margin-top:4px'><button class='ghost' data-fcdissolve='1' style='min-height:44px;color:#f87171'>Dissolve " + esc(room.room) + "</button></div></div>";
  }
  // Room settings — the operator's rulings, as settings.
  h += "<div style='margin-top:14px'><b>This node's room settings</b>"
    + "<div class='row' style='margin-top:8px'><span class='grow'>Navi answers in " + esc(room.room) + " <span style='color:var(--dim)'>— the muzzle: off means she speaks when asked; on means this room hears nothing from her.</span></span><button class='switch' id='fc-muzzle' role='switch' aria-checked='" + (room.muzzled ? "true" : "false") + "' aria-label='Muzzle Navi in this room'><span class='knob'></span></button><span>" + (room.muzzled ? "Muzzled" : "She answers") + "</span></div>"
    + "<div class='row' style='margin-top:8px'><span class='grow'>Relay carriage <span style='color:var(--dim)'>— let this node's room lines travel through relays (the carrier can read them; the About text above says so when it is on).</span></span><button class='switch' id='fc-relay' role='switch' aria-checked='" + (st.relay ? "true" : "false") + "' aria-label='Relay carriage for FediChat'><span class='knob'></span></button><span>" + (st.relay ? "Allowed" : "Direct only") + "</span></div>"
    + "<div class='row' style='margin-top:8px'><span class='grow'>Retention <span style='color:var(--dim)'>— this node's copy of the log.</span></span><button class='" + (st.retention === "keep" ? "act" : "ghost") + "' data-fcret='keep'>Keep everything</button><button class='" + (st.retention === "30d" ? "act" : "ghost") + "' data-fcret='30d'>30 days</button></div>"
    + "</div>";
  h += "</div>";
  return h;
}
function fedichatCreateHtml(room) {
  // The create-room overlay (Build 2): name, kind, the
  // member picker off the roster, the vanishing TTL, the
  // optional case tie. The bridge signs what the creator's
  // node mints; a DM is exactly one other member.
  var fed = null;
  fcState.rooms.forEach(function (r) { if (r.fedId === room.fedId && !fed) fed = r; });
  var roster = fed ? fed.members : room.members;
  var cases = (fcState.cases || []);
  var h = "<div id='fc-win' role='dialog' aria-modal='true' aria-label='Create a room'>";
  h += "<div class='fc-ov-head'><div class='fc-ov-title'>New room in " + esc(room.name) + "</div><button type='button' class='fc-ov-x' id='fc-win-x' aria-label='Close room creation'>×</button></div>";
  h += "<div class='row' style='margin-top:6px'><input class='grow' id='fc-new-name' type='text' placeholder='Room name — e.g. Night Desk' aria-label='New room name'></div>";
  h += "<div class='row' style='margin-top:6px'><span class='grow' style='color:var(--dim)'>Kind</span><select id='fc-new-kind' aria-label='Room kind'><option value='invite'>Invite room — a named set of members</option><option value='dm'>Direct message — exactly one other member</option></select></div>";
  h += "<div style='margin-top:8px'><b>Members</b><div style='color:var(--dim);font-size:12px'>You are always in your own room. Only ticked members' nodes are offered the room, hold its log, or see it.</div>"
    + roster.map(function (m) { return "<label class='row' style='padding:3px 0'><input type='checkbox' data-fcnewmember='" + esc(m.nodeId) + "'" + (m.self ? " checked disabled" : "") + "> <span><b>" + esc(fcDisplayName(m)) + "</b> <span style='color:var(--dim)'>· № " + m.n + (m.self ? " · this node" : "") + "</span></span></label>"; }).join("") + "</div>";
  h += "<div class='row' style='margin-top:8px'><span class='grow' style='color:var(--dim)'>Vanishing</span><select id='fc-new-ttl' aria-label='Vanishing TTL'><option value=''>Never — the room keeps its lines</option><option value='3600'>Lines vanish after 1 hour</option><option value='86400'>Lines vanish after 24 hours</option><option value='604800'>Lines vanish after 7 days</option></select></div>";
  h += "<div style='color:var(--dim);font-size:12px'>Vanishing is honored by every member node; it is not a proof of deletion.</div>";
  if (cases.length) h += "<div class='row' style='margin-top:8px'><span class='grow' style='color:var(--dim)'>Tie to a case</span><select id='fc-new-case' aria-label='Case to tie the new room to'><option value=''>— no case —</option>" + cases.map(function (c) { return "<option value='" + esc(c.slug) + "'>" + esc(c.title) + "</option>"; }).join("") + "</select></div>";
  h += "<div class='row' style='margin-top:10px'><button class='act' id='fc-new-go' style='min-height:44px'>Create the room</button></div>";
  h += "</div>";
  return h;
}
// The Vault create sheet's wiring (v0.5.9e, #279): bound
// by render() after the sheet is portaled to <body>, the
// #270 overlay pattern. Every control fires the same act
// as its desktop-card twin — the sheet is a second door to
// the same machinery, never a second implementation. A
// successful create closes the sheet and the new note
// opens in the reader, as the card's create does.
function bindVaultSheet() {
  var close = function () { vaultSheetOpen = false; render(); };
  var shade = document.getElementById("vault-shade");
  if (shade) shade.onclick = close;
  var x = document.getElementById("vault-sh-x");
  if (x) x.onclick = close;
  var create = document.getElementById("vault-sh-create");
  if (create) create.onclick = function () {
    var nm = document.getElementById("vault-sh-name"); var name = nm ? nm.value.trim().replace(/\.md$/i, "") : "";
    if (!name) return;
    var fo = document.getElementById("vault-sh-folder"); var folder = fo ? fo.value : "";
    var tp = document.getElementById("vault-sh-template");
    var content = "# " + name + "\n\n";
    if (tp && tp.value) { var tpl = vaultState.notes.filter(function (n) { return n.path === decodeURIComponent(tp.value); })[0]; if (tpl) content = tpl.content.split("Untitled").join(name); }
    var path = (folder ? folder + "/" : "") + name + ".md";
    vaultSel = path;
    vaultSheetOpen = false;
    vaultApply("save", { path: path, content: content });
  };
  var addFolder = document.getElementById("vault-sh-addfolder");
  if (addFolder) addFolder.onclick = function () { var fi = document.getElementById("vault-sh-foldername"); var name = fi ? fi.value.trim() : ""; if (name) { vaultSheetOpen = false; vaultApply("folder", { path: name }); } };
  var daily = document.getElementById("vault-sh-daily");
  if (daily) daily.onclick = function () {
    var today = new Date().toISOString().slice(0, 10); var path = "Daily/" + today + ".md";
    vaultSheetOpen = false;
    if (vaultState && vaultState.notes.some(function (n) { return n.path === path; })) vaultOpenPath(path);
    else { vaultSel = path; vaultApply("save", { path: path, content: "# " + today + "\n\n## Today\n\n- \n" }); }
  };
  var exp = document.getElementById("vault-sh-export");
  if (exp) exp.onclick = function () {
    vaultSheetOpen = false;
    if (CONFIG.backend === "live") { window.open("/api/vault/export", "_blank"); vaultMsg = "Exported the vault as a .zip of the folder — unzip it and Obsidian opens it as-is."; render(); return; }
    var bundle = vaultState.notes.map(function (n) { return "<!-- " + n.path + " -->\n\n" + n.content; }).join("\n\n---\n\n");
    var url = URL.createObjectURL(new Blob([bundle], { type: "text/markdown" }));
    var a = document.createElement("a"); a.href = url; a.download = "apex-vault-export.md"; a.click(); URL.revokeObjectURL(url);
    vaultMsg = "Mock mode exports a single markdown bundle; the live bridge exports the real .zip of the folder.";
    render();
  };
}

function bindFedichatOverlays() {
  // The tray/window close furniture. The overlays are
  // portaled to <body> by render(); this runs from the
  // portal step and from wireFedichat, so the controls bind
  // whichever copy of the markup is live. Assignment only —
  // re-binding is harmless.
  var sh = document.getElementById("fc-shade");
  if (sh) sh.onclick = function () { fcTrayOpen = false; fcWinOpen = false; fcCreateOpen = false; render(); };
  var tx = document.getElementById("fc-tray-x");
  if (tx) tx.onclick = function () { fcTrayOpen = false; render(); };
  var wx = document.getElementById("fc-win-x");
  if (wx) wx.onclick = function () { fcWinOpen = false; fcCreateOpen = false; render(); };
}
function wireFedichat() {
  var room = fedichatRoom();
  document.querySelectorAll("[data-fcroom]").forEach(function (b) { b.onclick = function () { var v = b.getAttribute("data-fcroom") || ""; var parts = v.split("|"); fcFed = parts[0]; fcRoomId = parts[1] || ""; fcCompleteIdx = 0; fcTrayOpen = false; fcSearchResults = null; fcJump = ""; render(); }; });
  document.querySelectorAll("[data-fccreate]").forEach(function (b) { b.onclick = function () { fcTrayOpen = false; fcCreateOpen = true; render(); }; });
  document.querySelectorAll("[data-fcdm]").forEach(function (b) { b.onclick = function () { if (!room) return; fedichatAct("create-dm", { fed: room.fedId, member: b.getAttribute("data-fcdm") }, function (j) { if (j && j.roomId !== undefined) { fcFed = room.fedId; fcRoomId = j.roomId; } fcTrayOpen = false; render(); }); }; });
  document.querySelectorAll("[data-fcquote]").forEach(function (b) { b.onclick = function () { fcQuote = b.getAttribute("data-fcquote"); render(); var inp = document.getElementById("fc-input"); if (inp) inp.focus(); }; });
  document.querySelectorAll("[data-fcquote-clear]").forEach(function (b) { b.onclick = function () { fcQuote = null; render(); }; });
  document.querySelectorAll("[data-fcjump]").forEach(function (b) { b.onclick = function () { fcJump = b.getAttribute("data-fcjump"); render(); }; });
  document.querySelectorAll("[data-fccase]").forEach(function (b) { b.onclick = function () { var slug = b.getAttribute("data-fccase"); fcWinOpen = false; goView("CaseFiles"); refreshCaseDetail(slug); }; });
  document.querySelectorAll("[data-fcfile]").forEach(function (b) { b.onclick = function () { if (!room) return; var sel = document.getElementById("fc-file-case-" + b.getAttribute("data-fcfile")); if (sel && sel.value) fedichatAct("file-referral", { fed: room.fedId, room: room.roomId || "", line: b.getAttribute("data-fcfile"), "case": sel.value }); }; });
  document.querySelectorAll("[data-fcsavemembers]").forEach(function (b) { b.onclick = function () { if (!room) return; var members = []; document.querySelectorAll("[data-fcmember]").forEach(function (c) { if (c.checked || c.disabled) members.push(c.getAttribute("data-fcmember")); }); fedichatAct("set-members", { fed: room.fedId, room: room.roomId || "", members: members }); }; });
  document.querySelectorAll("[data-fctie]").forEach(function (b) { b.onclick = function () { if (!room) return; var sel = document.getElementById("fc-tie-case"); fedichatAct("tie-case", { fed: room.fedId, room: room.roomId || "", "case": sel ? sel.value : "" }); }; });
  document.querySelectorAll("[data-fcdissolve]").forEach(function (b) { b.onclick = function () { if (!room) return; fedichatAct("dissolve", { fed: room.fedId, room: room.roomId || "" }); }; });
  var createGo = document.getElementById("fc-new-go");
  if (createGo && room) createGo.onclick = function () {
    var kindSel = document.getElementById("fc-new-kind");
    var kind = kindSel ? kindSel.value : "invite";
    var members = [];
    document.querySelectorAll("[data-fcnewmember]").forEach(function (c) { if (c.checked || c.disabled) members.push(c.getAttribute("data-fcnewmember")); });
    if (kind === "dm") {
      var others = members.filter(function (x) { return x !== (fcState && fcState.nodeId); });
      if (others.length !== 1) { fcMsg = "A direct message takes exactly one other member — tick one."; render(); return; }
      fedichatAct("create-dm", { fed: room.fedId, member: others[0] }, function (j) { if (j && j.roomId !== undefined) { fcFed = room.fedId; fcRoomId = j.roomId; } fcCreateOpen = false; render(); });
      return;
    }
    var nameEl = document.getElementById("fc-new-name");
    var ttlEl = document.getElementById("fc-new-ttl");
    var caseEl = document.getElementById("fc-new-case");
    var extra = { fed: room.fedId, name: nameEl ? nameEl.value : "", members: members };
    if (ttlEl && ttlEl.value) extra.ttlSeconds = parseInt(ttlEl.value, 10);
    if (caseEl && caseEl.value) extra["case"] = caseEl.value;
    fedichatAct("create-room", extra, function (j) { if (j && j.roomId !== undefined) { fcFed = room.fedId; fcRoomId = j.roomId; } fcCreateOpen = false; render(); });
  };
  var usersBtn = document.getElementById("fc-users");
  if (usersBtn) usersBtn.onclick = function () { fcTrayOpen = true; render(); };
  var setBtn = document.getElementById("fc-settings");
  if (setBtn) setBtn.onclick = function () { fcWinOpen = true; render(); };
  var setBtnM = document.getElementById("fc-settings-m");
  if (setBtnM) setBtnM.onclick = function () { fcWinOpen = true; render(); };
  bindFedichatOverlays();
  document.querySelectorAll("[data-fcret]").forEach(function (b) { b.onclick = function () { fedichatAct("settings", { retention: b.getAttribute("data-fcret") }); }; });
  var sync = document.getElementById("fc-sync");
  if (sync && room) sync.onclick = function () { fedichatAct("sync", { fed: room.fedId }); };
  var mz = document.getElementById("fc-muzzle");
  if (mz && room) mz.onclick = function () { fedichatAct("settings", { fed: room.fedId, room: room.roomId || "", muzzle: !room.muzzled }); };
  var rl = document.getElementById("fc-relay");
  if (rl && fcState) rl.onclick = function () { fedichatAct("settings", { relay: !(fcState.settings || {}).relay }); };
  var searchGo = document.getElementById("fc-search-go");
  var searchIn = document.getElementById("fc-search");
  if (searchGo && room) searchGo.onclick = function () { if (searchIn && searchIn.value.trim()) fedichatSearch(searchIn.value.trim(), room); };
  if (searchIn && room) searchIn.addEventListener("keydown", function (e) { if (e.key === "Enter") { e.preventDefault(); if (searchIn.value.trim()) fedichatSearch(searchIn.value.trim(), room); } });
  var attach = document.getElementById("fc-attach");
  var fileIn = document.getElementById("fc-file");
  if (attach && fileIn) attach.onclick = function () { fileIn.click(); };
  if (fileIn && room) fileIn.addEventListener("change", function () {
    var f = fileIn.files && fileIn.files[0];
    if (!f) return;
    var rd = new FileReader();
    rd.onload = function () {
      var s = String(rd.result || "");
      var b64 = s.indexOf(",") >= 0 ? s.slice(s.indexOf(",") + 1) : s;
      fedichatAct("media", { fed: room.fedId, room: room.roomId || "", name: f.name, mime: f.type, dataB64: b64 });
    };
    rd.readAsDataURL(f);
  });
  var form = document.getElementById("fc-form");
  var input = document.getElementById("fc-input");
  if (form && input && room) {
    // One post road (v0.5.5e, #270): the Post button's
    // submit and Enter in the composer both land here —
    // the main chat's semantics, where Enter sends. A
    // pending quote rides the post and clears with it.
    var postLine = function () {
      var v = input.value;
      if (!v.trim()) return;
      input.value = "";
      var extra = { fed: room.fedId, room: room.roomId || "", body: v };
      if (fcQuote) extra.quote = fcQuote;
      fcQuote = null;
      fedichatPresence("", room);
      fedichatAct("post", extra);
    };
    form.onsubmit = function (e) {
      e.preventDefault();
      postLine();
    };
    var complete = function () {
      var m = /^\/(navinavi|navi)\s+-(\S*)$/.exec(input.value);
      var box = document.getElementById("fc-complete");
      if (!m) { if (box) box.innerHTML = ""; return []; }
      var matches = fedichatUserMatches(m[2], room.members);
      if (box) box.innerHTML = matches.map(function (mm) { return "<button type='button' class='ghost' data-fccomplete='" + esc(fcDisplayName(mm)) + "' data-fccmd='" + m[1] + "'>" + esc(fcDisplayName(mm)) + " · № " + mm.n + "</button>"; }).join(" ");
      document.querySelectorAll("[data-fccomplete]").forEach(function (b) { b.onclick = function () { input.value = "/" + (b.getAttribute("data-fccmd") || "navi") + " -" + b.getAttribute("data-fccomplete") + " "; input.focus(); }; });
      return matches;
    };
    input.addEventListener("input", function () {
      fcCompleteIdx = 0; complete();
      var now = Date.now();
      if (now - fcPresenceAt > 4000) { fcPresenceAt = now; fedichatPresence("typing", room); }
    });
    input.addEventListener("keydown", function (e) {
      if (e.key === "Enter") { e.preventDefault(); postLine(); return; }
      if (e.key !== "Tab") return;
      var matches = complete();
      if (!matches.length) return;
      e.preventDefault();
      var pick = matches[fcCompleteIdx % matches.length];
      fcCompleteIdx++;
      var m2 = /^\/(navinavi|navi)\s+-/.exec(input.value);
      input.value = "/" + (m2 ? m2[1] : "navi") + " -" + fcDisplayName(pick) + " ";
    });
  }
  var log = document.getElementById("fc-log");
  if (log) {
    var jumpEl = fcJump ? document.getElementById("fc-line-" + fcJump) : null;
    if (jumpEl && jumpEl.scrollIntoView) jumpEl.scrollIntoView({ block: "center" });
    else log.scrollTop = log.scrollHeight;
  }
}

function federationHtml() {
  var s = federationState;
  var h = "<h3>Apex | Federation</h3>";
  var mv = meshViewState();
  if (mv.headline) h += "<p style='margin-top:2px'><span class='dot pdot' style='background:#3fb96f'></span> <b>" + esc(mv.headline) + "</b> <span style='color:var(--dim)'>— the headline is the state now, not a census of pairs.</span></p>";
  h += "<p style='color:var(--dim);max-width:680px'>A fedinet is a named set of machines whose gateway keeps a <b>signed roster</b>. A gateway card admits a node and introduces it — trust is a separate, deeper act: with trust-by-default on, the roster auto-pairs and auto-lends on this node. Every trust act, through any door, lands in the audit log at the foot of this page.</p>";
  if (CONFIG.backend === "live" && !s) {
    // 0.2.0e Build 10 (#195): render NEVER fetches. The state
    // is requested once per view entry by goView's enter hook;
    // this function only paints what it already holds — the
    // loading card while that one fetch is in flight, or,
    // when the answer did not set state, the answer's own
    // sentence, once, with a manual retry. The Build 9 shape
    // fetched from inside this render (and again from the
    // wiring pass) and re-rendered on every answer, so a
    // state-less answer — the field case: a pre-federation
    // bridge answering 404-shaped {"error": "not found"} —
    // repainted forever: the strobe.
    if (federationError) { h += federationErrorHtml(); return h; }
    h += "<div class='card'>Reading this node's fedinets…</div>"; return h;
  }
  if (!s) { h += "<div class='card'>Federation runs through the live bridge — switch the backend to Live in Settings.</div>"; return h; }
  if (fedMsg.sentence) h += "<p style='color:var(--accent);margin-top:8px'>" + esc(fedMsg.sentence) + "</p>";
  if (fedMsg.error && !fedMsg.errorFor) h += "<p style='color:#f58a8a;margin-top:8px'>" + esc(fedMsg.error) + "</p>";
  // The Mesh itself (v0.6.4e, #322): everything on this page
  // rides on the Mesh — fedinets, borrowing, FediChat, the
  // fleet — so the door's own toggle stands at the head of
  // the page, where its consequences are administered. It is
  // the SAME state the Mesh page's control flips: one stored
  // setting, one door (/api/mesh/settings), two surfaces. The
  // switch gate holds structurally, as everywhere on this
  // page: a press posts the flip and flips nothing locally;
  // the card re-renders from the bridge's state, and a
  // refusal lands on this card.
  h += "<div class='card' style='margin-top:12px'><b>The Mesh</b>" +
    "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'>" + fedSwitch(!!s.meshOn, "data-fed-mesh='1'", "The Mesh — this node's door") + "<span style='font-size:12px'><b>" + (s.meshOn ? "Mesh on" : "Mesh off") + "</b> — " + (s.meshOn ? "this node's door is open." : "this node's door is shut.") + "</span></div>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>Everything on this page rides on the Mesh: with it off, fedinets, borrowing, FediChat and Fleet Answers are dark — the roster may be signed and the ticks all set, and none of it moves. This is the same switch as the Mesh page's own control; there is one Mesh state, and both surfaces tell it.</p>" +
    (fedMsg.error && fedMsg.errorFor === "__mesh__" ? "<p style='color:#f58a8a;font-size:12px;margin-top:6px'>" + esc(fedMsg.error) + "</p>" : "") +
    "</div>";
  // The Badge card: the DHD step-up factor.
  h += "<div class='card' style='margin-top:12px'><b>The Operator Badge — the DHD step-up</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>" + (s.badgeEnrolled
      ? "A Badge is enrolled on this node (its fingerprint only — the picture itself never stays here). Through the DHD door, manage acts present it as the step-up and crown acts run under it with a named confirm."
      : "No Badge is enrolled on this node. Until one is, manage and crown acts cannot ride the DHD door — the ceremony cannot run, and the bridge says so by name. Enrollment happens in person, here, and stores only the Badge's fingerprint.") + "</p>" +
    "<div class='row' style='margin-top:8px'>" +
    (s.badgeEnrolled
      ? "<button class='ghost' data-fed-badge-clear='1' style='min-height:44px'>Clear the enrollment</button>"
      : "<button class='act' data-fed-badge-enroll='1' style='min-height:44px'>Enroll this dashboard's Operator Badge</button>") +
    (s.viaDhd ? "<span style='color:var(--dim);font-size:12px'>This view is through the DHD door — enrollment itself is in-person only.</span>" : "") +
    "</div></div>";
  // Fleet Answers (v0.6.1e, #293): this node's membership in
  // the fleet — a node-level posture, so it stands on its own
  // card, not on any one fedinet's. The switch gate holds, in
  // the page's own structural way: the switch never flips
  // locally — a press sends the act, the bridge answers with
  // the state it stored, and the card re-renders from that
  // answer alone; a refusal lands on THIS card as a sentence.
  var fl = s.fleet || { member: false, cap: 4, deadlineSec: 120 };
  h += "<div class='card' style='margin-top:12px'><b>Fleet Answers</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>A fleet ask divides one question into <b>whole sub-tasks</b> — each answered whole by the machine best placed for it, then composed into one answer with every contribution <b>signed by its machine</b>. Joining is this node's own posture: it offers this machine's answers to fleet asks from machines already paired here or on a joined fedinet's roster, under the same caps and consent as borrowing — and joining grants nobody anything by itself. <b>Off by default.</b> Join or leave at will; leaving mid-ask names the gap in that answer rather than hanging it.</p>" +
    (fedMsg.error && fedMsg.errorFor === "__fleet__" ? "<p style='color:#f58a8a;font-size:12px;margin-top:6px'>" + esc(fedMsg.error) + "</p>" : "") +
    "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'>" + fedSwitch(!!fl.member, "data-fed-fleet='1'", "Fleet Answers membership for this node") + "<span style='font-size:12px'><b>" + (fl.member ? "In the fleet" : "Not in the fleet") + "</b> — a fleet ask spans at most " + (fl.cap || 4) + " machines and ships by its " + (fl.deadlineSec || 120) + "s deadline, gaps badged by name.</span></div></div>";
  // Join: import a gateway card, or found a fedinet.
  h += "<div class='fed-grid'>";
  h += "<div class='card'><b>Join a fedinet</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>Import a gateway card — the green picture a gateway hands out. The import admits this node and introduces it to the gateway; it grants no trust by itself.</p>" +
    (s.invitationWaiting ? "<p style='margin-top:6px'>An invitation is waiting on this node (a container's preload, or a card dropped at <span class='mono'>config/federation/invitation.png</span>).</p><button class='act' data-fed-invitation='1' style='min-height:44px;margin-top:6px'>Import the waiting invitation</button>" : "") +
    "<div class='row' style='margin-top:8px'><button class='ghost' data-fed-import-btn='1' style='min-height:44px'>Import a gateway card PNG…</button><input id='fed-import-file' type='file' accept='.png,image/png' style='display:none' aria-label='Gateway card PNG file'></div>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:8px'>The known fedinet: <b>" + esc(s.knownDefault.name) + "</b>. " + esc(s.knownDefault.note) + "</p></div>";
  h += "<div class='card'><b>Found a fedinet</b>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>Mint a fedinet of your own: this node becomes its gateway, signs roster v1, and stands as its Founding Member — Node 1. Founding is a crown act: it needs its name spoken, and it audits.</p>" +
    "<div class='row' style='margin-top:8px'><input class='grow' id='fed-new-name' type='text' placeholder='Fedinet name — e.g. Rockport Mesh' aria-label='New fedinet name'><button class='act' data-fed-found='1' style='min-height:44px'>Found it</button></div>" +
    fedConfirmHtml("__found__") + "</div>";
  h += "</div>";
  if (fedMsg.error && fedMsg.errorFor === "__join__") h += "<p style='color:#f58a8a;margin-top:8px'>" + esc(fedMsg.error) + "</p>";
  // The fedinets themselves.
  (s.fedinets || []).forEach(function (f) {
    h += "<div class='card' style='margin-top:14px'><div class='row' style='flex-wrap:wrap;gap:8px'><b style='font-size:16px'>" + esc(f.name) + "</b>" +
      "<span style='color:var(--dim);font-size:12px'>" + (f.role === "gateway" ? "This node is the gateway" : "Gateway: " + esc(f.gatewayName || "") + " · <span class='mono'>" + esc(f.gatewayShort || "") + "</span>") + "</span>" +
      (f.ownNumber ? "<span style='color:var(--dim);font-size:12px'>You are Node " + f.ownNumber + "</span>" : "") +
      "<span style='color:var(--dim);font-size:12px'>Roster v" + f.rosterVersion + "</span></div>";
    if (f.staleSentence) h += "<p style='color:#f5b544;font-size:12px;margin-top:6px'>" + esc(f.staleSentence) + "</p>";
    if (fedMsg.error && fedMsg.errorFor === f.id) h += "<p style='color:#f58a8a;font-size:12px;margin-top:6px'>" + esc(fedMsg.error) + "</p>";
    if (f.gatewayCardPng) {
      h += "<div style='margin-top:10px'><img src='data:image/png;base64," + f.gatewayCardPng + "' alt='The gateway card for " + esc(f.name) + "' style='width:280px;max-width:100%;border-radius:8px'><div class='row' style='margin-top:6px'><a class='ghost' style='min-height:44px;display:inline-flex;align-items:center;padding:0 14px;text-decoration:none' href='data:image/png;base64," + f.gatewayCardPng + "' download='gateway-card-" + esc(f.name.replace(/[^A-Za-z0-9]+/g, "-")) + ".png'>Download the gateway card</a><span style='color:var(--dim);font-size:12px'>Green — admission + introduction, never trust by itself. Send the file itself, inside a .zip for chat apps.</span></div></div>";
    } else if (f.gatewayCardRefusal) {
      h += "<p style='color:#f5b544;font-size:12px;margin-top:10px'>" + esc(f.gatewayCardRefusal) + "</p>";
    }
    // Trust switches.
    h += "<div style='margin-top:12px;border-top:1px solid var(--line);padding-top:10px'><div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>TRUST — THIS NODE'S POSTURE</div>" +
      "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'>" + fedSwitch(f.trustByDefault, "data-fed-trust='" + esc(f.id) + "'", "Trust-by-default for " + f.name) + "<span style='font-size:12px'><b>Trust-by-default (deep)</b> — the signed roster auto-pairs and auto-lends here. Turning it on is a crown act.</span></div>" +
      "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'>" + fedSwitch(f.lendToNew, "data-fed-tick='" + esc(f.id) + "'", "Lend to new members automatically in " + f.name) + "<span style='font-size:12px'><b>Lend to new members automatically</b> — unticked, a member who joins later is pairing-only for this node until ticked.</span></div>" +
      "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'><span style='font-size:12px'>Borrow cap (generations/hour across this fedinet, 0 = none)</span><input id='fed-cap-" + esc(f.id) + "' type='number' min='0' value='" + (f.borrowCap || 0) + "' style='width:90px;min-height:44px' aria-label='Borrow cap for " + esc(f.name) + "'><button class='ghost' data-fed-cap='" + esc(f.id) + "' style='min-height:44px'>Save cap</button></div>" +
      fedConfirmHtml(f.id) + "</div>";
    // Pending joins (gateway).
    if (f.role === "gateway" && f.pendingJoins && f.pendingJoins.length) {
      h += "<div style='margin-top:12px;border-top:1px solid var(--line);padding-top:10px'><div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>KNOCKING — WAITING FOR YOUR WORD</div>";
      f.pendingJoins.forEach(function (p) {
        h += "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:8px'><span class='grow' style='font-size:12px'><b>" + esc(p.name || "Unnamed node") + "</b> · <span class='mono'>" + esc(String(p.nodeId).slice(0, 12)) + "</span></span><button class='act' data-fed-admit='" + esc(f.id) + "|" + esc(p.nodeId) + "' style='min-height:44px'>Admit</button><button class='ghost' data-fed-deny='" + esc(f.id) + "|" + esc(p.nodeId) + "' style='min-height:44px'>Deny</button></div>";
      });
      h += "</div>";
    }
    // Members.
    h += "<div style='margin-top:12px;border-top:1px solid var(--line);padding-top:10px'><div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>THE ROSTER</div>";
    if (!f.members || !f.members.length) h += "<p style='color:var(--dim);font-size:12px;margin-top:6px'>No roster here yet — " + (f.role === "gateway" ? "admit a knock and the roster begins." : "sync with the gateway and its signed roster lands.") + "</p>";
    (f.members || []).forEach(function (m) {
      var self = m.nodeId === s.nodeId;
      h += "<div style='margin-top:10px'><div class='row' style='flex-wrap:wrap;gap:8px'><span style='font-size:12px'><b>Node " + m.n + " — " + esc(m.name) + "</b>" + (m.gateway ? " · gateway" : "") + (m.founding ? " · Founding Member" : "") + " · <span class='mono' style='color:var(--dim)'>" + esc(m.short) + "</span></span></div>" +
        "<p style='color:var(--dim);font-size:12px;margin-top:2px'>" + esc(m.posture) + "</p>";
      if (!self) {
        h += "<div class='row' style='margin-top:4px;flex-wrap:wrap;gap:10px'><span class='row' style='gap:6px'>" + fedSwitch(m.lendOverride, "data-fed-lend='" + esc(f.id) + "|" + esc(m.nodeId) + "'", "Lend to " + m.name + " individually") + "<span style='font-size:12px'>Lend individually</span></span>" +
          "<span class='row' style='gap:6px'>" + fedSwitch(m.untrusted, "data-fed-untrust='" + esc(f.id) + "|" + esc(m.nodeId) + "'", "Untrust " + m.name) + "<span style='font-size:12px'>Untrusted — always wins</span></span>" +
          (f.role === "gateway" ? "<button class='ghost' data-fed-expel='" + esc(f.id) + "|" + esc(m.nodeId) + "' style='min-height:44px'>Expel</button>" : "") + "</div>";
      }
      h += "</div>";
    });
    h += "</div>";
    // Fedinet actions.
    h += "<div class='row' style='margin-top:12px;flex-wrap:wrap;gap:8px'>" +
      (f.role === "gateway"
        ? "<button class='ghost' data-fed-sync='" + esc(f.id) + "' style='min-height:44px'>Republish the roster</button>" +
          "<button class='ghost' data-fed-dissolve='" + esc(f.id) + "' style='min-height:44px;color:#f58a8a'>Dissolve this fedinet</button>"
        : "<button class='ghost' data-fed-sync='" + esc(f.id) + "' style='min-height:44px'>Sync the roster</button>" +
          (f.ownNumber ? "" : "<button class='ghost' data-fed-knock='" + esc(f.id) + "' style='min-height:44px'>Knock — ask to join</button>") +
          "<button class='ghost' data-fed-leave='" + esc(f.id) + "' style='min-height:44px'>Leave this fedinet</button>") +
      "</div></div>";
  });
  if (!(s.fedinets || []).length) {
    h += "<div class='card' style='margin-top:14px'><b>No fedinets yet</b><p style='color:var(--dim);font-size:12px;margin-top:4px'>This node keeps no fedinet. Import a gateway card to be admitted to one — " + esc(s.knownDefault.name) + " is the known one — or found your own above. Pairwise pairing on the Mesh page is untouched by any of this, either way.</p></div>";
  }
  // The audit log.
  h += "<div class='card' style='margin-top:14px'><b>The trust audit log</b><p style='color:var(--dim);font-size:12px;margin-top:4px'>Every trust-class act on this node, through any door, in sentences. Metadata only — never content.</p>";
  var rows = (s.audit || []).slice().reverse();
  if (!rows.length) h += "<p style='color:var(--dim);font-size:12px;margin-top:6px'>No trust acts yet.</p>";
  rows.forEach(function (r) {
    h += "<p style='font-size:12px;margin-top:6px'><span style='color:var(--dim)'>" + esc(new Date((r.ts || 0) * 1000).toLocaleString()) + " · " + esc(r.via === "dhd" ? "via DHD" : "at the desk") + "</span> — <b>" + esc(r.action) + "</b>" + (r.fedinet ? " · " + esc(r.fedinet) : "") + "<br><span style='color:var(--dim)'>" + esc(r.detail) + "</span></p>";
  });
  h += "</div>";
  return h;
}
function wireFederation() {
  // 0.2.0e Build 10 (#195): the wiring pass binds the error
  // card's Retry — it never fetches on its own (the Build 9
  // line here re-fired the state fetch on every render).
  var fedRetry = document.getElementById("fed-retry");
  if (fedRetry) fedRetry.onclick = function () { federationFetchState(); if (view === "Federation") render(); };
  var q = function (sel) { return mainEl.querySelector(sel); };
  var qa = function (sel) { return mainEl.querySelectorAll(sel); };
  var split = function (v) { var i = (v || "").indexOf("|"); return i < 0 ? [v, ""] : [v.slice(0, i), v.slice(i + 1)]; };
  qa("[data-fed-trust]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-fed-trust");
    var f = (federationState.fedinets || []).filter(function (x) { return x.id === id; })[0];
    if (!f) return;
    if (f.trustByDefault) { fedConfirm = null; federationAct({ act: "trust", fedinetId: id, on: false }, id); }
    else { fedConfirm = { act: "trust", fedinetId: id, label: "trust " + f.name, extra: { on: true } }; fedMsg.error = ""; render(); }
  }; });
  qa("[data-fed-mesh]").forEach(function (b) { b.onclick = function () {
    // v0.6.4e (#322): the flip goes to the Mesh settings door
    // — the same door the Mesh page's control posts to — and
    // the card then re-renders from the bridge's federation
    // state, which carries the stored Mesh state back. A
    // refusal (the identity cannot be minted, the save does
    // not take) lands on this card and the switch stays where
    // the stored state puts it.
    if (CONFIG.backend !== "live") { fedMsg.error = "Federation runs through the live bridge."; fedMsg.errorFor = "__mesh__"; render(); return; }
    var want = !(federationState && federationState.meshOn);
    fetch("/api/mesh/settings", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ enabled: want }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (j && j.ok) { fedMsg.error = ""; fedMsg.errorFor = ""; }
        else { fedMsg.error = (j && j.error) || "The bridge refused that change."; fedMsg.errorFor = "__mesh__"; }
        federationFetchState();
      })
      .catch(function () { fedMsg.error = "The bridge did not answer — is it running?"; fedMsg.errorFor = "__mesh__"; if (view === "Federation") render(); });
  }; });
  qa("[data-fed-fleet]").forEach(function (b) { b.onclick = function () {
    var flNow = (federationState && federationState.fleet) || {};
    federationAct({ act: "fleet", on: !flNow.member }, "__fleet__");
  }; });
  qa("[data-fed-tick]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-fed-tick");
    var f = (federationState.fedinets || []).filter(function (x) { return x.id === id; })[0];
    if (f) federationAct({ act: "tick", fedinetId: id, on: !f.lendToNew, badge: fedBadge() }, id);
  }; });
  qa("[data-fed-lend]").forEach(function (b) { b.onclick = function () {
    var pr = split(b.getAttribute("data-fed-lend"));
    var f = (federationState.fedinets || []).filter(function (x) { return x.id === pr[0]; })[0];
    var m = f && (f.members || []).filter(function (x) { return x.nodeId === pr[1]; })[0];
    if (m) federationAct({ act: "lend", fedinetId: pr[0], nodeId: pr[1], on: !m.lendOverride, badge: fedBadge() }, pr[0]);
  }; });
  qa("[data-fed-untrust]").forEach(function (b) { b.onclick = function () {
    var pr = split(b.getAttribute("data-fed-untrust"));
    var f = (federationState.fedinets || []).filter(function (x) { return x.id === pr[0]; })[0];
    var m = f && (f.members || []).filter(function (x) { return x.nodeId === pr[1]; })[0];
    if (m) federationAct({ act: "untrust", fedinetId: pr[0], nodeId: pr[1], on: !m.untrusted, badge: fedBadge() }, pr[0]);
  }; });
  qa("[data-fed-cap]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-fed-cap");
    var inp = q("#fed-cap-" + id);
    federationAct({ act: "cap", fedinetId: id, cap: inp ? Number(inp.value || 0) : 0, badge: fedBadge() }, id);
  }; });
  qa("[data-fed-admit]").forEach(function (b) { b.onclick = function () { var pr = split(b.getAttribute("data-fed-admit")); federationAct({ act: "admit", fedinetId: pr[0], nodeId: pr[1], badge: fedBadge() }, pr[0]); }; });
  qa("[data-fed-deny]").forEach(function (b) { b.onclick = function () { var pr = split(b.getAttribute("data-fed-deny")); federationAct({ act: "deny", fedinetId: pr[0], nodeId: pr[1], badge: fedBadge() }, pr[0]); }; });
  qa("[data-fed-expel]").forEach(function (b) { b.onclick = function () { var pr = split(b.getAttribute("data-fed-expel")); federationAct({ act: "expel", fedinetId: pr[0], nodeId: pr[1], badge: fedBadge() }, pr[0]); }; });
  qa("[data-fed-knock]").forEach(function (b) { b.onclick = function () { federationAct({ act: "knock", fedinetId: b.getAttribute("data-fed-knock") }, b.getAttribute("data-fed-knock")); }; });
  qa("[data-fed-sync]").forEach(function (b) { b.onclick = function () { federationAct({ act: "sync", fedinetId: b.getAttribute("data-fed-sync") }, b.getAttribute("data-fed-sync")); }; });
  qa("[data-fed-leave]").forEach(function (b) { b.onclick = function () { federationAct({ act: "leave", fedinetId: b.getAttribute("data-fed-leave") }, b.getAttribute("data-fed-leave")); }; });
  qa("[data-fed-dissolve]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-fed-dissolve");
    var f = (federationState.fedinets || []).filter(function (x) { return x.id === id; })[0];
    if (!f) return;
    // v0.6.4e (#321): dissolving is a crown act — the confirm
    // label is computed exactly as the bridge computes it,
    // louder when the roster holds other members.
    var others = (f.members || []).filter(function (m) { return m.nodeId !== federationState.nodeId; }).length;
    var label = "dissolve " + f.name + (others ? " and strand " + others + " member" + (others !== 1 ? "s" : "") : "");
    fedConfirm = { act: "dissolve", fedinetId: id, label: label, extra: {} }; fedMsg.error = ""; render();
  }; });
  qa("[data-fed-invitation]").forEach(function (b) { b.onclick = function () { federationAct({ act: "import-invitation", badge: fedBadge() }, "__join__"); }; });
  qa("[data-fed-badge-enroll]").forEach(function (b) { b.onclick = function () {
    if (!CONFIG.operatorBadge) { fedMsg.error = "This dashboard holds no Operator Badge yet — set it under Settings → Operator first, then enroll it here."; fedMsg.errorFor = ""; render(); return; }
    federationAct({ act: "badge-enroll", badge: CONFIG.operatorBadge }, "");
  }; });
  qa("[data-fed-badge-clear]").forEach(function (b) { b.onclick = function () { federationAct({ act: "badge-clear" }, ""); }; });
  var foundBtn = q("[data-fed-found]");
  if (foundBtn) foundBtn.onclick = function () {
    var nameEl = q("#fed-new-name");
    var name = nameEl ? String(nameEl.value || "").trim() : "";
    if (!name) { fedMsg.error = "A fedinet needs a name first."; fedMsg.errorFor = "__join__"; render(); return; }
    fedConfirm = { act: "found", fedinetId: "__found__", label: "found " + name, extra: { name: name } };
    fedMsg.error = ""; render();
  };
  var impBtn = q("[data-fed-import-btn]");
  var impFile = q("#fed-import-file");
  if (impBtn && impFile) {
    impBtn.onclick = function () { impFile.click(); };
    impFile.onchange = function () {
      var file = impFile.files && impFile.files[0];
      if (!file) return;
      var rd = new FileReader();
      rd.onload = function () {
        var buf = new Uint8Array(rd.result), bin = "";
        for (var i = 0; i < buf.length; i++) bin += String.fromCharCode(buf[i]);
        federationAct({ act: "import", cardPngB64: btoa(bin), badge: fedBadge() }, "__join__");
      };
      rd.readAsArrayBuffer(file);
    };
  }
  qa("[data-fed-confirm-cancel]").forEach(function (b) { b.onclick = function () { fedConfirm = null; render(); }; });
  qa("[data-fed-confirm-go]").forEach(function (b) { b.onclick = function () {
    if (!fedConfirm) return;
    var inp = q("#fed-confirm-in");
    var body = { act: fedConfirm.act, confirm: inp ? inp.value : "", badge: fedBadge() };
    if (fedConfirm.fedinetId !== "__found__") body.fedinetId = fedConfirm.fedinetId;
    Object.keys(fedConfirm.extra || {}).forEach(function (k) { body[k] = fedConfirm.extra[k]; });
    var forId = fedConfirm.fedinetId === "__found__" ? "__join__" : fedConfirm.fedinetId;
    fedConfirm = null;
    federationAct(body, forId);
  }; });
}
function renderMeshStatus() {
  var m = meshViewState();
  // Build 2 (#204): the status-bar law — Mesh off in Settings
  // means the Mesh row is ABSENT from the status column, not
  // greyed, not a placeholder. Off = no trace.
  if (!m.enabled) return "";
  var dot = meshDotColor(m);
  var line = "Off";
  if (m.enabled) {
    // Build .87: a warning from the payload outranks every other
    // line — the row is the persistent warning surface, so a
    // dropped or unreachable peer is what the row SAYS, in the
    // payload's own short words ("<name> dropped"). Short by
    // construction, and the span wraps rather than widening the
    // column on a phone.
    if (m.state === "warning" && m.warning) line = m.warning;
    // Build 8 (#154; ruling 2026-11-13): while this node keeps a
    // fedinet, the headline is the STATE — "Mesh Online" — not a
    // census of pairs. A warning still outranks it (the row is
    // the persistent warning surface), and the activity words
    // below still speak while work is actually moving.
    else if (m.headline) line = m.headline;
    else if (m.borrowing) line = "Borrowing now";
    else if (m.lendingNow) line = "Lending now";
    else if (m.lendEnabled && m.borrowEnabled) line = "On — " + (m.lend && m.lend.idleOnly === false ? "lending even while busy" : "lending when idle") + " · borrowing automatic";
    else if (m.lendEnabled) line = "On — " + (m.lend && m.lend.idleOnly === false ? "lending even while busy" : "lending when idle");
    else if (m.borrowEnabled) line = m.peers && m.peers.length ? "On — borrowing automatic" : "On — no peers paired yet";
    else line = "On — borrowing and lending both off";
  }
  var h = "<div class='stat'><span class='dot" + (m.enabled ? " pdot" : "") + "' style='background:" + dot + "'></span><span style='color:var(--dim)'>Apex | Mesh</span><br>" +
    "<span class='v mono' style='font-weight:700;overflow-wrap:anywhere'>" + esc(line) + "</span>";
  h += "<details id='mesh-details'" + (meshDetailsOpen ? " open" : "") + " style='margin-top:2px'><summary style='color:var(--dim);font-size:12px;cursor:pointer'>Mesh details</summary><div style='margin-top:6px'>";
  if (!m.enabled) {
    h += "<div style='color:var(--dim);font-size:12px'>Mesh is off. Turn it on from the Mesh page to pair this machine with your others.</div>";
  } else {
    h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em'>THIS MACHINE</div><div style='font-size:12px'>" + esc(m.nodeName || "Unnamed") + (m.nodeId ? " · <span class='mono'>" + esc(String(m.nodeId).slice(0, 12)) + "</span>" : "") + "</div>";
    if (m.peers && m.peers.length) {
      h += "<div style='color:var(--dim);font-size:11px;letter-spacing:.06em;margin-top:8px'>PAIRED MACHINES</div>";
      m.peers.forEach(function (p) {
        var reach = (p.status && p.status !== "unknown") ? p.status : p.reachable === true ? "reachable" : p.reachable === false ? "unreachable" : "not checked yet";
        if (p.status === "unreachable" || (p.status == null && p.reachable === false)) reach = meshUnreachableText(p);
        h += "<div style='font-size:12px;margin-top:2px'>" + esc(p.name) + " <span style='color:var(--dim)'>· " + reach + " · balance " + meshFmtSec(p.balanceSec) + "</span></div>";
      });
    } else {
      h += "<div style='color:var(--dim);font-size:12px;margin-top:4px'>No machines paired yet — pairing is a card exchange, on the Mesh page.</div>";
    }
  }
  h += "</div></details></div>";
  return h;
}
function wireMeshStatus() {
  var det = document.getElementById("mesh-details");
  if (det) det.addEventListener("toggle", function () { meshDetailsOpen = det.open; });
}
function meshSave(patch) {
  if (patch.borrowEnabled) { meshPrivacyAcked = true; try { localStorage.setItem("navi.meshPrivacyAck", "1"); } catch (e) {} }
  if (CONFIG.backend !== "live") {
    var m = CONFIG.mesh = CONFIG.mesh || { enabled: false, role: "both", nodeName: "", publicEndpoint: "", listenPorts: [8765, 8443, 2475], borrow: { enabled: true, graceMs: 2000, peers: [] }, lend: { enabled: false, maxSharePercent: 25, maxGenerationsPerHour: 20, perPeerPerHour: 10, idleOnly: true, idleAfterSeconds: 60 }, port: 8765, nodeId: "", ledger: true, ledgerInBackup: false };
    m.borrow = m.borrow || { enabled: true, graceMs: 2000, peers: [] };
    m.lend = m.lend || { enabled: false, maxSharePercent: 25, maxGenerationsPerHour: 20, perPeerPerHour: 10, idleOnly: true, idleAfterSeconds: 60 };
    if (patch.enabled != null) m.enabled = patch.enabled;
    if (patch.nodeName != null) m.nodeName = patch.nodeName;
    if (patch.publicEndpoint != null) m.publicEndpoint = patch.publicEndpoint;
    if (patch.borrowEnabled != null) m.borrow.enabled = patch.borrowEnabled;
    if (patch.graceMs != null) m.borrow.graceMs = patch.graceMs;
    if (patch.lendEnabled != null) m.lend.enabled = patch.lendEnabled;
    if (patch.maxSharePercent != null) m.lend.maxSharePercent = patch.maxSharePercent;
    if (patch.maxGenerationsPerHour != null) m.lend.maxGenerationsPerHour = patch.maxGenerationsPerHour;
    if (patch.perPeerPerHour != null) m.lend.perPeerPerHour = patch.perPeerPerHour;
    if (patch.idleOnly != null) m.lend.idleOnly = patch.idleOnly;
    if (patch.ledgerInBackup != null) m.ledgerInBackup = patch.ledgerInBackup;
    saveCfg(); meshMsg = "Saved (mock mode — simulated)."; render(); return;
  }
  fetch("/api/mesh/settings", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(patch) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (j && j.ok) { meshState = j.mesh; meshMsg = "Saved."; }
      else meshMsg = (j && j.error) || "The bridge refused that change.";
      render();
    })
    .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
}
function meshSwitchRow(id, on, label, ariaLabel, locked) {
  // Build 6 (#86): a locked switch is the same switch, dimmed
  // and disabled — no new visual language. It is used for the
  // Lend switch on the DHD-door view while lending is off: the
  // bridge refuses that turn-on in person only, so the control
  // says so instead of dressing as a working toggle.
  var lockAttrs = locked ? " disabled aria-disabled='true'" : "";
  var lockStyle = locked ? " style='opacity:.45;cursor:not-allowed'" : "";
  return "<div style='display:flex;align-items:center;gap:8px;margin-top:8px'><button class='switch' id='" + id + "' role='switch' aria-checked='" + (on ? "true" : "false") + "' aria-label='" + esc(ariaLabel) + "'" + lockAttrs + lockStyle + "><span class='knob'></span></button><span>" + esc(label) + "</span></div>";
}
function meshCapLine(caps) {
  // One node's capability line for the roster: hardware class,
  // GPU (or CPU), RAM, model, speed class, caps — the peer's own
  // self-report from its card / latest probe. Absent fields are
  // skipped, never invented. Addresses are NEVER part of this
  // line: a peer's addresses are plumbing, not display.
  if (!caps) return "";
  var bits = [];
  if (caps.hardwareClass) bits.push(caps.hardwareClass);
  if (caps.gpu) bits.push(caps.gpu);
  else if (caps.cpu) bits.push(caps.cpu);
  if (caps.ramGB) bits.push((Math.round(caps.ramGB * 10) / 10) + " GiB RAM");
  bits.push("model " + (caps.model || "navi"));
  if (caps.speedClass) bits.push(caps.speedClass);
  if (caps.caps) bits.push("lends at most " + caps.caps.maxSharePercent + "% of each hour, " + caps.caps.maxGenerationsPerHour + " generations an hour");
  // Build 7 (Fix F2): the posture closes the line, stated the
  // same way on the machine's own row and every peer row.
  bits.push(caps.idleOnly === false ? "lends even while busy" : "lends when idle");
  return bits.join(" · ");
}
function meshHealthHtml() {
  // The Mesh health check card (0.2.0e Build 1, #143): the two
  // decisive reads of the field diagnosis, run by the product
  // itself — this machine's disk certificate against the one
  // its door presents, every peer's pinned fingerprint against
  // the one it presents, the bridge census, and the door's
  // owner. Operator-triggered; results render here, the
  // fingerprints side by side, every verdict in words. The
  // check changes nothing — it only reports.
  var m = meshViewState();
  var h = "<div class='card'><h3>Mesh health check</h3>";
  if (CONFIG.backend !== "live") {
    h += "<p style='color:var(--dim)'>The health check runs through the live bridge — switch the backend to Live in Settings and it will be here.</p></div>";
    return h;
  }
  if (!m.enabled) {
    h += "<p style='color:var(--dim)'>Mesh is off — the check reads the live Mesh door, so turn Mesh on above first.</p></div>";
    return h;
  }
  h += "<p style='color:var(--dim)'>When a pair goes dark and the roster's word is not enough: this check compares the certificate on this machine's disk with the one its Mesh door actually presents, holds every paired machine's pinned fingerprint up against the certificate it presents, counts the bridges running here, and names the process that owns the Mesh door. The check itself changes nothing — and where a record can be healed in place, the heal is offered right on the card that names the problem.</p>";
  h += "<p><button type='button' class='act' id='mesh-health-run' aria-label='Run the Mesh health check' style='min-height:44px'" + (meshHealthBusy ? " disabled" : "") + ">" + (meshHealthBusy ? "Checking…" : "Run health check") + "</button></p>";
  if (meshHealthMsg) h += "<p style='font-size:12px'>" + esc(meshHealthMsg) + "</p>";
  var hh = meshHealth;
  if (hh) {
    var fp = function (v) { return v ? "<span class='mono' style='overflow-wrap:anywhere'>" + esc(v) + "</span>" : "<span style='color:var(--dim)'>—</span>"; };
    var verdictWord = function (v) { return v === "match" ? "match" : v === "mismatch" ? "MISMATCH" : v === "own-door" ? "your own door" : v === "not-listening" ? "not answering" : v === "unauthorized" ? "signature refused" : v === "unreachable" ? "unreachable" : v === "off" ? "off" : v; };
    h += "<p style='font-size:13px'><b>" + esc(hh.summary || "") + "</b></p>";
    var sf = hh.self || {};
    h += "<div style='border:1px solid var(--line);border-radius:6px;padding:8px;margin-top:6px'>" +
      "<div style='font-size:12px;letter-spacing:.06em;color:var(--dim)'>THIS MACHINE — CERTIFICATE</div>" +
      "<div style='font-size:12px;margin-top:4px'>On disk: " + fp(sf.diskFingerprint) + "</div>" +
      "<div style='font-size:12px;margin-top:2px'>Presented: " + fp(sf.presentedFingerprint) + "</div>" +
      "<div style='font-size:12px;margin-top:4px'><b>" + esc(verdictWord(sf.verdict)) + "</b> — " + esc(sf.verdictText || "") + "</div></div>";
    var pr = hh.processes || {};
    h += "<div style='border:1px solid var(--line);border-radius:6px;padding:8px;margin-top:6px'>" +
      "<div style='font-size:12px;letter-spacing:.06em;color:var(--dim)'>PROCESSES — WHO SERVES THE DOOR</div>" +
      "<div style='font-size:12px;margin-top:4px'>" + esc(pr.verdictText || "") + "</div></div>";
    (hh.peers || []).forEach(function (pp) {
      h += "<div style='border:1px solid var(--line);border-radius:6px;padding:8px;margin-top:6px'>" +
        "<div style='display:flex;align-items:center;gap:8px;flex-wrap:wrap'><b>" + esc(pp.name) + "</b><span style='color:var(--dim);font-size:12px'>" + esc(verdictWord(pp.verdict)) + (pp.stage && pp.verdict !== "match" && pp.verdict !== "own-door" ? " · failed at the " + esc(pp.stage) + " stage" : "") + "</span></div>" +
        "<div style='font-size:12px;margin-top:4px'>Pinned by its card: " + fp(pp.pinnedFingerprint) + "</div>" +
        "<div style='font-size:12px;margin-top:2px'>It presents: " + fp(pp.presentedFingerprint) + "</div>" +
        "<div style='font-size:12px;margin-top:4px'>" + esc(pp.verdictText || "") + "</div>" +
        (pp.repair ? "<div style='font-size:12px;margin-top:6px;border-top:1px dashed var(--line);padding-top:6px'>" + esc(pp.repair.sentence) + "</div><div style='display:flex;gap:8px;flex-wrap:wrap;margin-top:6px'>" + (pp.repair.kind === "card-refresh" ? "<button type='button' class='act' data-mesh-repair-clear='" + esc(pp.nodeId) + "' aria-label='Refresh " + esc(pp.name) + "'s record from its gateway card' style='min-height:44px'>Refresh from the gateway card</button>" : "<button type='button' class='act' data-mesh-repair-import='" + esc(pp.nodeId) + "' aria-label='Import a fresh card for " + esc(pp.name) + "' style='min-height:44px'>Import a fresh card</button><button type='button' class='ghost' data-mesh-repair-clear='" + esc(pp.nodeId) + "' aria-label='Clear the dead addresses from " + esc(pp.name) + "'s record' style='min-height:44px'>Clear the dead addresses</button>") + "</div>" : "") + "</div>";
    });
    (hh.relay || []).forEach(function (rr) {
      h += "<div style='border:1px solid var(--line);border-radius:6px;padding:8px;margin-top:6px'>" +
        "<div style='font-size:12px;letter-spacing:.06em;color:var(--dim)'>RELAY — TO " + esc(String(rr.toName || "").toUpperCase()) + " VIA " + esc(String(rr.viaName || "").toUpperCase()) + "</div>" +
        "<div style='font-size:12px;margin-top:4px'>" + esc(rr.verdictText || "") + "</div></div>";
    });
  }
  h += "</div>";
  return h;
}
function meshRunHealth() {
  if (meshHealthBusy) return;
  meshHealthBusy = true; meshHealthMsg = ""; render();
  fetch("/api/mesh/health", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      meshHealthBusy = false;
      if (j && j.ok && j.health) { meshHealth = j.health; meshHealthMsg = ""; }
      else meshHealthMsg = (j && j.error) || "The bridge refused the check.";
      render();
    })
    .catch(function () { meshHealthBusy = false; meshHealthMsg = "The bridge did not answer — is it running?"; render(); });
}
function meshTunnelHtml(p, m) {
  // Build 4 (#178): one paired machine's managed backup route —
  // the SSH tunnel this bridge keeps open itself — plus the
  // lender-side act for that machine's tunnel key. The route
  // fields always render (the switch says whether the road is
  // live); authorizing is offered only off the DHD view, the
  // same in-person doctrine as arming lending, and the bridge
  // refuses it there regardless.
  var t = p.tunnel || { enabled: false, sshHost: "", sshPort: 227, sshUser: "", localPort: null };
  var sug = (p.tunnelSuggestion && p.tunnelSuggestion.sshHost) || "";
  var armed = !t.enabled && !!meshTunnelArmed[p.nodeId];
  var inp = "min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px";
  var h = "<div style='margin-top:8px;border-top:1px solid var(--line);padding-top:8px'>" +
    "<div style='display:flex;align-items:center;gap:8px;flex-wrap:wrap'><button class='switch' role='switch' aria-checked='" + (t.enabled || armed ? "true" : "false") + "' data-mesh-tunnel='" + esc(p.nodeId) + "' aria-label='Backup route to " + esc(p.name) + " — a managed SSH tunnel this bridge keeps open itself'><span class='knob'></span></button><span style='font-size:12px'>Backup route — managed SSH tunnel</span></div>" +
    "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:6px'>" +
    "<label style='font-size:12px'>SSH host <input type='text' data-mesh-tun-host='" + esc(p.nodeId) + "' value='" + esc(t.sshHost || sug) + "' placeholder='e.g. ssh.example.org' aria-label='SSH host for the backup route to " + esc(p.name) + "' style='" + inp + "'></label>" +
    "<label style='font-size:12px'>SSH port <input type='number' data-mesh-tun-port='" + esc(p.nodeId) + "' value='" + (t.sshPort || 227) + "' min='1' max='65535' aria-label='SSH port for the backup route to " + esc(p.name) + "' style='width:84px;" + inp + "'></label>" +
    "<label style='font-size:12px'>SSH user <input type='text' data-mesh-tun-user='" + esc(p.nodeId) + "' value='" + esc(t.sshUser || "") + "' aria-label='SSH user for the backup route to " + esc(p.name) + "' style='" + inp + "'></label>" +
    "<button type='button' class='ghost' data-mesh-tunnel-save='" + esc(p.nodeId) + "' aria-label='Save the backup route to " + esc(p.name) + "' style='min-height:44px'>Save route</button></div>";
  if (!t.enabled && !armed) {
    h += "<p style='color:var(--dim);font-size:12px;margin-top:6px'>The walked path, three presses: flip the switch to arm the route" + (sug ? " — the SSH host is already filled in from the public endpoint " + esc(p.name) + " declared on its card" : "") + ", fill whichever field is still blank, then Save route, and the bridge keeps the tunnel open by itself. One step no button here can take: " + esc(p.name) + " must authorize this machine's tunnel key on its own dashboard — a crown act there, spoken by name (and, through its DHD door, under its Operator Badge) — before the tunnel can open. It is never the borrower's to give.</p>";
  }
  if (armed) {
    h += "<p style='font-size:12px;margin-top:6px;color:var(--accent)'>The route is armed, not on: it still needs that machine's SSH host and user. Fill in whichever is blank above and press Save route — the bridge keeps the tunnel from there.</p>";
  }
  if (t.enabled) {
    h += p.tunnelSentence
      ? "<p style='font-size:12px;margin-top:6px'>" + esc(p.tunnelSentence) + "</p>"
      : "<p style='color:var(--dim);font-size:12px;margin-top:6px'>The bridge keeps this tunnel open itself — no command to run, nothing to keep alive by hand. Direct addresses are always tried first; the tunnel is the road of last resort.</p>";
  }
  if (p.hasTunnelKey) {
    // Build 8: authorizing a tunnel key is a CROWN ACT (the
    // 2026-11-13 ruling amended Build 4's in-person-only door):
    // it runs at the desk and through DHD alike, under a
    // named-action confirm — and through DHD, the Operator
    // Badge step-up rides with it. Revoking stays open through
    // every door. The confirm strip opens ON THIS CARD and a
    // refusal lands on it too — never a silent snap-back.
    h += "<div style='display:flex;align-items:center;gap:8px;margin-top:8px;flex-wrap:wrap'><button type='button' class='ghost' data-mesh-tunnel-auth='" + esc(p.nodeId) + "' data-auth-action='" + (p.tunnelKeyAuthorized ? "revoke" : "authorize") + "' aria-label='" + (p.tunnelKeyAuthorized ? "Revoke " : "Authorize ") + esc(p.name) + "'s tunnel key on this machine' style='min-height:44px'>" + (p.tunnelKeyAuthorized ? "Revoke tunnel key" : "Authorize tunnel key") + "</button><span style='font-size:12px;color:var(--dim)'>" + (p.tunnelKeyAuthorized ? "This machine lets " + esc(p.name) + "'s tunnel key forward to its Mesh door — nothing else. Revoking works from anywhere." : "Let " + esc(p.name) + " keep a backup tunnel to this machine's Mesh door — forwarding only, to this door only. A crown act: it asks you to speak its name" + (m.viaDhd ? ", and your Operator Badge rides as the step-up through this door" : "") + ".") + "</span></div>";
    if (meshTunnelConfirmFor === p.nodeId && !p.tunnelKeyAuthorized) {
      h += "<div style='margin-top:8px;border:1px solid #f5b544;border-radius:8px;padding:10px'><b style='font-size:12px'>A crown act — speak its name.</b><p style='color:var(--dim);font-size:12px;margin-top:4px'>Type the words <span class='mono' style='color:var(--ink)'>authorize tunnel key for " + esc(p.name) + "</span> to confirm. The act lands in the Federation audit log.</p><div class='row' style='margin-top:6px'><input class='grow' id='mesh-tunnel-confirm-in' type='text' placeholder='authorize tunnel key for " + esc(p.name) + "' aria-label='Type the confirm words'><button class='act' data-mesh-tunnel-confirm='" + esc(p.nodeId) + "' style='min-height:44px'>Confirm</button><button class='ghost' data-mesh-tunnel-cancel='1' style='min-height:44px'>Cancel</button></div></div>";
    }
    if (meshTunnelAuthMsg && meshTunnelAuthMsg.nodeId === p.nodeId) {
      h += "<p style='font-size:12px;margin-top:6px;color:" + (meshTunnelAuthMsg.ok ? "var(--accent)" : "#f58a8a") + "'>" + esc(meshTunnelAuthMsg.text) + "</p>";
    }
  } else {
    h += "<p style='color:var(--dim);font-size:12px;margin-top:8px'>" + esc(p.name) + "'s card carries no tunnel key — it predates the managed backup route. Re-share its card from its Mesh page and pair it again, and the key arrives with the card.</p>";
  }
  h += "</div>";
  return h;
}
function meshRelayHtml(p, m) {
  // 0.2.0e Build 7 (#154): one paired machine's relay consent —
  // the third road. When the direct road to a lender is dark, a
  // paired third machine can carry the sealed envelope; the
  // carrier sees envelope metadata only, never the words. Three
  // consents meet for every carriage, and each lives on its own
  // machine: this machine's two ticks render here on the
  // carrier's card (may carry mine out; may carry to me), and
  // the carrier's own tick renders here too, on the ASKER's
  // card (I will carry theirs) — the section reads the stored
  // lists from the payload, so the switches, the saved config,
  // and a reload all show the same truth. Nothing renders at
  // all until a third machine is paired: a relay needs three.
  var peers = (m && m.peers) || [];
  if (peers.length < 2) return "";
  var others = peers.filter(function (q) { return q.nodeId !== p.nodeId; });
  if (!others.length) return "";
  var row = function (kind, list, q, label) {
    var on = (list || []).indexOf(q.nodeId) >= 0;
    return "<div style='display:flex;align-items:center;gap:8px;margin-top:6px;flex-wrap:wrap'><button class='switch' role='switch' aria-checked='" + (on ? "true" : "false") + "' data-mesh-relay='" + kind + "' data-peer='" + esc(p.nodeId) + "' data-other='" + esc(q.nodeId) + "' aria-label='" + esc(label) + "'><span class='knob'></span></button><span style='font-size:12px'>" + esc(q.name) + "</span></div>";
  };
  var h = "<div style='margin-top:8px;border-top:1px solid var(--line);padding-top:8px'>" +
    "<div style='font-size:12px'><b>Relay</b> — sealed carriage when a direct road is dark</div>" +
    "<p style='color:var(--dim);font-size:12px;margin-top:4px'>If this machine cannot reach another paired machine directly, a third machine can carry the envelope — sealed on both hops; the carrier sees who asks, who is asked, and how many bytes, never the words. A carriage needs all three machines' consent for the pair, each ticked on its own machine; unticking any one stops the next carriage.</p>";
  h += "<div style='font-size:12px;margin-top:8px'>" + esc(p.name) + " may carry <b>my</b> envelopes to:</div>";
  others.forEach(function (q) { h += row("carryTo", p.relayCarryTo, q, p.name + " may carry my envelopes to " + q.name); });
  h += "<div style='font-size:12px;margin-top:8px'>" + esc(p.name) + " may carry envelopes <b>to me</b> from:</div>";
  others.forEach(function (q) { h += row("acceptFrom", p.relayAcceptFrom, q, p.name + " may carry envelopes to me from " + q.name); });
  h += "<div style='font-size:12px;margin-top:8px'><b>I</b> will carry " + esc(p.name) + "'s envelopes to:</div>";
  others.forEach(function (q) { h += row("willCarryTo", p.relayWillCarryTo, q, "I will carry " + p.name + "'s envelopes to " + q.name); });
  if (m.viaDhd) h += "<p style='color:var(--dim);font-size:12px;margin-top:6px'>You are viewing through the DHD door: these ticks can be unticked from here, but consent itself is given at this machine, in person — a tick flipped on here will be refused, and the refusal will say so.</p>";
  h += "</div>";
  return h;
}
function meshPostureHtml(m) {
  // The posture sentence under the page header (0.2.0e Build
  // 10, #196): the same priority the status column's Mesh row
  // speaks — a warning first, then the federation headline
  // ("Mesh Online" — the state, not a census of pairs), then
  // activity, then the plain on/off posture.
  var line;
  if (!m.enabled) line = "The Mesh is off — this machine neither lends nor borrows. Turn it on at your door and it can pair with your other machines.";
  else if (m.state === "warning" && m.warning) line = m.warning;
  else if (m.headline) line = m.headline + " — this node keeps a federation, and the door below is its own.";
  else if (m.borrowing) line = "Borrowing now — a paired machine is answering for this one.";
  else if (m.lendingNow) line = "Lending now — this machine is answering for a paired machine.";
  else if (m.lendEnabled && m.borrowEnabled) line = "On — " + (m.lend && m.lend.idleOnly === false ? "lending even while busy" : "lending when idle") + " · borrowing automatic.";
  else if (m.lendEnabled) line = "On — " + (m.lend && m.lend.idleOnly === false ? "lending even while busy" : "lending when idle") + ".";
  else if (m.borrowEnabled) line = "On — borrowing automatic.";
  else line = "On — borrowing and lending both off.";
  return "<p style='color:var(--dim)'>" + esc(line) + "</p>";
}
function meshPageHtml() {
  // 0.2.0e Build 10 (#196): the Mesh page as one calm column,
  // modeled on the Federation page — the posture sentence
  // under the header, then the sections as cards: Your door,
  // Lending & borrowing, one card per peer, Roads, and Health
  // at the foot. Every control the single Build 9 card
  // carried keeps its id, its handler (wireMeshCard is
  // unchanged) and its words; only the presentation moved,
  // and the roster's rows folded into the door and peer cards.
  var m = meshViewState();
  var h = "<h3>Apex | Mesh</h3>";
  h += meshPostureHtml(m);
  // Build 6 (#86) seat, kept: save confirmations and refusals
  // render at the head of the page, directly under the
  // posture — where a person mid-click sees them.
  if (meshMsg) h += "<p style='font-size:12px'>" + esc(meshMsg) + "</p>";
  h += meshDoorHtml(m);
  h += meshLendBorrowHtml(m);
  h += meshPeersHtml(m);
  h += meshRoadsHtml(m);
  h += meshHealthHtml();
  return h;
}
function meshDoorHtml(m) {
  var h = "<div class='card'><h3 style='margin-top:0'>Your door</h3>" +
    "<p style='color:var(--dim)'>Pooled inference across your machines: a stronger machine answers whole chat requests for a weaker one, and the reply says which machine wrote it. Pairing is a card you carry by hand — a paired machine is a trusted machine.</p>";
  h += meshSwitchRow("mesh-master", m.enabled, "Apex | Mesh", "Apex | Mesh master switch");
  if (m.enabled) {

    h += "<div class='row' style='margin-top:10px;flex-wrap:wrap;gap:6px'><label style='font-size:12px'>This machine's name <input id='mesh-name' type='text' value='" + esc(m.nodeName || "") + "' placeholder='e.g. Home server' aria-label='This machine’s Mesh name' style='min-height:44px'></label>" +
      (m.nodeId ? "<span class='mono' style='font-size:11px;color:var(--dim)'>node " + esc(String(m.nodeId).slice(0, 12)) + "</span>" : "") + "</div>";
    // 0.2.0e Build 4 (#154): the operator-declared public
    // endpoint — typed here, never harvested, carried on this
    // machine's card as a host:port entry beside the LAN
    // addresses, so a paired machine reaches this one at home
    // and away off a single record: no tunnel, no hand edits.
    // It is the one address Apex ever shows — the operator
    // typed it for publication. It rides the card's Save
    // button, like the name and the caps.
    h += "<div class='row' style='margin-top:10px;flex-wrap:wrap;gap:6px'><label style='font-size:12px'>Public endpoint (optional) <input id='mesh-endpoint' type='text' value='" + esc(m.publicEndpoint || "") + "' placeholder='e.g. mesh.example.org:8443' aria-label='This machine’s public Mesh endpoint — a host and port, optional' style='min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'></label></div>" +
      "<p style='color:var(--dim);font-size:12px'>If this machine's Mesh door can be reached from beyond its own networks — a port you forwarded to it — declare that door here as host:port. It rides your card next to the LAN addresses, and changing it means re-sharing your card, the same as a new name. Leave it empty when this machine is only reachable on its own networks: the card then carries its LAN addresses and nothing else.</p>";
    // Build 4 (#154): the door's numbers, where the door's
    // status already lives — one door, one certificate, every
    // port; a peer record entry may name any of them.
    var doorPorts = (m.listenPorts && m.listenPorts.length) ? m.listenPorts : [8765, 8443, 2475];
    h += "<p style='color:var(--dim);font-size:12px'>This machine's Mesh door listens on port" + (doorPorts.length > 1 ? "s " : " ") + doorPorts.join(" · ") + " — one door, one certificate, every number.</p>";
    // Build 4 (#178): this machine's tunnel public key — the
    // copyable line a paired machine's operator never has to
    // handle by hand, because it rides the pairing card; shown
    // here so the operator can see exactly what was shared.
    if (m.tunnelPublicKey) h += "<div class='row' style='margin-top:10px;flex-wrap:wrap;gap:6px'><label style='font-size:12px'>This machine's tunnel key <input type='text' readonly value='" + esc(m.tunnelPublicKey) + "' aria-label='This machine’s Mesh tunnel public key — it rides the pairing card, and a paired machine authorizes it once, at that machine, in person' class='mono' style='min-height:44px;min-width:min(520px,80vw);background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'></label></div>" +
      "<p style='color:var(--dim);font-size:12px'>The public half of the key this machine's managed backup tunnels use. It rides your pairing card; a paired machine authorizes it once, at home, in person — then this machine keeps its backup roads open by itself. The private half never leaves this machine and never rides a backup.</p>";
    var selfStatus = m.lendingNow ? "lending now" : m.borrowing ? "borrowing now" : "idle";
    var selfDot = (m.lendingNow || m.borrowing) ? "#f5b544" : "#34d399";
    h += "<div style='border-top:1px solid var(--line);margin-top:12px;padding-top:8px'><div style='display:flex;align-items:center;gap:8px;flex-wrap:wrap'><span class='dot' style='background:" + selfDot + "'></span><b>" + esc(m.nodeName || "This machine") + "</b><span style='color:var(--dim);font-size:12px'>this machine · " + selfStatus + "</span></div>" +
      (meshCapLine(m.capabilities) ? "<div style='color:var(--dim);font-size:12px;margin-top:4px'>" + esc(meshCapLine(m.capabilities)) + "</div>" : "") + "</div>";
h += "<div style='margin-top:10px;display:flex;gap:8px;flex-wrap:wrap'><button type='button' class='act' id='mesh-card-png' aria-label='Download my Mesh card as a PNG picture' style='min-height:44px'>Download PNG card</button></div>" +
      "<p style='color:var(--dim);font-size:12px'>Your card is a picture: a face you can read — name, node, the offer — with the whole card riding inside the picture, the way chips travel. Pairing is the picture, only — there is no text card to copy or paste anymore. Send the file itself; for a chat app, put it inside a .zip first, because chat apps recompress pictures and strip the data out.</p>";
    // Build 7 (spec section 13): the PNG import — a visible drop
    // target plus an upload control, both feeding the one
    // importer (wireMeshCard). Drag-and-drop is the desktop
    // path; the upload button is the path that works everywhere,
    // phones included: a 44px target, and the zone wraps its
    // words instead of widening the page at 360-430px.
    if (!m.certNoticeSeen) h += "<p style='color:var(--am);font-size:12px;margin-top:10px'>First pairing — one thing worth knowing before any trust is pinned: a Mesh door's certificate is <b>self-signed</b>. No authority vouches for it, and none should: the card you were handed <i>is</i> the trust. Importing it pins that machine's certificate fingerprint on this machine, and from then on this machine accepts that door only when it presents exactly that certificate — a different one is named, never quietly trusted.</p>";
    h += "<div id='mesh-png-zone' style='margin-top:12px;border:1.5px dashed " + (meshPngDrag ? "var(--accent)" : "var(--line)") + ";border-radius:6px;padding:12px;text-align:center'>" +
      "<p style='font-size:12px;margin:0'>Pair by picture — drop the other machine's PNG card here</p>" +
      "<p style='margin:8px 0 0'><label for='mesh-png-file' class='act' style='display:inline-flex;align-items:center;min-height:44px;padding:0 14px;cursor:pointer'>Upload a PNG card</label>" +
      "<input type='file' id='mesh-png-file' accept='image/png,.png' aria-label='Upload a Mesh PNG card file' style='display:none'></p>" +
      "<p style='color:var(--dim);font-size:11px;margin:8px 0 0'>The picture must arrive as the original file — inside a .zip for chat apps, which recompress pictures and strip the card data out.</p>" +
      (meshPngMsg ? "<p style='font-size:12px;margin:8px 0 0'>" + esc(meshPngMsg) + "</p>" : "") +
      "</div>";
    h += "<div style='display:flex;align-items:center;gap:8px;margin-top:10px'><input type='checkbox' id='mesh-ledger-backup' aria-label='Include the Mesh ledger in backups'" + (m.ledgerInBackup ? " checked" : "") + " style='width:20px;height:20px'><label for='mesh-ledger-backup' style='font-size:12px'>Include the Mesh ledger in backups (off by default — the ledger is metadata, but whose machines talked is yours)</label></div>";
    h += "<p style='margin-top:10px'><button type='button' class='act' id='mesh-save' aria-label='Save Mesh name, public endpoint, grace, lending caps, and ledger choice' style='min-height:44px'>Save name, grace &amp; caps</button></p>";
  }
  h += "</div>";
  return h;
}
function meshLendBorrowHtml(m) {
  if (!m.enabled) return "";
  var lend = m.lend || {};
  var h = "<div class='card'><h3 style='margin-top:0'>Lending &amp; borrowing</h3>";
h += "<h4 style='margin:14px 0 0'>Borrowing</h4>";
    h += meshSwitchRow("mesh-borrow", m.borrowEnabled, "Borrow from my paired machines (automatic)", "Borrow from paired machines — automatic when a lender is reachable, with this off switch");
    if (m.borrowEnabled && !meshPrivacyAcked) {
      h += "<p style='font-size:12px;border:1px solid var(--line);border-radius:6px;padding:8px;margin-top:8px'>Borrowing sends this chat's full context — including your Soul and Memory pages and recent conversation — to the lending machine, whose model reads it to write the reply. It is encrypted on the way and the lender keeps none of it, but that machine's operator owns the machine it runs on. Only borrow from machines you trust with the conversation.</p>";
    }
    h += "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:6px'><label style='font-size:12px'>Grace when a lender is busy <input id='mesh-grace' type='number' min='0' max='30000' step='100' value='" + (m.graceMs != null ? m.graceMs : 2000) + "' aria-label='Borrow grace in milliseconds' style='width:96px;min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'> ms</label></div>" +
      "<p style='color:var(--dim);font-size:12px'>If a lender is busy, this machine waits up to the grace for it to free up, then answers locally. A freed lender serves the next request — never a half-written reply.</p>";
h += "<h4 style='margin:14px 0 0'>Lending</h4>";
    // Build 6 (#86): on the DHD-door view (viaDhd) with lending
    // off, the switch renders LOCKED — the bridge refuses a
    // remote turn-on, so the control shows the lock and the
    // reason sits directly under it. With lending on, the
    // switch stays live (off works from anywhere) with the
    // turn-back-on note. Locally (viaDhd false) this section
    // renders exactly as it always has.
    var lendLocked = !!m.viaDhd && !m.lendEnabled;
    // Build 7 (Fix F2): the switch's own words follow the
    // idle-only posture — with the override off, "automatic when
    // idle" would be a false label on a live control.
    var lendIdleOnly = lend.idleOnly !== false;
    h += meshSwitchRow("mesh-lend", m.lendEnabled, lendIdleOnly ? "Lend this machine (automatic when idle)" : "Lend this machine (automatic, even while busy)", lendIdleOnly ? "Lend this machine — automatic when idle, inside the caps below" : "Lend this machine — automatic, even while busy, inside the caps below", lendLocked);
    if (lendLocked) h += "<p style='color:var(--dim);font-size:12px'>Lending can only be turned on at this machine, in person — and this view is through the DHD door, so the switch is locked here. Turning it off works from anywhere.</p>";
    else if (m.viaDhd && m.lendEnabled) h += "<p style='color:var(--dim);font-size:12px'>You are viewing through the DHD door: you can turn lending off from here, but turning it back on will require being at this machine, in person.</p>";
    h += "<p style='color:var(--dim);font-size:12px'>Caps: at most " + (lend.maxSharePercent != null ? lend.maxSharePercent : 25) + "% of each hour · " + (lend.maxGenerationsPerHour != null ? lend.maxGenerationsPerHour : 20) + " borrowed generations an hour · " + (lend.perPeerPerHour != null ? lend.perPeerPerHour : 10) + " per machine" + (lend.idleOnly !== false ? " · idle only (" + (lend.idleAfterSeconds != null ? lend.idleAfterSeconds : 60) + "s quiet)" : "") + ". Your own work always takes the next slot.</p>" +
      // Build 7 (Fix F): the caps are operator settings, set
      // right here in the Lending section — inline with the Lend
      // switch and the Caps line above, never behind a panel or
      // in Settings. Same input family as the grace field; the
      // row wraps on a phone and every input is a 44px target.
      // Saving rides the card's Save button (wireMeshCard), and
      // the bridge applies the new caps to the next admission
      // and the next card it shows — no restart.
      "<div class='row' style='margin-top:8px;flex-wrap:wrap;gap:6px'>" +
      "<label style='font-size:12px'>Share of each hour <input id='mesh-cap-share' type='number' min='1' max='100' step='1' value='" + (lend.maxSharePercent != null ? lend.maxSharePercent : 25) + "' aria-label='Lending cap — share of each hour, percent' style='width:84px;min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'> %</label>" +
      "<label style='font-size:12px'>Generations per hour <input id='mesh-cap-gen' type='number' min='1' max='10000' step='1' value='" + (lend.maxGenerationsPerHour != null ? lend.maxGenerationsPerHour : 20) + "' aria-label='Lending cap — borrowed generations per hour' style='width:96px;min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'></label>" +
      "<label style='font-size:12px'>Per machine per hour <input id='mesh-cap-peer' type='number' min='1' max='10000' step='1' value='" + (lend.perPeerPerHour != null ? lend.perPeerPerHour : 10) + "' aria-label='Lending cap — borrowed generations per machine per hour' style='width:96px;min-height:44px;background:var(--panel2);border:1px solid var(--line);color:var(--text);border-radius:4px;padding:8px'></label></div>" +
      "<p style='color:var(--dim);font-size:12px'>Set these for the machine this is — a strong server can take far more than the shipped defaults (25%, 20 an hour, 10 per machine). Your card advertises what you set here, so a paired machine sees the real offer.</p>" +
      "<p style='color:var(--dim);font-size:12px'>Lending can only be turned ON here, on this machine's own dashboard — never through the DHD door. Turning it off works from anywhere you are signed in. When you lend, this machine's model reads prompts assembled on your paired machines to answer them; the bridge keeps none of their content — the ledger holds only who, which direction, and how long.</p>";
    // Build 7 (Fix F2): the idle-only override, in the same
    // always-visible Lending section as the caps — the switch
    // flips on click like the others (wireMeshCard), the bridge
    // saves it into the lend block, and admission reads it on
    // the next request. The line under it states the current
    // posture's consequence in plain words, both ways.
    h += meshSwitchRow("mesh-idle-only", lendIdleOnly, "Lend only when idle", "Lend only when idle — off means this machine lends without waiting for a quiet minute");
    h += lendIdleOnly
      ? "<p style='color:var(--dim);font-size:12px'>On: this machine lends only after " + (lend.idleAfterSeconds != null ? lend.idleAfterSeconds : 60) + " seconds of quiet. Turn it off and it lends without waiting to be idle.</p>"
      : "<p style='color:var(--dim);font-size:12px'>Off: this machine lends without waiting to be idle — borrowed generations run alongside your own work, and your own replies may slow down. The caps above still apply, and lending still serves one borrowed generation at a time.</p>";
    if (m.lendEnabled) h += "<p><button type='button' class='ghost' id='mesh-stop' aria-label='Stop lending now — ends lending and aborts any borrowed generation in flight' style='min-height:44px;color:#f87171;border-color:#f87171'>Stop lending now</button></p>";
  h += "</div>";
  return h;
}
function meshPeerHtml(p, m) {
  var status = p.reachable === false ? meshUnreachableText(p) : p.probeBusy ? "busy" : p.probeLending ? "lending" : p.reachable === true ? (p.probeIdle ? "idle" : "reachable") : "not checked yet";
  var dot = p.reachable === false ? "#f87171" : p.probeBusy ? "#f5b544" : p.reachable === true ? "#34d399" : "#9fb3c8";
  var t = p.tunnel || { enabled: false };
  var road = p.reachable === true ? "direct road answers" : p.reachable === false ? "direct road dark" : "direct road not checked yet";
  if (t.enabled) road += " · backup route on";
  else if (meshTunnelArmed[p.nodeId]) road += " · backup route armed — fields to finish, in Roads below";
  var h = "<div class='card'><div style='display:flex;align-items:center;gap:8px;flex-wrap:wrap'><span class='dot' style='background:" + dot + "'></span><b>" + esc(p.name) + "</b><span class='mono' style='font-size:11px;color:var(--dim)'>" + esc(p.short || "") + "</span><span style='color:var(--dim);font-size:12px'>" + esc(status) + " · balance " + meshFmtSec(p.balanceSec) + "</span></div>" +
    (meshCapLine(p.capabilities) ? "<div style='color:var(--dim);font-size:12px;margin-top:4px'>" + esc(meshCapLine(p.capabilities)) + "</div>" : "") +
    (p.publicEndpoint ? "<div style='font-size:12px;color:var(--dim);margin-top:2px'>Public endpoint <span class='mono'>" + esc(p.publicEndpoint) + "</span> — declared on its card</div>" : "") +
          (p.reachable === false && p.lastProbeReason ? "<div style='font-size:12px;margin-top:4px'>" + esc(p.lastProbeReason) + "</div>" : "") +
          "<div style='display:flex;align-items:center;gap:8px;margin-top:6px;flex-wrap:wrap'><button class='switch' role='switch' aria-checked='" + (p.canBorrow ? "true" : "false") + "' data-mesh-borrow='" + esc(p.nodeId) + "' aria-label='Borrow from " + esc(p.name) + " — this machine borrows from it while this is on'><span class='knob'></span></button><span style='font-size:12px'>Can borrow from this machine</span>" +
          "<button type='button' class='ghost' data-mesh-remove='" + esc(p.nodeId) + "' aria-label='Remove the paired machine " + esc(p.name) + "' style='min-height:44px;margin-left:auto'>Remove</button></div>" +
    "<div style='font-size:12px;color:var(--dim);margin-top:6px'>Road: " + esc(road) + "</div>" +
    meshRelayHtml(p, m) + "</div>";
  return h;
}
function meshPeersHtml(m) {
  if (!m.enabled) return "";
  if (!m.peers || !m.peers.length) return "<div class='card'><h3 style='margin-top:0'>Peers</h3><p style='color:var(--dim);font-size:12px'>No machines paired yet. Download your PNG card at your door above, carry the file to the other machine, and bring its PNG card back to the drop zone there.</p></div>";
  var h = "";
  m.peers.forEach(function (p) { h += meshPeerHtml(p, m); });
  return h;
}
function meshRoadsHtml(m) {
  if (!m.enabled) return "";
  var h = "<div class='card'><h3 style='margin-top:0'>Roads</h3>";
  if (!m.peers || !m.peers.length) h += "<p style='color:var(--dim);font-size:12px'>No paired machines yet — the managed backup route is set per machine, here, once it is paired.</p>";
  (m.peers || []).forEach(function (p) {
    h += "<h4 style='margin:14px 0 0'>" + esc(p.name) + "</h4>" + meshTunnelHtml(p, m);
  });
  return h + "</div>";
}
function meshPngFileName() {
  // The download's file name, from this machine's Mesh name —
  // cosmetic only; the bridge names nothing.
  var m = meshViewState();
  var base = String(m.nodeName || "mesh").replace(/[^A-Za-z0-9-_]+/g, "-").replace(/^-+|-+$/g, "");
  return (base || "mesh") + "-mesh-card.png";
}
function meshImportPngFile(file) {
  // THE PNG card importer (Build 7, spec section 13): the file-
  // upload control and the drag-and-drop zone both land here,
  // so both paths run the same bytes through the same bridge
  // parser — there is no second, weaker path. The verdict,
  // success or one of the bridge's specific refusals, renders
  // at the drop zone in meshPngMsg: never a silent no-op.
  if (!file) return;
  if (CONFIG.backend !== "live") { meshPngMsg = "Pairing needs the live bridge — the bench does not simulate Mesh traffic."; render(); return; }
  meshPngMsg = "Reading " + file.name + "…"; render();
  var reader = new FileReader();
  reader.onload = function () {
    var url = String(reader.result || "");
    var b64 = url.indexOf(",") !== -1 ? url.split(",")[1] : "";
    if (!b64) { meshPngMsg = "That file could not be read — try the upload again."; render(); return; }
    fetch("/api/mesh/peers/png", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ png: b64 }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (j && j.ok) { meshState = j.mesh; meshPngMsg = "Peer added from the picture. Pairing is complete when this machine's card is added on the other side too."; }
        else meshPngMsg = (j && j.error) || "That picture was refused.";
        render();
      })
      .catch(function () { meshPngMsg = "The bridge did not answer — is it running?"; render(); });
  };
  reader.onerror = function () { meshPngMsg = "That file could not be read — try the upload again."; render(); };
  reader.readAsDataURL(file);
}
function meshTunnelPost(nodeId, enabled) {
  if (CONFIG.backend !== "live") { meshMsg = "The backup route needs the live bridge."; render(); return; }
  var hostEl = mainEl.querySelector("[data-mesh-tun-host='" + nodeId + "']");
  var portEl = mainEl.querySelector("[data-mesh-tun-port='" + nodeId + "']");
  var userEl = mainEl.querySelector("[data-mesh-tun-user='" + nodeId + "']");
  var body = { nodeId: nodeId, enabled: !!enabled };
  if (hostEl) body.sshHost = hostEl.value;
  if (portEl) body.sshPort = parseInt(portEl.value, 10);
  if (userEl) body.sshUser = userEl.value;
  fetch("/api/mesh/peers/tunnel", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) })
    .then(function (r) { return r.json(); })
    .then(function (j) { if (j && j.ok) { meshState = j.mesh; delete meshTunnelArmed[nodeId]; meshMsg = "Backup route saved."; } else meshMsg = (j && j.error) || "The bridge refused."; render(); })
    .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
}
function wireMeshCard() {
  var healthRun = document.getElementById("mesh-health-run");
  if (healthRun) healthRun.onclick = function () { meshRunHealth(); };
  var master = document.getElementById("mesh-master");
  if (master) master.onclick = function () { var m = meshViewState(); meshSave({ enabled: !m.enabled }); };
  var borrow = document.getElementById("mesh-borrow");
  if (borrow) borrow.onclick = function () { var m = meshViewState(); meshSave({ borrowEnabled: !m.borrowEnabled }); };
  var lend = document.getElementById("mesh-lend");
  if (lend) lend.onclick = function () { var m = meshViewState(); meshSave({ lendEnabled: !m.lendEnabled }); };
  var idleOnly = document.getElementById("mesh-idle-only");
  if (idleOnly) idleOnly.onclick = function () { var m = meshViewState(); var l = m.lend || {}; meshSave({ idleOnly: l.idleOnly === false ? true : false }); };
  var save = document.getElementById("mesh-save");
  if (save) save.onclick = function () {
    var nameEl = document.getElementById("mesh-name");
    var graceEl = document.getElementById("mesh-grace");
    var ledgerEl = document.getElementById("mesh-ledger-backup");
    var patch = {};
    if (nameEl) patch.nodeName = nameEl.value;
    // Build 4 (#154): the declared public endpoint rides the
    // same save as the name — the bridge canonicalizes it and
    // refuses the save in words when it is not a host:port.
    var endpointEl = document.getElementById("mesh-endpoint");
    if (endpointEl) patch.publicEndpoint = endpointEl.value;
    if (graceEl) patch.graceMs = parseInt(graceEl.value, 10);
    if (ledgerEl) patch.ledgerInBackup = !!ledgerEl.checked;
    // Build 7 (Fix F): the lending caps ride the same save.
    var shareEl = document.getElementById("mesh-cap-share");
    var genEl = document.getElementById("mesh-cap-gen");
    var peerCapEl = document.getElementById("mesh-cap-peer");
    if (shareEl) patch.maxSharePercent = parseInt(shareEl.value, 10);
    if (genEl) patch.maxGenerationsPerHour = parseInt(genEl.value, 10);
    if (peerCapEl) patch.perPeerPerHour = parseInt(peerCapEl.value, 10);
    meshSave(patch);
  };
  var stop = document.getElementById("mesh-stop");
  if (stop) stop.onclick = function () {
    if (CONFIG.backend !== "live") { meshSave({ lendEnabled: false }); return; }
    fetch("/mesh/lend/stop", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.ok) { meshState = j.mesh; meshMsg = "Lending is off — any borrowed generation in flight was stopped."; } else meshMsg = (j && j.error) || "The bridge refused."; render(); })
      .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
  };
  var pngDl = document.getElementById("mesh-card-png");
  if (pngDl) pngDl.onclick = function () {
    if (CONFIG.backend !== "live") { meshMsg = "Mock mode — the live bridge issues the real PNG card."; render(); return; }
    fetch("/api/mesh/card.png").then(function (r) {
      var ct = r.headers.get("Content-Type") || "";
      if (ct.indexOf("image/png") === -1) return r.json().then(function (j) { meshMsg = (j && j.error) || "The bridge could not produce a PNG card."; render(); });
      return r.blob().then(function (b) {
        var a = document.createElement("a");
        a.href = URL.createObjectURL(b);
        a.download = meshPngFileName();
        document.body.appendChild(a); a.click(); a.remove();
        setTimeout(function () { URL.revokeObjectURL(a.href); }, 4000);
        meshMsg = "PNG card downloaded — send the file itself (inside a .zip for chat apps)."; render();
      });
    }).catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
  };
  var pngFile = document.getElementById("mesh-png-file");
  if (pngFile) pngFile.onchange = function () {
    var f = pngFile.files && pngFile.files[0];
    pngFile.value = "";
    if (f) meshImportPngFile(f);
  };
  var pngZone = document.getElementById("mesh-png-zone");
  if (pngZone) {
    pngZone.addEventListener("dragover", function (e) {
      e.preventDefault();
      if (!meshPngDrag) { meshPngDrag = true; pngZone.style.borderColor = "var(--accent)"; }
    });
    pngZone.addEventListener("dragleave", function () {
      meshPngDrag = false; pngZone.style.borderColor = "var(--line)";
    });
    pngZone.addEventListener("drop", function (e) {
      e.preventDefault();
      meshPngDrag = false;
      var f = e.dataTransfer && e.dataTransfer.files && e.dataTransfer.files[0];
      if (f) meshImportPngFile(f); else render();
    });
  }
  mainEl.querySelectorAll("[data-mesh-borrow]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-borrow");
      var now = b.getAttribute("aria-checked") === "true";
      if (CONFIG.backend !== "live") { meshMsg = "Peer consent needs the live bridge."; render(); return; }
      fetch("/api/mesh/peers/can-borrow", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: nodeId, canBorrow: !now }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) meshState = j.mesh; else meshMsg = (j && j.error) || "The bridge refused."; render(); })
        .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
    };
  });
  mainEl.querySelectorAll("[data-mesh-relay]").forEach(function (b) {
    b.onclick = function () {
      var peer = b.getAttribute("data-peer");
      var other = b.getAttribute("data-other");
      var kind = b.getAttribute("data-mesh-relay");
      var now = b.getAttribute("aria-checked") === "true";
      if (CONFIG.backend !== "live") { meshMsg = "Relay consent needs the live bridge."; render(); return; }
      // The switch gate: the flip posts, and the page re-renders
      // from the bridge's returned payload — stored truth, never
      // the click's hope. A refusal (consent ticked from the DHD
      // door) lands in meshMsg at the head of this same card and
      // the re-render restores the switch to the stored state:
      // no snap-back without its reason, no reason stranded.
      fetch("/api/mesh/peers/relay", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: peer, kind: kind, other: other, on: !now }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) meshState = j.mesh; else meshMsg = (j && j.error) || "The bridge refused."; render(); })
        .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
    };
  });
  mainEl.querySelectorAll("[data-mesh-tunnel]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-tunnel");
      var now = b.getAttribute("aria-checked") === "true";
      if (now) {
        // Build 5 (#179): turning OFF always lands at once — a
        // saved route posts its disable; an armed-only route
        // simply disarms, with nothing to post.
        var wasArmed = !!meshTunnelArmed[nodeId];
        delete meshTunnelArmed[nodeId];
        var vsOff = meshViewState();
        var peersNow = vsOff.peers || (vsOff.borrow || {}).peers || [];
        var pv = null;
        for (var i = 0; i < peersNow.length; i++) if (peersNow[i].nodeId === nodeId) pv = peersNow[i];
        if (pv && pv.tunnel && pv.tunnel.enabled) meshTunnelPost(nodeId, false);
        else if (wasArmed) render();
        return;
      }
      // Build 5 (#179): turning ON posts only when the route
      // fields already stand filled — the bridge refuses an
      // enable with a blank SSH host or user, and that refusal
      // re-rendered the switch off under the operator's finger.
      // With a blank field the flip ARMS the route instead: the
      // switch stays on, the route block says what is missing,
      // and focus lands on the first blank field. Save route
      // commits an armed route once the fields stand filled.
      var hostEl = mainEl.querySelector("[data-mesh-tun-host='" + nodeId + "']");
      var userEl = mainEl.querySelector("[data-mesh-tun-user='" + nodeId + "']");
      var host = hostEl ? hostEl.value.trim() : "";
      var user = userEl ? userEl.value.trim() : "";
      if (host && user) { delete meshTunnelArmed[nodeId]; meshTunnelPost(nodeId, true); return; }
      meshTunnelArmed[nodeId] = true;
      render();
      var miss = mainEl.querySelector(!host ? "[data-mesh-tun-host='" + nodeId + "']" : "[data-mesh-tun-user='" + nodeId + "']");
      if (miss) { try { miss.focus(); } catch (e) {} }
    };
  });
  mainEl.querySelectorAll("[data-mesh-tunnel-save]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-tunnel-save");
      var sw = mainEl.querySelector("[data-mesh-tunnel='" + nodeId + "']");
      meshTunnelPost(nodeId, sw ? sw.getAttribute("aria-checked") === "true" : false);
    };
  });
  mainEl.querySelectorAll("[data-mesh-tunnel-auth]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-tunnel-auth");
      var action = b.getAttribute("data-auth-action") || "authorize";
      if (CONFIG.backend !== "live") { meshMsg = "Tunnel keys need the live bridge."; render(); return; }
      // Build 8: authorize opens the crown confirm strip on the
      // peer's own card; revoke — open through every door —
      // posts at once.
      if (action === "authorize") { meshTunnelConfirmFor = nodeId; meshTunnelAuthMsg = null; render(); return; }
      fetch("/api/mesh/peers/tunnel-authorize", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: nodeId, action: action }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) { meshState = j.mesh; meshTunnelAuthMsg = { nodeId: nodeId, ok: true, text: j.message || "Saved." }; } else meshTunnelAuthMsg = { nodeId: nodeId, ok: false, text: (j && (j.error || j.message)) || "The bridge refused." }; render(); })
        .catch(function () { meshTunnelAuthMsg = { nodeId: nodeId, ok: false, text: "The bridge did not answer — is it running?" }; render(); });
    };
  });
  mainEl.querySelectorAll("[data-mesh-tunnel-confirm]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-tunnel-confirm");