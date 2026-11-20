# app/index.html — part 8 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 13792–15697 of app/index.html.

      else if (tdev.detail) h += "<p style='color:var(--dim);font-size:12px'>Speech device: " + esc(tdev.detail) + "</p>";
    }
  h += voiceSettingsHtml();
    if (CONFIG.backend === "live" && voiceState && (!sttOk || !ttsOk)) {
      h += "<div class='card' style='border-color:#f5b544'><b>Her voice isn't installed yet.</b><br><span style='color:var(--dim)'>" + esc((!sttOk && voiceState.stt.detail) || "") + ((!sttOk && !ttsOk) ? "<br>" : "") + esc((!ttsOk && voiceState.tts.detail) || "") + "</span><br><span style='color:var(--dim)'>The speech engines are optional on purpose — the bridge stays dependency-free until you want her voice. The installer offers them at the voice step (an isolated venv), or: <span class='mono'>python3 -m pip install -r requirements-voice.txt</span>. Models download once, on first use, into voice/models/ — and that folder survives updates and reinstalls like her chips and her Soul/Memory.</span></div>";
      if (voiceState.python) h += "<p style='color:var(--dim);font-size:12px'>Diagnostics — bridge Python: <span class='mono'>" + esc(voiceState.python) + "</span> · installer's voice venv: " + (voiceState.venv ? "<span class='mono'>" + esc(voiceState.venv) + "</span> (adopted — its engines are on this bridge's module path)" : "not found beside the app") + " · if you picked voice at install, its log is <span class='mono'>voice/voice-install.log</span> and names any step that failed.</p>";
    }
    // Known issue (field report, 2026-11-07): the microphone in the
    // desktop window. Noted here at the operator's request — the
    // note stays until the fix lands, and it never claims it has.
    h += "<div class='card' style='border-color:#f5b544'><b>Known issue — microphone access in the desktop app.</b><br><span style='color:var(--dim)'>Field report from the operator's Framework 13 (Pop!_OS), 2026-11-07: in the standalone window, Navi cannot hear him — push-to-talk and calls get no working microphone — even though her shell is built to grant the dashboard's mic requests and the OS prompt is allowed. <b>Typing works regardless:</b> the call panel takes typed lines ('type what you'd say'), Chat takes typed messages, and she still answers out loud — a whole call can be had by keyboard. The fix ships in <b>v0.4.1e</b>: her shell now switches WebKit's media-capture settings on explicitly — the layer below permission, where the field evidence pointed (a mic request granted, and the stream delivering nothing). The change is compile-verified; this note stays on the Voice page until the field confirms she hears you in the window, and it will say so plainly if the bench disagrees. The same entry lives in the wiki's Troubleshooting page. <b>New in v0.3.0e — Build 6:</b> this window reports its mic reading to the bridge on every push-to-talk attempt — the reading below also lands in the bridge's log (start.log, beside the app). That record is what aimed the v0.4.1e fix: the window's own reading said the request was granted, which is how the break was known to sit below permission, in the capture settings the fix now turns on.</span><br><span style='color:var(--dim);font-size:12px'>This window's own mic, this session: " + (micState === "ok" ? "a mic request was granted — if she still can't hear you in the desktop app, that is the known issue above." : micState === "refused" ? "the last mic request was refused or failed — the known issue, live in this window." : micState === "unavailable" ? "this window exposes no recording API at all — typing is the way in here." : "not tested yet — press Push to talk on a call and this line reports what happened.") + "</span></div>";
    if (CONFIG.backend !== "live") {
      h += "<div class='card' style='border-color:#f5b544;color:#f5b544'>Mock mode: this call is simulated — her lines are spoken by your browser's built-in voice and answered by the mock brain. Switch the backend to Live in Settings for the real local engines (faster-whisper + Kokoro).</div>";
    }
    // Past calls.
    if (CONFIG.backend === "live") {
      h += "<h3 style='margin-top:16px'>Past calls</h3>";
      if (voiceCalls === null) h += "<p style='color:var(--dim)'>Loading the call log…</p>";
      else if (!voiceCalls.length) h += "<p style='color:var(--dim)'>No calls yet — the log (voice/calls.json) starts with your first one.</p>";
      else h += voiceCalls.slice().reverse().map(function (c) {
        var n = (c.turns || []).length;
        return "<div class='card'><b>" + esc(String(c.started_at || "").slice(0, 16).replace("T", " ")) + "</b> <span style='color:var(--dim)'>" + n + " turns" + (c.ended_at ? "" : " · (in progress)") + "</span></div>";
      }).join("");
    }
  return h;
}
function voiceSettingsHtml() {
  // Settings -> Voice: the voice settings card — since Build 6
  // (#113) it shares its tab with the call panel itself, which
  // moved in from the retired Voice view. Same v-* ids, same
  // wiring — one home for everything voice.
  var vc = voiceState ? voiceState.config : { stt_model: "small.en", tts_engine: "kokoro", tts_voice: "bf_isabella", rate: 1, pitch: 1 };
  var vEnabled = voiceState ? voiceState.enabled : true;
  var h = "";
  h += "<div class='card'><h3>Voice settings</h3>";
  if (CONFIG.backend !== "live") {
    h += "<p style='color:var(--dim)'>Settings save through the live bridge — in mock mode they're shown as the bench holds them. Switch to Live in Settings to change them for real.</p>";
  }
  h += "<div style='display:flex;align-items:center;gap:8px'><button class='switch' id='v-enabled' role='switch' aria-checked='" + (vEnabled ? "true" : "false") + "' aria-label='Voice enabled'><span class='knob'></span></button><span>Voice enabled — she can hear and be called</span></div>";
  h += "<div class='row' style='margin-top:10px'><label class='grow'>She hears with · faster-whisper model<br><select id='v-stt' aria-label='Speech to text model'><option value='tiny.en'" + (vc.stt_model === "tiny.en" ? " selected" : "") + ">tiny.en — fastest, rough edges (~75 MB)</option><option value='base.en'" + (vc.stt_model === "base.en" ? " selected" : "") + ">base.en — quick and decent (~150 MB)</option><option value='small.en'" + (vc.stt_model === "small.en" ? " selected" : "") + ">small.en — the sweet spot on CPU (~500 MB)</option></select></label>";
  h += "<label class='grow'>She speaks with · engine<br><select id='v-engine' aria-label='Text to speech engine'><option value='kokoro'" + (vc.tts_engine === "kokoro" ? " selected" : "") + ">Kokoro-82M — natural, CPU-friendly</option><option value='piper'" + (vc.tts_engine === "piper" ? " selected" : "") + ">Piper — lighter fallback</option></select></label></div>";
  // The voice list is auto-generated: in live mode the bridge
  // enumerates the voices this machine's TTS installation
  // actually has (Kokoro's installed .pt voices, or the catalog
  // the installed package pulls from) and the picker is built
  // from that — never a hardcoded list. The built-in lists below
  // are only the mock backend's simulated installation.
  var vlist;
  if (CONFIG.backend === "live" && voiceState && voiceState.voices && voiceState.voices.length) {
    vlist = voiceState.voices.map(function (v) { return [v.id, v.label]; });
    if (!vlist.some(function (o) { return o[0] === vc.tts_voice; })) vlist.unshift([vc.tts_voice, vc.tts_voice + " (saved)"]);
  } else {
    vlist = vc.tts_engine === "piper" ? PIPER_VOICES : KOKORO_VOICES;
  }
  h += "<div class='row' style='margin-top:8px'><label class='grow'>Her voice<br><select id='v-voice' aria-label='Her voice'>" + vlist.map(function (o) { return "<option value='" + o[0] + "'" + (vc.tts_voice === o[0] ? " selected" : "") + ">" + esc(o[1]) + "</option>"; }).join("") + "</select></label></div>";
  if (CONFIG.backend === "live" && voiceState && voiceState.voices_source) {
    h += "<p style='color:var(--dim);font-size:12px;margin-top:4px'>" + (voiceState.voices_source === "installed" ? "This list is enumerated from the voices installed on this machine — it shows exactly what her " + (vc.tts_engine === "piper" ? "Piper" : "Kokoro") + " has." : voiceState.voices_source === "catalog" ? "This list is enumerated from the voice catalog the installed " + (vc.tts_engine === "piper" ? "Piper" : "Kokoro") + " pulls from — a voice downloads the first time she speaks with it." : "The full voice list appears once the speech engine is installed — for now only her saved voice is known.") + " Default: Isabella (British female).</p>";
  }
  h += "<div class='row' style='margin-top:8px'><label class='grow'>Speed · <span id='v-rate-v'>" + Number(vc.rate).toFixed(2) + "×</span><br><input id='v-rate' type='range' min='0.5' max='2' step='0.05' value='" + vc.rate + "' aria-label='Speaking speed' style='width:100%'></label>";
  h += "<label class='grow'>Pitch · <span id='v-pitch-v'>" + Number(vc.pitch).toFixed(2) + "×</span><br><input id='v-pitch' type='range' min='0.5' max='2' step='0.05' value='" + vc.pitch + "' aria-label='Voice pitch' style='width:100%'></label></div>";
  h += "<label style='display:flex;gap:8px;align-items:center;margin-top:10px'><input type='checkbox' id='v-speakreplies' " + (CONFIG.speakReplies ? "checked" : "") + " style='width:auto'> Speak typed-chat replies aloud on this machine too (the bridge plays them; on a call she speaks in this panel instead)</label>";
  h += "<p style='color:var(--dim);font-size:12px'>Settings live in config/navi.config.json's voice section. Speech models download once into voice/models/ and are never removed by updates or reinstalls.</p></div>";
  return h;
}

function deckSleeveChip(c) {
  var stem = String(c.file || "").replace(/.png$/i, "");
  var delName = c.sourceKey ? c.sourceKey : "local/" + stem;
  var state = c.unreadable ? "Couldn't be read — deletable, not runnable"
    : c.sourceKey ? "Installed from " + c.sourceKey
    : "Forged here / dropped in";
  if (c.locallyEdited) state += " · EDITED LOCALLY";
  return "<div class='sleeve'><img class='sleeve-img' src='/api/chips/file?name=" + encodeURIComponent(c.file) + "' alt='The cartridge picture for " + esc(c.name) + "'><div class='sleeve-name'>" + esc(c.name) + "</div><div class='sleeve-state'>Chip" + (c.glyph ? " · " + esc(c.glyph) + " face" : "") + (c.version ? " · v" + esc(c.version) : "") + "<br>" + esc(state) + "</div><div class='deck-actions'>"
    + (c.unreadable ? "" : "<button class='act' data-deckrun='" + esc(stem) + "' aria-label='Run chip " + esc(c.name) + " in chat'>Run</button>")
    + "<a class='ghost' href='/api/chips/file?name=" + encodeURIComponent(c.file) + "' download='" + esc(c.file) + "'>Download picture</a>"
    + (c.unreadable ? "" : "<button class='ghost' data-deckstudio='" + esc(c.file) + "' aria-label='Open chip " + esc(c.name) + " in Chip Studio'>Studio</button>")
    + "<button class='ghost' data-deckuninstall='" + esc(delName) + "' aria-label='Uninstall chip " + esc(c.name) + "'>Uninstall</button></div></div>";
}
function deckSleeveMesh(c) {
  var state = c.own ? "This machine's own card" : c.paired ? "Paired" : "Card on file — not paired";
  return "<div class='sleeve'><img class='sleeve-img' src='/api/deck/file?name=" + encodeURIComponent(c.file) + "' alt='The Mesh pairing card picture for " + esc(c.name) + "' style='width:112px;height:84px;image-rendering:auto'><div class='sleeve-name'>" + esc(c.name) + "</div><div class='sleeve-state'>Apex | Mesh card<br>" + esc(state) + "</div><div class='deck-actions'>"
    + "<a class='ghost' href='/api/deck/file?name=" + encodeURIComponent(c.file) + "' download='" + esc(c.file) + "'>Download picture</a>"
    + (c.paired && !c.own ? "<button class='ghost' data-deckunpair='" + esc(c.nodeId) + "' aria-label='Unpair " + esc(c.name) + "'>Unpair</button>" : "")
    + "</div></div>";
}
function deckSleeveRag(p) {
  return "<div class='sleeve'><img class='sleeve-img' src='/api/rag/cartridge.png?slug=" + encodeURIComponent(p.slug) + "' alt='The knowledge cartridge picture for " + esc(p.name) + "' onerror='this.style.display=\"none\"'><div class='sleeve-name'>" + esc(p.name) + "</div><div class='sleeve-state'>Knowledge pack · " + p.docCount + " documents<br>" + (p.active ? "Active — feeds her answers" : "Off") + "</div><div class='deck-actions'>"
    + "<button class='ghost' data-deckragtoggle='" + esc(p.fullName) + "' data-active='" + (p.active ? "1" : "") + "' aria-label='" + (p.active ? "Deactivate" : "Activate") + " pack " + esc(p.name) + "'>" + (p.active ? "Deactivate" : "Activate") + "</button>"
    + "<a class='ghost' href='/api/rag/cartridge.png?slug=" + encodeURIComponent(p.slug) + "' download='" + esc(p.slug) + "-cartridge.png'>Download picture</a>"
    + "<button class='ghost' data-deckragdelete='" + esc(p.fullName) + "' aria-label='Remove pack " + esc(p.name) + "'>Remove</button></div></div>";
}
function deckHtml() {
  var h = "<h3>ChipDeck</h3>";
  h += "<p style='color:var(--dim)'>The binder: everything on this machine that rides a picture — chips, Apex | Mesh pairing cards, knowledge cartridges — each sleeve holding the actual picture, its state, and its actions. What never rides a picture stays out: Soul, Memory, settings, the Vault, and keys live in backups, never in cartridges.</p>";
  h += "<div class='deck-drop' id='deck-drop'><b>Drop a picture to slot it in</b><br><span style='color:var(--dim);font-size:12px'>A chip, a knowledge cartridge, or a Mesh card — the data inside the picture says which. Or</span> <button class='ghost' id='deck-pick'>pick a picture…</button><input id='deck-file' type='file' accept='image/png' style='display:none' aria-label='Choose a picture to install'></div>";
  if (deckMsg) h += "<p>" + esc(deckMsg) + "</p>";
  if (CONFIG.backend !== "live") { h += "<div class='card'>ChipDeck reads the live bridge — switch the backend to Live in Settings.</div>"; return h; }
  if (!deckData) { h += "<p style='color:var(--dim)'>Opening the binder…</p>"; return h; }
  h += "<h3 style='margin-top:16px'>Chips · " + deckData.chips.length + "</h3>";
  h += deckData.chips.length ? "<div class='deck-grid'>" + deckData.chips.map(deckSleeveChip).join("") + "</div>" : "<p style='color:var(--dim)'>No chips yet — install one from Navi User Repo (NUR), forge one in Chip Studio, or drop its picture above.</p>";
  h += "<h3 style='margin-top:16px'>Apex | Mesh cards · " + deckData.meshCards.length + "</h3>";
  h += deckData.meshCards.length ? "<div class='deck-grid'>" + deckData.meshCards.map(deckSleeveMesh).join("") + "</div>" : "<p style='color:var(--dim)'>No pairing cards on file — cards land in config/mesh/cards/ when a backup restores them; your own card is on the Mesh page.</p>";
  h += "<h3 style='margin-top:16px'>Knowledge cartridges · " + deckData.ragPacks.length + "</h3>";
  h += deckData.ragPacks.length ? "<div class='deck-grid'>" + deckData.ragPacks.map(deckSleeveRag).join("") + "</div>" : "<p style='color:var(--dim)'>No knowledge packs yet — forge or install one on the RAG tab and its cartridge picture shows here.</p>";
  return h;
}
// Guardian (0.1.9e Build 14, issue #156): the face of the
// bridge's system monitor — the engine that protects Navi's
// headroom lives bridge-side and is always on with the bridge
// (headless included); this view is its window. State arrives
// from GET /api/guardian/state, polled every 3 seconds while
// the view is open (the app's live-refresh idiom: a timer that
// exists only while its view does, like the Updater's result
// poll), and the poll stops the moment the view changes —
// render() itself enforces that below. A refused fetch fails
// soft into the "Guardian unreachable" note over the view's
// static skeleton, never a throw: the skeleton (header, cards,
// settings form) renders from guardianState === null exactly
// as it renders from a live state.
var guardianState = null;    // live: last /api/guardian/state answer
var guardianError = false;   // the last poll was refused / unanswered
var guardianMsg = "";        // the settings save's verdict line
var guardianTimer = null;    // the 3s poll, only while the view is open
var guardianFetching = false;
function guardianStopPolling() { if (guardianTimer) { clearInterval(guardianTimer); guardianTimer = null; } }
function guardianFormFocused() { var ae = document.activeElement; return !!(ae && ae.id && String(ae.id).indexOf("guardian-") === 0); }
function guardianStartPolling() {
  if (guardianTimer) return;
  fetchGuardianState();
  guardianTimer = setInterval(function () {
    if (view !== "Guardian") { guardianStopPolling(); return; }
    fetchGuardianState();
  }, 3000);
}
function fetchGuardianState() {
  if (guardianFetching) return;
  guardianFetching = true;
  fetch("/api/guardian/state").then(function (r) { return r.json(); }).then(function (j) {
    guardianFetching = false;
    if (j && typeof j === "object") { guardianState = j; guardianError = false; }
    // A re-render mid-edit would strand the operator's typing
    // in the settings form — the fresh state waits for the
    // next natural render while any Guardian field holds focus.
    if (view === "Guardian" && !guardianFormFocused()) render();
  }).catch(function () {
    guardianFetching = false;
    guardianError = true;
    if (view === "Guardian" && !guardianFormFocused()) render();
  });
}
function guardianSaveSettings() {
  var cur = (guardianState && guardianState.settings) || {};
  var num = function (id, fallback) { var el = document.getElementById(id); var v = el ? parseFloat(el.value) : NaN; return isNaN(v) ? (typeof fallback === "number" ? fallback : 0) : v; };
  var chk = function (id, fallback) { var el = document.getElementById(id); return el ? !!el.checked : !!fallback; };
  var ta = document.getElementById("guardian-expendable");
  var names = ta ? ta.value.split("\n").map(function (s) { return s.trim(); }).filter(function (s) { return !!s; }) : [];
  // The full settings object: every key the bridge holds rides
  // back, with the form's edits laid over it.
  var settings = {};
  Object.keys(cur).forEach(function (k) { settings[k] = cur[k]; });
  settings.enabled = chk("guardian-enabled", cur.enabled);
  settings.reserveMb = num("guardian-reserve", cur.reserveMb);
  settings.sampleSeconds = num("guardian-sample", cur.sampleSeconds);
  settings.slowGenSeconds = num("guardian-slowgen", cur.slowGenSeconds);
  settings.cpuCapPercent = num("guardian-cpu-cap", cur.cpuCapPercent);
  settings.killEnabled = chk("guardian-kill-enabled", cur.killEnabled);
  settings.expendable = names;
  settings.warnAfterSeconds = num("guardian-warn-after", cur.warnAfterSeconds);
  settings.killGraceSeconds = num("guardian-kill-grace", cur.killGraceSeconds);
  fetch("/api/guardian/settings", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(settings) }).then(function (r) { return r.json(); }).then(function (j) {
    if (j && typeof j === "object") { guardianState = (j.state && typeof j.state === "object") ? j.state : j; guardianError = false; }
    guardianMsg = "Guardian settings saved — the Guardian is working to them now.";
    if (view === "Guardian") render();
  }).catch(function () {
    guardianMsg = "The settings could not be saved — the bridge didn't answer, so nothing was changed.";
    if (view === "Guardian") render();
  });
}
function guardianNum(v) { return v == null || v === "" || isNaN(Number(v)) ? "—" : String(Math.round(Number(v))); }
function guardianTs(ts) {
  if (ts == null || ts === "") return "";
  if (typeof ts === "number") {
    var d = new Date(ts > 1e12 ? ts : ts * 1000);
    return isNaN(d.getTime()) ? String(ts) : d.toLocaleString();
  }
  return String(ts);
}
function guardianPressureInfo(p) {
  if (p === "ok") return { cls: "guardian-chip-ok", label: "OK", words: "Comfortable — Navi has room to think." };
  if (p === "tight") return { cls: "guardian-chip-tight", label: "Tight", words: "Memory is tight — getting close to the reserve Navi needs." };
  if (p === "critical") return { cls: "guardian-chip-critical", label: "Critical", words: "Critical — the Guardian is acting to protect Navi's headroom." };
  return { cls: "", label: "Unknown", words: "No report yet — the Guardian has not answered." };
}
function guardianHtml() {
  var st = guardianState;
  var set = (st && st.settings) || {};
  var pi = guardianPressureInfo(st ? st.pressure : null);
  var reserve = st && st.reserveMb != null ? st.reserveMb : set.reserveMb;
  var expendable = Array.isArray(set.expendable) ? set.expendable : (st && Array.isArray(st.expendable) ? st.expendable : []);
  var mesh = (st && st.mesh) || {};
  var meshPeers = typeof mesh.peers === "number" ? mesh.peers : (Array.isArray(mesh.peers) ? mesh.peers.length : 0);
  var h = "<div class='guardian-view'>";
  h += "<h3>Guardian</h3>";
  h += "<p style='color:var(--dim)'>The system monitor that protects Navi's headroom: it watches this machine's memory and Navi's own footprint, and when other work crowds her out it climbs a ladder — watch, warn, throttle, and only ever as a last resort, kill — and only a process you have named expendable below. The Guardian itself runs with the bridge, even headless; this page is its face.</p>";
  if (guardianError) h += "<div class='card guardian-unreachable' style='border-color:#f5b544;color:#f5b544'>Guardian unreachable — the bridge did not answer /api/guardian/state. The cards below show the Guardian's shape; its live numbers land here when the bridge answers.</div>";
  if (guardianMsg) h += "<p>" + esc(guardianMsg) + "</p>";
  // Summary
  h += "<div class='card guardian-card guardian-summary'><h3>Guardian — summary</h3>";
  h += "<div><span class='guardian-chip " + pi.cls + "'>" + esc(pi.label) + "</span> <span>" + esc(pi.words) + "</span></div>";
  h += "<div class='guardian-stats'>";
  h += "<div class='guardian-stat'><span class='guardian-k'>Free RAM</span><b>" + guardianNum(st ? st.freeMb : null) + " MB</b> <span style='color:var(--dim)'>free · reserve is " + guardianNum(reserve) + " MB</span></div>";
  h += "<div class='guardian-stat'><span class='guardian-k'>Navi's footprint</span><b>" + guardianNum(st ? st.llamafileRssMb : null) + " MB</b> <span style='color:var(--dim)'>held by navi.llamafile</span></div>";
  h += "<div class='guardian-stat'><span class='guardian-k'>Generating</span>" + (st && st.generating ? "<b style='color:var(--cy)'>Generating now</b> <span style='color:var(--dim)'>— Navi is writing a reply</span>" : "<span>Not generating right now</span>") + "</div>";
  h += "<div class='guardian-stat'><span class='guardian-k'>Guardian</span>" + (st && st.enabled === false ? "<span>The Guardian is <b>off</b> — turn it on in the settings below.</span>" : "<span>On watch over this machine.</span>") + "</div>";
  h += "</div>";
  h += mesh.enabled
    ? "<p style='margin-top:10px'><b>Apex | Mesh</b> <span style='color:var(--dim)'>is on — borrowing is available from " + meshPeers + " paired machine" + (meshPeers === 1 ? "" : "s") + " if this machine runs short; the Guardian can lean on the Mesh before it ever leans on a local process.</span></p>"
    : "<p style='margin-top:10px'><b>Apex | Mesh</b> <span style='color:var(--dim)'>is off — the Guardian is working locally only, with this machine's own memory.</span></p>";
  h += "</div>";
  // Live watch
  var top = st && Array.isArray(st.top) ? st.top : [];
  h += "<div class='card guardian-card guardian-watch'><h3>Live watch</h3>";
  h += "<p style='color:var(--dim)'>The heaviest processes on this machine right now, sampled by the Guardian. Navi's own house is marked; everything else is a guest — dimmed, watched, and never actionable from here.</p>";
  h += "<div class='guardian-table-wrap'><table class='guardian-table'><thead><tr><th>Process</th><th class='guardian-pid-col'>PID</th><th>CPU %</th><th>Memory (MB)</th><th>State</th></tr></thead><tbody>";
  if (top.length) {
    h += top.map(function (p) {
      return "<tr class='guardian-row" + (p.ours ? "" : " guardian-foreign") + "'><td><b>" + esc(p.name || "—") + "</b>" + (p.ours ? " <span class='guardian-badge guardian-badge-ours'>Navi / Apex</span>" : "") + "</td><td class='guardian-pid-col mono'>" + esc(p.pid == null ? "—" : String(p.pid)) + "</td><td class='mono'>" + guardianNum(p.cpu) + "</td><td class='mono'>" + guardianNum(p.rssMb) + "</td><td>" + (p.throttled ? "<span class='guardian-badge'>Throttled</span>" : "<span style='color:var(--dim)'>—</span>") + "</td></tr>";
    }).join("");
  } else {
    h += "<tr class='guardian-row'><td colspan='5' style='color:var(--dim)'>No processes reported yet — the Guardian's first sample lands here.</td></tr>";
  }
  h += "</tbody></table></div></div>";
  // Learned offenders
  var offenders = st && Array.isArray(st.offenders) ? st.offenders : [];
  h += "<div class='card guardian-card guardian-offenders'><h3>Learned offenders</h3>";
  if (offenders.length) {
    h += "<p style='color:var(--dim)'>Processes that have collided with Navi before — counted, not modeled. The Guardian learns this machine by keeping score.</p>";
    h += "<div class='guardian-table-wrap'><table class='guardian-table'><thead><tr><th>Process</th><th>Seen</th><th>Collisions</th><th>Warns</th><th>Throttles</th><th>Kills</th></tr></thead><tbody>";
    h += offenders.map(function (o) {
      return "<tr class='guardian-row'><td><b>" + esc(o.name || "—") + "</b>" + (o.lastSeen ? "<br><span style='color:var(--dim);font-size:11px'>last seen " + esc(guardianTs(o.lastSeen)) + "</span>" : "") + "</td><td class='mono'>" + guardianNum(o.seen) + "</td><td class='mono'>" + guardianNum(o.collisions) + "</td><td class='mono'>" + guardianNum(o.warns) + "</td><td class='mono'>" + guardianNum(o.throttles) + "</td><td class='mono'>" + guardianNum(o.kills) + "</td></tr>";
    }).join("");
    h += "</tbody></table></div>";
  } else {
    h += "<p style='color:var(--dim)'>Nothing has collided with Navi yet — the Guardian is still learning this machine.</p>";
  }
  h += "</div>";
  // Settings
  var sv = function (v) { return v == null ? "" : esc(String(v)); };
  h += "<div class='card guardian-card guardian-settings'><h3>Guardian settings</h3>";
  h += "<label class='guardian-check'><input type='checkbox' id='guardian-enabled'" + (set.enabled !== false ? " checked" : "") + "> <span><b>Guardian enabled</b> — watch this machine and protect Navi's headroom</span></label>";
  h += "<div class='guardian-fields'>";
  h += "<label for='guardian-reserve'>Reserve (MB) — free RAM kept for Navi<input class='guardian-in' id='guardian-reserve' type='number' min='0' step='1' value='" + sv(set.reserveMb) + "'></label>";
  h += "<label for='guardian-sample'>Sample every (seconds)<input class='guardian-in' id='guardian-sample' type='number' min='1' step='1' value='" + sv(set.sampleSeconds) + "'></label>";
  h += "<label for='guardian-slowgen'>Slow generation (seconds) — a reply slower than this counts against the machine<input class='guardian-in' id='guardian-slowgen' type='number' min='0' step='1' value='" + sv(set.slowGenSeconds) + "'></label>";
  h += "<label for='guardian-cpu-cap'>CPU cap (%) — the ceiling the Guardian throttles offenders to<input class='guardian-in' id='guardian-cpu-cap' type='number' min='1' max='100' step='1' value='" + sv(set.cpuCapPercent) + "'></label>";
  h += "<label for='guardian-warn-after'>Warn after (seconds) of crowding<input class='guardian-in' id='guardian-warn-after' type='number' min='0' step='1' value='" + sv(set.warnAfterSeconds) + "'></label>";
  h += "<label for='guardian-kill-grace'>Kill grace (seconds) after a final warning<input class='guardian-in' id='guardian-kill-grace' type='number' min='0' step='1' value='" + sv(set.killGraceSeconds) + "'></label>";
  h += "</div>";
  h += "<label class='guardian-check'><input type='checkbox' id='guardian-kill-enabled'" + (set.killEnabled ? " checked" : "") + "> <span><b>Killing enabled</b> — as a last resort, the Guardian may kill a process named expendable below</span></label>";
  h += "<label for='guardian-expendable' style='display:block;font-size:12px;color:var(--dim);margin-top:10px'>Expendable processes — the ONLY names the Guardian may ever kill; ships empty on purpose<textarea class='guardian-expendable' id='guardian-expendable' rows='5' placeholder='One process name per line — leave this empty and the Guardian can never kill anything'>" + esc(expendable.join("\n")) + "</textarea></label>";
  h += "<p style='color:var(--dim);font-size:12px'>Killing never happens to anything not on that list — not Navi, not Apex, not a stranger the Guardian merely dislikes. With the list empty, the Guardian can watch, warn, and throttle, but it can never kill.</p>";
  h += "<button type='button' class='act guardian-save' id='guardian-save'>Save Guardian settings</button>";
  h += "</div>";
  // Action log
  var log = st && Array.isArray(st.log) ? st.log.slice().reverse() : [];
  h += "<div class='card guardian-card guardian-log-card'><h3>Action log</h3>";
  if (log.length) {
    h += "<ul class='guardian-log'>" + log.map(function (e) {
      return "<li><span class='mono' style='color:var(--dim);font-size:11px'>" + esc(guardianTs(e.ts)) + "</span> <b>" + esc(e.kind || "") + "</b> " + esc(e.text || "") + (e.name ? " <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(e.name) + (e.pid != null ? " · pid " + esc(String(e.pid)) : "") + "</span>" : "") + "</li>";
    }).join("") + "</ul>";
  } else {
    h += "<p style='color:var(--dim)'>No Guardian actions yet — when the Guardian warns, throttles, or kills, the receipt lands here.</p>";
  }
  h += "</div>";
  h += "</div>";
  return h;
}
function render() {
  settingsTabValid();
  // The Guardian's poll lives only while its view does (Build
  // 14, #156): leaving the view stops the timer here, in the
  // one place every view change passes through.
  if (view !== "Guardian") guardianStopPolling();
  // A popped docs window is docs-only: no nav, no status column,
  // no polling behind the wiki.
  if (DOCS_POP) { renderDocsPop(); return; }
  // The Memory view retired in Build 4: any state still naming it
  // (an old deep link, a saved view) lands on Chat, where its
  // chats now live as the Chats tab.
  if (view === "Memory") view = "Chat";
  // The Voice view retired in Build 6 (#113) the same way: a
  // stale state naming it lands on Chat; her voice lives in
  // Settings → Voice now.
  if (view === "Voice") view = "Chat";
  // The Updater view folded into Settings → System in Build 10
  // (#196): a stale state naming it lands on the System tab
  // with the updater fold open.
  if (view === "Updater") { view = "Settings"; settingsTab = "system"; updaterFoldOpen = true; }
  // A portaled article reader from the previous render is
  // taken down before the new view paints — it lives at <body>
  // level, outside the view root's HTML (Build 10, #196).
  document.querySelectorAll("body > #wire-reader, body > #wire-reader-shade, body > #fc-tray, body > #fc-win, body > #fc-shade, body > #vault-shade, body > #vault-sheet").forEach(function (n) { n.remove(); });
  if (!wireReader) document.body.classList.remove("reader-open");
  // Focus return for the chat composer: render() rebuilds the whole
  // view, which would strand the operator's cursor after every send.
  // If the composer held focus — or a send just asked for it back —
  // the rebuilt composer takes focus again at the end of render.
  var keepChatFocus = view === "Chat" && (chatFocusWanted || (document.activeElement && document.activeElement.id === "chat-in"));
  // If the operator closed the popped docs window directly, take
  // back the page it was left on and dock the wiki again.
  if (docsPopWin && docsPopWin.closed) {
    docsPopWin = null;
    try { var p = localStorage.getItem("navi.docs.page"); if (p) docsPage = p; } catch (e) {}
  }
  appEl.classList.toggle("closed", navClosed);
  appEl.classList.toggle("no-status", statusHidden);
  appEl.classList.toggle("dock-open", dockOpen);
  // Build 10 (#131): while Chat is up, the frame carries the
  // in-chat class — at phone widths that is what height-bounds
  // the frame (see the small-screen block), so the composer's
  // foot is the frame's rendered bottom, not a page position.
  appEl.classList.toggle("in-chat", view === "Chat");
  // Build 11 (#135): focus mode's class is render()'s
  // reflection of the one vaultFocus flag — cleared by any
  // view switch, so the frame can never stay collapsed
  // around another view.
  if (view !== "Vault") vaultFocus = false;
  appEl.classList.toggle("vault-focus", vaultFocus);
  // The Vault's mobile furniture (#279) belongs to the
  // Vault: leaving the view resets the reader to the list
  // and closes the create sheet, so neither can hang over
  // another view.
  if (view !== "Vault") { vaultReader = false; vaultSheetOpen = false; vaultElArm = ""; }
  // The room's overlays (#270) belong to the room: leaving
  // the view closes them, so a tray can never hang over
  // another destination.
  if (view !== "FediChat") { fcTrayOpen = false; fcWinOpen = false; }
  // Build 13 (#146): Studio focus mode's class is the same
  // kind of reflection of the one studioFocus flag — cleared
  // by any move away from the Studio, so the collapsed frame
  // can never strand around another tab or view.
  if (!(view === "Chips" && chipsTab === "studio")) studioFocus = false;
  appEl.classList.toggle("studio-focus", studioFocus);
  var smallScreen = window.innerWidth <= 800;
  toggleBtn.textContent = navClosed ? (smallScreen ? "☰" : "»") : (smallScreen ? "☰" : "« Menu");
  toggleBtn.title = smallScreen ? (navClosed ? "Open the main menu" : "Close the main menu") : (navClosed ? "Expand the side menu" : "Collapse the side menu");
  toggleBtn.setAttribute("aria-label", smallScreen ? (navClosed ? "Open the main menu" : "Close the main menu") : "Collapse or expand the side menu");
  toggleBtn.setAttribute("aria-expanded", navClosed ? "false" : "true");
  dockBtn.setAttribute("aria-expanded", dockOpen ? "true" : "false");
  // Her presence bar repaints with every render (Build 8): the
  // persistent element keeps its identity; only its contents —
  // the live presence state and the pane button's pressed
  // state — are refreshed, on every view.
  renderPresenceBar();
  dockBtn.setAttribute("aria-label", dockOpen ? "Close the status dock" : "Open the status dock");
  syncNav();
  ensureLiveCommands();
  // While she is generating, the dashboard does no background
  // polling: the status fetch (and its llamafile probe on the
  // bridge) waits for the reply, so nothing competes with the
  // model for the machine's cores mid-answer.
  if (CONFIG.backend === "live" && !naviThinking && !(call && callState === "thinking")) { fetch("/api/status").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { bridgeStale = !!j && j.bridge !== BRIDGE_BUILD; pageStale = !!j && typeof j.buildSeq === "number" && j.buildSeq > SHIPPED_BUILD_SEQ; if (j && typeof j.buildSeq === "number") bridgeBuildSeq = j.buildSeq; publicReportLive = !!(j && j.publicServer); if (j && j.publicServer) publicState = j.publicServer; if (j && j.dhd) dhdState = j.dhd; if (j && j.mesh) meshState = j.mesh; if (j && j.sudo) { sudoState = j.sudo; CONFIG.sudo = !!j.sudo.enabled; } if (j && j.notifications) applyNotifications(j.notifications); if (j && Array.isArray(j.presenceExtra)) presenceExtra = j.presenceExtra; if (j && "openCase" in j) openCaseState = j.openCase; if (j && j.root) bridgeRoot = j.root; if (j && j.updaterChannel) { if (typeof j.updaterChannelExplicit === "boolean") updaterChannelExplicit = j.updaterChannelExplicit; if (j.updaterChannel !== updater.channel && updater.state !== "checking" && updater.state !== "applying") { updater.channel = j.updaterChannel; CONFIG.updaterChannel = j.updaterChannel; if (view === "Settings") render(); } } if (j && "update" in j) applyBackgroundUpdate(j.update); if (j && j.version) { CONFIG.version = j.version; if (j.versionStatus) CONFIG.versionStatus = j.versionStatus; var nv = document.getElementById("nav-version-val"); if (nv) nv.textContent = j.version + verTag(j.versionStatus); var nvv = document.getElementById("nav-version"); if (nvv) { nvv.setAttribute("aria-label", "Apex version " + j.version + (j.versionStatus && j.versionStatus !== "stable" ? " (" + j.versionStatus + ")" : "")); nvv.classList.remove("st-alpha", "st-edge", "st-beta", "st-stable"); nvv.classList.add("st-" + (j.versionStatus || "stable")); } } renderStatus(j); }).catch(function () { bridgeStale = false; pageStale = false; publicReportLive = false; renderStatus(null); }); }
  else renderStatus(null);
  var h = "";
  if (view === "Chat") {
    // Her presence lives in the persistent bar above every
    // view now (Build 8) — the Chat view carries no header of
    // its own; the thread simply starts below the bar.
    var cliMode = llama.status !== "running";
    // No header bar over the thread (build .52, issue #15): the
    // strip that read "Chat · <session> · navi.llamafile
    // connected · CLI-first chat" is gone entirely, at the
    // operator's ruling — the connection state already lives in
    // the status column and the mode in the composer's pills and
    // hint, so the thread simply starts at the top now.
    if (cliMode) h += "<div class='card' style='border-color:#f87171;color:#f87171'>Command-line mode — navi.llamafile is disconnected, so there's no model to answer. What you type runs as a shell command and you get the system output. Connect from the status column to bring the model back." + (CONFIG.sudo ? " Sudo mode is on — commands run as root." : "") + "</div>";
    // The reference shelf (v0.5.10e, #283): in a case chat the
    // shelf stands at the head of the room, labeled as
    // reference — visibly apart from the thread and from the
    // case's findings.
    h += caseShelfHtml((typeof activeChat === "function") ? activeChat() : null);
    // No Navi status line over the thread — the operator asked for
    // it gone. Her typing shows as the in-thread typing indicator,
    // and her presence line lives in the status column.
    h += "<div id='msgs'>";
    var lastYouIdx = -1;
    messages.forEach(function (mm, ii) { if (mm.who === "YOU") lastYouIdx = ii; });
    messages.forEach(function (m, idx) {
      var isCurrent = idx >= lastYouIdx;
      var foldKey = String(activeChatId) + ":" + idx;
      h += "<div class='msg'" + (m.shell ? " data-shellrec='1'" : "") + "><div class='msg-row'>" + msgAvatarHtml(m) + "<div class='msg-body'><div class='who'>" + msgWhoHtml(m) + "</div>";
      if (m.who === "YOU" && m.text) {
        var nxt = messages[idx + 1];
        var isCliYou = routeOfLine(m.text).kind === "cli" || (nxt && nxt.exec && (nxt.exec.command === m.text || nxt.exec.command === String(m.text).replace(/^\$\s*/, "")));
        h += isCliYou ? "<div class='mono'>" + cmdTokensHtml(String(m.text).replace(/^\$\s*/, "")) + "</div>" : "<div>" + esc(m.text) + "</div>";
        // Save to memory (build .81, #72): the tap-driven door
        // beside the spoken trigger — files this exact message
        // through the same secret screen and receipt as any
        // other chat save.
        h += "<div style='margin-top:4px'><button class='ghost' data-memsg='" + idx + "' aria-label='Save this message to memory' style='padding:1px 8px;font-size:11px;border-radius:999px'>Save to memory</button></div>";
      } else if (m.who === "NAVI" && m.text) {
        if (isBriefDoc(m.text)) {
          h += "<div class='brief-doc'>" + briefListHtml(m.text, true) + "</div>"; if (m.who === "NAVI") h += pillRow(m.mesh, m.sources);
        } else if (!isCurrent && isLongNavi(m.text)) {
          var openAttr = chatFoldState[foldKey] ? " open" : "";
          var lineCount = String(m.text).split("\n").length;
          h += "<details class='foldcard' data-fold='" + esc(foldKey) + "'" + openAttr + "><summary>Navi reply · " + esc(firstHintLine(m.text, true)) + " · " + lineCount + " lines · " + String(m.text).length + " chars</summary><div class='md'>" + mdToHtml(stripCite(m.text)) + "</div>" + pillRow(m.mesh, m.sources) + "</details>";
        } else { h += "<div class='md'>" + mdToHtml(stripCite(m.text)) + "</div>"; if (m.who === "NAVI") h += pillRow(m.mesh, m.sources); }
      } else if (m.text) h += "<div>" + esc(m.text) + "</div>";
      if (m.exec) {
        if (!isCurrent) {
          var info = execExitInfo(m.exec);
          var openAttr2 = chatFoldState[foldKey + ":exec"] ? " open" : "";
          h += "<details class='foldcard' data-fold='" + esc(foldKey + ":exec") + "'" + openAttr2 + "><summary>" + cmdTokensHtml(m.exec.command) + " · " + esc(info.label) + " · " + esc(firstHintLine(execOutputClean(m.exec), false)) + "</summary>" + execChatHtml(m.exec) + "</details>";
        } else h += execChatHtml(m.exec);
      }
      if (m.operation) h += operationCardHtml(m.operation);
      if (m.shell) h += shellEventsHtml(m.shell.events);
      if (m.meshFirst && m.mesh) h += "<div class='card' style='margin-top:6px;font-size:12px'><b>Apex | Mesh</b> — this reply was written by <b>" + esc(m.mesh.node || "a paired machine") + "</b>, a machine you paired. Borrowing is automatic for paired machines: hold a chat home with the route pill beside the composer set to “Local”, or turn borrowing off on the Mesh page.</div>";
      if (m.who === "NAVI" && !(m.mesh && m.mesh.servedBy === "mesh")) h += meshLineHtml(m.mesh);
      if (m.who === "NAVI") h += memoryReceiptHtml(m, idx);
      if (m.who === "NAVI") h += caseFilingsHtml(m);
      if (m.who === "NAVI") h += refPullsHtml(m);
      if (m.who === "NAVI") h += caseLogHtml(m);
      if (m.who === "NAVI") h += caseIntakeHtml(m);
      if (m.who === "NAVI") h += caseFiledReplyHtml(m);
      if (m.who === "NAVI") h += caseFileReplyBtnHtml(m, idx);
      if (m.who === "NAVI") h += caseStartedHtml(m);
      if (m.who === "NAVI") h += caseStartBtnHtml(m, idx);
      h += "</div></div></div>";
    });
    if (naviThinking) h += "<div class='msg' aria-live='polite'><div class='msg-row'>" + msgAvatarHtml({ who: "NAVI" }) + "<div class='msg-body'><div class='who'>Navi</div><div><span class='tdots'><i></i><i></i><i></i></span> <span style='color:var(--dim);font-size:12px'>thinking…</span></div></div></div></div>";
    h += "</div>";
    // No pill rows above the composer at all (0.1.9e, issue
    // #97): the quick-action suggestions went first, and now the
    // installed-chips pill shelf retires with them — chips run
    // from the keyboard instead (./<name> in chat, bare ./ for
    // the shelf), and their browsable home is ChipDeck. Clear
    // chat rides the hint row under the chat bar (build .52,
    // issue #13).
    // The composer dock (build .66): everything under the thread —
    // the bar, the toggle row — renders inside one .composer zone
    // so it reads as a single docked unit.
    h += shellPaneHtml();
    h += "<div class='composer'>";
    if (voiceMsg) h += "<div style='color:#f5b544;font-size:12px;margin:4px 0'>" + esc(voiceMsg) + "</div>";
    h += shellHeadHtml();
    // The call, in the chat window: a call started here (or in
    // the Voice view) runs in place — timer, state, push-to-talk,
    // and End above the composer, with the thread itself as the
    // live transcript and the composer talking to the call.
    if (call) {
      h += "<div class='card' style='margin-top:8px'><div style='display:flex;gap:14px;align-items:center;flex-wrap:wrap'><span class='wave " + (callState === "speaking" ? "speaking" : callState === "listening" ? "" : "idle") + "' aria-hidden='true'><i></i><i></i><i></i><i></i><i></i></span><div><b class='mono' id='call-clock' style='font-size:18px'>" + fmtCallTime(callSeconds) + "</b><br><span style='color:var(--dim);font-size:12px'>" + (callState === "thinking" ? "Navi is thinking…" : callState === "speaking" ? "Navi is speaking…" : "Listening — push to talk, or type below") + "</span></div><button class='callbtn end' id='call-end' style='width:60px;height:60px;margin-left:auto' aria-label='End the call'>End</button></div>" +
        "<div style='margin-top:8px;display:flex;gap:8px;flex-wrap:wrap;align-items:center'><button class='ptt' id='call-ptt' data-ptt='call' aria-label='Push to talk: click to start talking, click again to send'>Push to talk</button><span style='color:var(--dim);font-size:12px'>You're on a call — its transcript is this thread, and the composer below speaks to the call. Ending it files the call here, the same filing a call from Settings → Voice gets.</span></div></div>";
    } else if (callStarting) {
      h += "<div class='card' style='margin-top:8px;border-color:#f5b544;color:#f5b544'>Calling Navi — warming her voice…</div>";
    }
    if (pendingUpload) h += "<div style='margin-top:6px'><span class='srcpill'>Attached: " + esc(pendingUpload.name) + (pendingUpload.kind === "text" ? " — its contents ride with your next message" : " — image, stored for reference by name") + "</span> <button class='ghost' id='chat-upload-clear' aria-label='Remove the pending attachment' style='padding:1px 8px'>Remove</button></div>";
    // Clearing the chat is a command now (0.1.9e Build 11,
    // issue #133): the Clear chat button that rode this row
    // since build .52 (issue #13) is retired — /cls, typed in
    // the composer and intercepted in sendChat(), does its
    // work identically, and the row keeps the same members in
    // an empty thread as in a full one. What remains of the
    // old construction is the history note the row carries.
    var clearChatBtn = "<span id='hist-note'></span>";
    // Navi Mesh (build .85's escape, v0.4.1e #228's control):
    // the route pill rides the hint row beside the mode latch —
    // one pill, three roads, its face rendered from the
    // bridge's stored route for this chat (routePillHtml).
    var meshLocalBtn = routePillHtml();
    h += "<div class='row'><span class='grow ac-wrap" + (chatMode === "cli" ? " cli-mode" : "") + "'><input id='chat-in' type='text' placeholder='" + chatPlaceholder() + "' aria-label='Chat message'></span>" + (CONFIG.backend === "live" ? "<button class='ghost' id='chat-newcase' aria-label='Start a new case from this chat' title='New case — Navi swipes this conversation into a case'>" + caseSvg() + "<span class='call-label'>New case</span></button>" : "") + "<button class='ghost' id='chat-upload' aria-label='Attach a picture or a text file to this chat' title='Attach a file'>" + uploadSvg() + "</button><input id='chat-file' type='file' accept='image/*,.txt,.md,.json,.csv,.log,.py,.js,.ts,.sh,text/*' style='display:none' aria-label='Choose a file to attach'>" + (CONFIG.backend === "live" ? "<button class='ghost' id='chat-mic' data-ptt='chat' aria-label='Push to talk: click to start talking, click again to send it to Navi' title='Push to talk'>" + micSvg() + "</button>" : "") + (!call && !callStarting ? "<button class='ghost' id='chat-call' aria-label='Call Navi — start a voice call here in the chat'>" + phoneSvg() + "<span class='call-label'>Call</span></button>" : "") + "<button class='ghost' id='chat-tools' aria-label='Chat tools' title='Chat tools' aria-haspopup='true' aria-expanded='" + (chatToolsOpen ? "true" : "false") + "'>☰</button></div>" + toolsMenuHtml() +
    (call ? "<div class='modehint' id='chat-hint'><span>On a call — Enter sends what you type to the call. Attachments wait until the call ends.</span>" + clearChatBtn + "</div>"
      : cliMode ? "<div class='modehint' id='chat-hint'><span class='modegroup' role='group' aria-label='Composer mode'><button type='button' data-mode='cli' aria-pressed='false' aria-label='CLI mode — press once to latch: lines type into this chat's shell session until switched'>CLI</button><button type='button' data-mode='navi' aria-pressed='true' aria-label='Navi mode — press once to latch: lines go to Navi until switched'>Navi</button></span><span class='press-hint'>| Press enter to run.</span>" + meshLocalBtn + clearChatBtn + "</div>"
      : "<div class='modehint' id='chat-hint'><span class='modegroup' role='group' aria-label='Composer mode'><button type='button' data-mode='cli' aria-pressed='false' aria-label='CLI mode — press once to latch: lines type into this chat's shell session until switched'>CLI</button><button type='button' data-mode='navi' aria-pressed='true' aria-label='Navi mode — press once to latch: lines go to Navi until switched'>Navi</button></span><span class='press-hint'>| Press enter to send.</span>" + meshLocalBtn + clearChatBtn + "</div>") + "</div>";
    // New case from chat (v0.4.5e, #241): the form stands over
    // the chat it swipes — a dialog, never a page.
    if (newCaseState) h += newCaseFormHtml();
  } else if (view === "Chips") {
    h += "<h3>Chips</h3>";
    h += "<div class='tabbar' role='tablist' aria-label='Chips sections' style='margin-bottom:10px'><button class='" + (chipsTab === "management" ? "act" : "ghost") + "' data-ctab='management' aria-label='Navi User Repo tab'>Navi User Repo (NUR)</button><button class='" + (chipsTab === "studio" ? "act" : "ghost") + "' data-ctab='studio' aria-label='Chip studio tab'>Chip Studio</button><button class='" + (chipsTab === "workflows" ? "act" : "ghost") + "' data-ctab='workflows' aria-label='Chip workflows tab'>Workflows</button></div>";
    if (chipsTab === "studio") {
      var d = studioDraft || (studioDraft = defaultStudioDraft());
      h += "<div class='studio-root'><div class='studio-head'>";
      h += "<p style='color:var(--dim)'>Write a chip in a real code editor — Python, the way the published navi-chips are written. Saving packs the whole thing into a .png chip — code inside the image — in the chips folder, ready to run from the folder, from chat (<span class='mono'>/chip &lt;name&gt;</span>), and chained in Workflows.</p>";
      if (studioMsg) h += "<p>" + esc(studioMsg) + "</p>";
      if (chipMsg) h += "<p>" + esc(chipMsg) + "</p>";
      if (CONFIG.backend === "live") {
        h += "<div class='row'><select id='st-open' class='grow' aria-label='Open a chip from the chips folder'><option value=''>Open a chip from the folder…</option>" + (wfData ? wfData.chips.map(function (c) { return "<option value='" + esc(c.file) + "'" + (d.file === c.file ? " selected" : "") + ">" + esc(c.name) + " (" + esc(c.file) + ")" + (c.locallyEdited ? " · edited locally" : "") + "</option>"; }).join("") : "") + "</select><button class='ghost' id='st-open-btn'>Open in editor</button><button class='ghost' id='st-new'>New chip</button></div>";
        var stChip = wfData ? (wfData.chips.filter(function (c) { return c.file === d.file; })[0] || null) : null;
        if (stChip && stChip.sourceKey) h += "<p class='mono' style='color:" + (stChip.locallyEdited ? "#f5b544" : "var(--dim)") + ";font-size:11px;margin-top:6px'>INSTALLED FROM " + esc(stChip.sourceKey) + (stChip.locallyEdited ? " · EDITED LOCALLY — this copy no longer matches the published source; export it from Navi User Repo (NUR) to publish your version" : " · saving edits re-forges the chip here and marks the chip edited locally") + "</p>";
      } else {
        h += "<div class='card'>Mock mode: the editor and templates all work — forging and saving chips happens through the live bridge (switch the backend to Live in Settings).</div>";
      }
      h += "<div class='row' style='margin-top:8px'><input class='grow' id='st-name' type='text' placeholder='Chip name — e.g. Port Sweep' aria-label='Chip name' value='" + esc(d.name) + "'></div>";
      h += "<div class='row' style='margin-top:8px'><input class='grow' id='st-desc' type='text' placeholder='Description — what the chip does, in one line' aria-label='Chip description' value='" + esc(d.description) + "'></div>";
      // Build 13 (#146): the face art renders below, as the collapsible strip (.studio-face) — not as a head row.
      h += "<div class='row' style='margin-top:8px'><select id='st-template' class='grow' aria-label='Apply a chip template'><option value=''>Apply a template…</option>" + CHIP_TEMPLATES.map(function (t) { return "<option value='" + t.id + "'>" + esc(t.label) + "</option>"; }).join("") + "</select></div>";
      h += "</div>"; // .studio-head
      h += "<div class='studio-face" + (studioFaceOpen ? " open" : "") + "'><button class='studio-face-toggle' id='st-face-toggle' aria-expanded='" + (studioFaceOpen ? "true" : "false") + "'>Face art — the chip's picture, a 20×20 grid</button><div class='row studio-face-row' style='margin-top:8px'><textarea class='grow mono' id='st-icon' rows='6' placeholder='Face art (optional) — 20 rows of 20 characters: # accent, o shade, . empty. Your art outranks the built-in glyph.' aria-label='Chip face art, 20 rows of 20 characters'>" + esc(d.icon || "") + "</textarea></div></div>";
      h += "<div class='studio-switch' role='tablist' aria-label='Studio panes — one at a time on a phone'><button class='" + (studioFaceOpen ? "act" : "ghost") + "' data-stpane='face' aria-pressed='" + studioFaceOpen + "'>Face</button><button class='" + (studioPane === "code" ? "act" : "ghost") + "' data-stpane='code' aria-pressed='" + (studioPane === "code") + "'>Code</button><button class='" + (studioPane === "test" ? "act" : "ghost") + "' data-stpane='test' aria-pressed='" + (studioPane === "test") + "'>Test</button></div>";
      h += "<div class='studio-panes'><div class='studio-pane studio-pane-code" + (studioPane === "code" ? " act" : "") + "' data-spane='code'>";
      h += "<div class='studio-snips' aria-label='Snippet shelf — tap to insert at the caret'>" + [["run", "def run()"], ["import", "import navi_internal"], ["header", "command / use"], ["print", "print_message()"], ["llm", "llm_chat()"], ["user", "get_user()"]].map(function (sn) { return "<button class='ghost' data-stsnip='" + sn[0] + "'>" + sn[1] + "</button>"; }).join("") + "</div>";
      var edLines = d.code.split("\n");
      var edSymbols = [];
      edLines.forEach(function (ln, i) { var m = /^(\s*)(def|class)\s+(\w+)/.exec(ln); if (m) edSymbols.push({ line: i + 1, label: (m[1] ? "· " : "") + m[2] + " " + m[3] }); });
      var filesOf = studioFilesOf(d);
      var rootCode = studioRootCode(d);
      h += "<div style='display:flex;gap:10px;align-items:flex-start;margin-top:8px;flex-wrap:wrap'>"
        + "<div class='card' style='margin:0;flex:0 0 205px;min-width:180px'><b class='mono' style='font-size:10px;letter-spacing:.16em;color:var(--dim)'>FILES · ROOT.PY RUNS FIRST</b>"
        + filesOf.map(function (f) {
            var depth = f.path.split("/").length - 1;
            return "<div style='display:flex;align-items:center;gap:2px;margin-top:2px;padding-left:" + depth * 12 + "px'>"
              + "<button class='ghost' style='flex:1;text-align:left;" + (d.active === f.path ? "border-color:var(--cy);color:var(--cy)" : "") + "' data-stfile='" + esc(f.path) + "' aria-label='Open file " + esc(f.path) + "'>" + (studioFileDirty(d, f.path) ? "● " : "") + esc(f.path.split("/").pop()) + "</button>"
              + (f.path !== "root.py" ? "<button class='ghost' data-strename='" + esc(f.path) + "' aria-label='Rename file " + esc(f.path) + "'>✎</button><button class='ghost' data-stdelete='" + esc(f.path) + "' aria-label='Delete file " + esc(f.path) + "' style='color:#f87171'>✕</button>" : "")
              + "</div>"
              + (studioRename === f.path ? "<div class='row' style='margin-top:2px'><input class='grow' id='st-rename-in' type='text' value='" + esc(studioRenameText) + "' aria-label='New name for " + esc(f.path) + "'><button class='ghost' id='st-rename-go'>Rename</button></div>" : "");
          }).join("")
        + "<div class='row' style='margin-top:6px'><input class='grow' id='st-newfile' type='text' placeholder='helpers.py' aria-label='New file path'><button class='ghost' id='st-addfile'>Add file</button></div>"
        + "<p style='color:var(--dim);font-size:11px;margin:6px 0 0'>Other files are modules: <span class='mono'>import helpers</span> in root.py. They travel with the chip — its picture, the GitHub export, and installs.</p></div>"
        + "<div style='flex:1;min-width:300px'>"
        + "<div style='display:flex;gap:4px;flex-wrap:wrap'>" + filesOf.map(function (f) { return "<button class='" + (d.active === f.path ? "act" : "ghost") + "' data-stfile='" + esc(f.path) + "' aria-label='Switch to file " + esc(f.path) + "'>" + (studioFileDirty(d, f.path) ? "● " : "") + esc(f.path) + "</button>"; }).join("") + "</div>"
        + "<div class='ed-wrap' style='margin-top:6px'><div class='ed-bar'>"
        + "<button class='ghost' id='ed-undo' aria-label='Undo'>↩ Undo</button><button class='ghost' id='ed-redo' aria-label='Redo'>↪ Redo</button>"
        + "<input id='ed-find' type='text' placeholder='Find in code' aria-label='Find in code' value='" + esc(studioFind) + "' style='width:130px'>"
        + "<button class='ghost' id='ed-find-next' aria-label='Find next match'>↓ Next</button>"
        + "<select id='ed-outline' aria-label='Outline — jump to a function or class' style='max-width:170px'><option value=''>Outline…</option>" + edSymbols.map(function (s) { return "<option value='" + s.line + "'>" + esc(s.label) + " · " + s.line + "</option>"; }).join("") + "</select>"
        + "<input id='ed-goto' type='text' placeholder='Line #' aria-label='Go to line number' style='width:64px'><button class='ghost' id='ed-goto-btn' aria-label='Go to the typed line'>Go</button>"
        + "<button class='ghost' id='ed-zoom-out' aria-label='Zoom the editor out'>A−</button><button class='ghost' id='ed-zoom-in' aria-label='Zoom the editor in'>A+</button>"
        + "<button class='ghost' id='ed-wrap' aria-label='Toggle word wrap'>" + (studioWrap ? "Wrap on" : "Wrap off") + "</button>"
        + "<button class='ghost' id='ed-pal' aria-label='Open the editor command palette'>⌘ Commands</button>"
        + "<button class='ghost' id='ed-fs' aria-pressed='" + studioFocus + "' aria-label='" + (studioFocus ? "Exit focus mode" : "Enter focus mode — the app's one expand idiom") + "'>" + (studioFocus ? "⤡ Exit focus" : "⤢ Focus") + "</button>"
        + "<span class='mono' id='ed-status' style='color:var(--dim);font-size:11px;margin-left:auto'></span></div>"
        + (studioPal ? "<div class='ed-pal'><input id='ed-pal-q' type='text' placeholder='Type a command…' aria-label='Command palette — type to filter commands' value='" + esc(studioPalQuery) + "' style='width:100%'>" + STUDIO_COMMANDS.filter(function (c) { return !studioPalQuery || c[1].toLowerCase().indexOf(studioPalQuery.toLowerCase()) >= 0; }).map(function (c) { return "<button class='ghost' style='display:block;width:100%;text-align:left;margin-top:4px' data-pal='" + c[0] + "'>" + esc(c[1]) + "</button>"; }).join("") + "</div>" : "")
        + "<div class='ed-scroll'><div class='ed-inner'><div class='ed-gutter' id='ed-gutter'>" + edLines.map(function (ln, i) {
            var btn = "";
            if (studioIsFoldLine(ln)) btn = "<button data-unfold='" + i + "' aria-label='Unfold the block at line " + (i + 1) + "'>▸</button>";
            else if (studioBlockStart(edLines, i) > i) btn = "<button data-fold='" + i + "' aria-label='Fold the block at line " + (i + 1) + "'>▾</button>";
            return "<div>" + btn + (i + 1) + "</div>";
          }).join("") + "</div>"
        + "<div class='ed-code'><div class='ed-guides' id='ed-guides' aria-hidden='true'></div><div class='ed-squigs' id='ed-squigs' aria-hidden='true'></div><div class='ed-extras' id='ed-extras' aria-hidden='true'></div><pre id='ed-pre' aria-hidden='true' style='font-size:" + studioZoom + "px" + (studioWrap ? ";white-space:pre-wrap" : "") + "'>" + highlightPy(d.code) + "\n</pre><textarea id='st-code' aria-label='Chip code editor — " + esc(d.active) + "' spellcheck='false' wrap='" + (studioWrap ? "soft" : "off") + "' style='font-size:" + studioZoom + "px" + (studioWrap ? ";white-space:pre-wrap" : "") + "'>" + esc(d.code) + "</textarea></div>"
        + "<div class='ed-mini' id='ed-mini' aria-hidden='true'><pre id='ed-mini-pre'>" + esc(d.code) + "</pre></div></div></div></div>"
        + "<div class='row' style='margin-top:6px'><input class='grow' id='st-findall-q' type='text' placeholder='Find in all files of this chip' aria-label='Find text in all files of this chip' value='" + esc(studioFindAll) + "'><button class='ghost' id='st-findall-btn'>Find in files</button></div>"
        + (studioFindAllResults ? (studioFindAllResults.length ? "<div style='margin-top:4px'>" + studioFindAllResults.map(function (r) { return "<button class='ghost' style='display:block;width:100%;text-align:left;margin-top:2px' data-findall-jump='" + esc(r.path) + "|" + r.line + "' aria-label='Go to " + esc(r.path) + " line " + r.line + "'><span class='mono' style='color:var(--cy)'>" + esc(r.path) + ":" + r.line + "</span> " + esc(r.text) + "</button>"; }).join("") + "</div>" : "<p style='color:var(--dim);font-size:12px'>No matches in this chip's files.</p>") : "")
        + "</div></div>";
      var chk = [["imports navi_internal", /^\s*import\s+navi_internal/m.test(rootCode)], ["declares command", /^\s*command\s*[:=]/m.test(rootCode)], ["declares use (one-line description)", /^\s*use\s*[:=]/m.test(rootCode)], ["defines run(arguments) — the entry point Navi calls", /^\s*def\s+run\s*\(/m.test(rootCode)]];
      h += "</div><div class='studio-pane studio-pane-test" + (studioPane === "test" ? " act" : "") + "' data-spane='test'>";
      h += "<div class='card' style='margin-top:8px'><b class='mono' style='font-size:11px;letter-spacing:.14em;color:var(--dim)'>STRUCTURE CHECK — THE SHAPE NAVI LOADS</b><br>" + chk.map(function (c) { return "<span class='mono' style='font-size:11px;color:" + (c[1] ? "#34d399" : "#f87171") + "'>" + (c[1] ? "✓" : "✗") + " " + c[0] + "</span>"; }).join(" · ") + "</div>";
      h += "<textarea id='st-output' rows='3' class='mono' placeholder='Declared run output — shown when the chip runs and its code defines no run(); in Workflows this output flows into the next chip (optional)' aria-label='Declared run output' style='margin-top:8px'>" + esc(d.output) + "</textarea>";
      if (studioRunOut) h += "<pre class='out mono studio-run-out' id='st-run-out' style='font-size:11px;white-space:pre-wrap;margin-top:8px'>" + esc(studioRunOut) + "</pre>";
      h += "<button class='act' id='st-forge' style='margin-top:8px'>" + (d.file ? "Save changes · re-forge chip" : "Forge chip") + "</button>";
      if (studioMade) h += "<div class='card studio-made' style='margin-top:10px'>Forged <b>" + esc(studioMade) + "</b> — find it on the Local Chips tab of Navi User Repo (NUR), under Forged. <a class='ghost' href='/api/chips/file?name=" + encodeURIComponent(studioMade) + "' download='" + esc(studioMade) + "'>Download the chip .png</a></div>";
      h += "</div>"; // .studio-pane-test
      h += "<div class='studio-tail'><h3 style='margin-top:12px'>The chip API, in full</h3><div class='api-m'><span style='color:var(--dim);font-size:12px'>Every function a chip can call — what it takes, what it gives back, and when to reach for it — is written up on the wiki's <b>Navi Chip Functions</b> page. Open it beside the editor and read while you write:</span><br><button class='ghost' id='studio-api-docs' style='margin-top:4px' aria-label='Open the documentation at Navi Chip Functions page'>Open the chip API docs ⧉</button></div>";
      h += "<h3 style='margin-top:16px'>Publish it to GitHub</h3><ol style='color:var(--dim)'><li>Forge the chip here.</li><li>On its card under Navi User Repo (NUR) → Local Chips, press <b>Edit README</b> if you want your own words on its GitHub page — your edited README is kept inside the chip and ships in the export and the upload alike (Restore generated README undoes it).</li><li>Signed in to GitHub? Press <b>Publish to GitHub</b> on the chip's card, check the repo name (pre-filled from the chip's name, editable), and confirm: Navi creates the public repo if it is new, sets the <span class='mono'>navi-chips</span> topic on its own, and uploads root.py and every module file at the root, a navi-chip.json manifest, the chip .png, and the README. An existing repo is updated, and she says so.</li><li>Not signed in? Use <b>Export for GitHub</b> instead: it builds the same file set as a repo-ready ZIP — create a public GitHub repo and unzip the export at its root, then add the <span class='mono'>navi-chips</span> topic yourself.</li><li>With the ZIP route, the last step is yours: publish the release on GitHub — the version tag is created with that publish, so use the version in the manifest. With Publish to GitHub there is no last step: the release is published for you, picture attached. A newer release later offers every install an update, and installs download the whole file set.</li><li>Navi User Repo (NUR) finds it on the next search.</li></ol></div>";
      h += "</div>"; // .studio-panes
      h += "<div class='studio-ac' id='st-ac-row' aria-label='Autocomplete suggestions'></div>";
      h += "<div class='studio-actions'><button class='act' id='st-act-run'" + (d.file ? "" : " disabled title='Forge the chip first — Run tests the forged chip'") + ">Run</button><button class='" + (d.file ? "ghost" : "act") + "' id='st-act-forge'" + (d.file ? " disabled title='Already forged — Save re-forges your changes'" : "") + ">Forge</button><button class='" + (d.file ? "act" : "ghost") + "' id='st-act-save'" + (d.file ? "" : " disabled title='Nothing to save yet — forge the chip first'") + ">Save</button></div>";
      h += "</div>"; // .studio-root
    } else if (chipsTab === "workflows") {
      h += "<p style='color:var(--dim)'>A workflow chains chips together: the first chip runs, its output becomes the next chip's input, and so on — the last chip's output is the result.</p>";
      if (chipMsg) h += "<p>" + esc(chipMsg) + "</p>";
      var wfNameOf = function (key) {
        if (CONFIG.backend === "live") { var hit = (wfData ? wfData.chips : []).filter(function (c) { return c.file === key; })[0]; return hit ? hit.name : key; }
        var def = CHIP_DEFS.filter(function (c) { return c.id === key; })[0]; return def ? def.name : key;
      };
      var wfResult = function (result) {
        if (!result) return "";
        var r = "";
        if (result.error) r += "<p style='color:#f87171'>" + esc(result.error) + "</p>";
        r += result.steps.map(function (s, i) {
          return (i > 0 ? "<p class='mono' style='text-align:center;color:var(--cy);margin:2px 0'>↓ output flows down ↓</p>" : "") +
            "<div class='wf-step'><b class='mono' style='font-size:11px'>STEP " + (i + 1) + " · " + esc(s.name) + "</b>" +
            (s.input ? "<div class='mono' style='color:var(--dim);font-size:10px;margin-top:4px'>IN · FROM STEP " + i + "</div><pre class='out mono' style='font-size:11px'>" + esc(s.input) + "</pre>" : "<div class='mono' style='color:var(--dim);font-size:10px;margin-top:4px'>IN · — (first step)</div>") +
            "<div class='mono' style='color:var(--dim);font-size:10px;margin-top:4px'>OUT</div><pre class='out mono' style='font-size:11px'>" + esc(s.output) + "</pre></div>";
        }).join("");
        if (result.finalOutput) r += "<div class='card' style='border-color:#34d399'><b class='mono' style='font-size:10px;letter-spacing:.16em;color:var(--dim)'>FINAL OUTPUT</b><pre class='mono' style='white-space:pre-wrap;margin:4px 0 0'>" + esc(result.finalOutput) + "</pre></div>";
        return r;
      };
      if (CONFIG.backend !== "live") {
        h += "<div class='card'><b>New workflow</b><div class='row' style='margin-top:8px'><input class='grow' id='wf-name' type='text' placeholder='Workflow name — e.g. Morning sweep' aria-label='Workflow name'></div>";
        h += "<div class='row' style='margin-top:8px'><input class='grow' id='wf-desc' type='text' placeholder='Description (optional)' aria-label='Workflow description'></div>";
        h += "<div style='margin-top:8px'>" + (mockWfSteps.length ? mockWfSteps.map(function (sid, i) { return "<div class='wf-step'><b class='mono' style='font-size:11px'>" + (i + 1) + ". " + esc(wfNameOf(sid)) + "</b> <span style='color:var(--dim);font-size:11px'>" + (i === 0 ? "runs first" : "← takes step " + i + "'s output") + "</span> <button class='ghost' data-wf-up='" + i + "'>↑</button> <button class='ghost' data-wf-down='" + i + "'>↓</button> <button class='ghost' data-wf-rm='" + i + "'>Remove</button></div>"; }).join("") : "<p style='color:var(--dim)'>No steps yet — add chips below. A workflow needs at least two.</p>") + "</div>";
        h += "<div class='row'><select id='wf-add' class='grow' aria-label='Choose a chip to add as a workflow step'><option value=''>Pick a chip…</option>" + CHIP_DEFS.map(function (c) { return "<option value='" + c.id + "'>" + esc(c.name) + "</option>"; }).join("") + "</select><button class='ghost' id='wf-add-btn'>Add step</button></div>";
        h += "<button class='act' id='wf-save' style='margin-top:8px'>Save workflow</button></div>";
        h += "<h3 style='margin-top:16px'>Saved workflows</h3>";
        h += mockWorkflows.length ? mockWorkflows.map(function (w) {
          return "<div class='card'><b>" + esc(w.name) + "</b>" + (w.description ? " <span style='color:var(--dim)'>" + esc(w.description) + "</span>" : "") + " <button class='act' data-wfrun='" + w.id + "'>Run workflow</button> <button class='ghost' data-wfdel='" + w.id + "'>Delete</button><br><span class='mono' style='font-size:11px;color:var(--dim)'>" + w.steps.map(function (sid) { return esc(wfNameOf(sid)); }).join(" → ") + "</span>" + wfResult(mockWfRuns[w.id]) + "</div>";
        }).join("") : "<p style='color:var(--dim)'>None yet — build the first chain above. (Mock mode keeps workflows in this browser; Live mode saves them on the machine, beside the chips.)</p>";
      } else {
        h += "<div class='card'><b>New workflow</b><div class='row' style='margin-top:8px'><input class='grow' id='wf-name' type='text' placeholder='Workflow name — e.g. Morning sweep' aria-label='Workflow name'></div>";
        h += "<div class='row' style='margin-top:8px'><input class='grow' id='wf-desc' type='text' placeholder='Description (optional)' aria-label='Workflow description'></div>";
        h += "<div style='margin-top:8px'>" + (wfDraftSteps.length ? wfDraftSteps.map(function (fn, i) { return "<div class='wf-step'><b class='mono' style='font-size:11px'>" + (i + 1) + ". " + esc(wfNameOf(fn)) + "</b> <span style='color:var(--dim);font-size:11px'>" + (i === 0 ? "runs first" : "← takes step " + i + "'s output") + "</span> <button class='ghost' data-wf-up='" + i + "'>↑</button> <button class='ghost' data-wf-down='" + i + "'>↓</button> <button class='ghost' data-wf-rm='" + i + "'>Remove</button></div>"; }).join("") : "<p style='color:var(--dim)'>No steps yet — add chips below. A workflow needs at least two.</p>") + "</div>";
        h += "<div class='row'><select id='wf-add' class='grow' aria-label='Choose a chip to add as a workflow step'><option value=''>Pick a chip…</option>" + (wfData ? wfData.chips.map(function (c) { return "<option value='" + esc(c.file) + "'>" + esc(c.name) + " (" + esc(c.file) + ")</option>"; }).join("") : "") + "</select><button class='ghost' id='wf-add-btn'>Add step</button></div>";
        h += "<button class='act' id='wf-save' style='margin-top:8px'>Save workflow</button></div>";
        h += "<h3 style='margin-top:16px'>Saved workflows</h3>";
        if (!wfData) h += "<p style='color:var(--dim)'>Loading workflows from the bridge…</p>";
        else h += wfData.workflows.length ? wfData.workflows.map(function (w) {
          return "<div class='card'><b>" + esc(w.name) + "</b>" + (w.description ? " <span style='color:var(--dim)'>" + esc(w.description) + "</span>" : "") + " <button class='act' data-wfrun='" + w.id + "'>Run workflow</button> <button class='ghost' data-wfdel='" + w.id + "'>Delete</button><br><span class='mono' style='font-size:11px;color:var(--dim)'>" + w.steps.map(function (fn) { return esc(wfNameOf(fn)); }).join(" → ") + "</span>" + wfResult(wfRuns[w.id]) + "</div>";
        }).join("") : "<p style='color:var(--dim)'>None yet — build the first chain above.</p>";
      }
    } else if (CONFIG.backend !== "live") {
      h += "<div class='card'>GitHub chip management runs through the live bridge — switch the backend to Live in Settings and this tab searches GitHub for the navi-chips topic, with install, delete, and update. Mock mode's shelf is empty too: Navi ships with no pre-installed chips in any mode.</div>";
    } else {
      h += "<div class='row'><b>Powered by GitHub · topic: navi-chips</b><span style='color:var(--dim)'>" + (chipCatalog && chipCatalog.fetchedAt ? "searched " + esc(String(chipCatalog.fetchedAt).slice(0, 10)) : chipCatalogBusy ? "searching…" : "not searched yet") + "</span><button class='ghost' id='cat-refresh'>Search NUR</button></div>";
      h += "<p style='color:var(--dim)'>Every public repo tagged navi-chips is a Navi chip. Install one to track it here; when the repo cuts a newer release, the chip offers an update.</p>";
      if (chipMsg) h += "<p>" + esc(chipMsg) + "</p>";
      var nurAll = (chipCatalog && chipCatalog.chips ? chipCatalog.chips : []).filter(function (c) { return c.inCatalog; });
      var nurSaintsN = nurAll.filter(nurIsSaints).length;
      h += "<div class='tabbar' role='tablist' aria-label='NUR chip sources' style='margin:10px 0 8px'><button class='" + (nurTab === "saintssec" ? "act" : "ghost") + "' data-nurtab='saintssec' aria-label='Saints Sec Chips tab'>Saints Sec Chips (" + nurSaintsN + ")</button><button class='" + (nurTab === "community" ? "act" : "ghost") + "' data-nurtab='community' aria-label='Community Chips tab'>Community Chips (" + (nurAll.length - nurSaintsN) + ")</button><button class='" + (nurTab === "local" ? "act" : "ghost") + "' data-nurtab='local' aria-label='Local Chips tab'>Local Chips (" + (wfData ? wfData.chips.length : 0) + ")</button></div>";
      if (nurTab === "community") {
        h += "<div class='row' style='margin-bottom:4px'><label for='nur-search' style='color:var(--dim);font-size:12px'>SEARCH NUR</label><input class='grow' id='nur-search' type='search' value='" + esc(nurQuery) + "' placeholder='Filter community chips by name or description…' aria-label='Search NUR community chips'><button class='ghost' id='nur-clear' style='" + (nurQuery ? "" : "display:none") + "'>Clear</button></div>";
        h += "<p class='mono' id='nur-meta' style='color:var(--dim);font-size:11px;margin:2px 0 6px'>" + esc(nurMetaText()) + "</p>";
      }
      if (nurTab === "local") h += "<div id='nur-list'>" + nurCardsHtml() + "</div>";
      else if (chipCatalogBusy) h += "<p style='color:var(--dim)'>Searching GitHub for navi-chips…</p>";
      else if (chipCatalog && chipCatalog.error) h += "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(chipCatalog.error) + "</div>";
      else if (chipCatalog) {
        if (chipCatalog.warning) h += "<div class='card' style='border-color:#f5b544;color:#f5b544'>" + esc(chipCatalog.warning) + "</div>";
        h += "<div id='nur-list'>" + nurCardsHtml() + "</div>";
      }
    }
  } else if (view === "Vault") {
    h += vaultHtml();
  } else if (view === "CaseFiles") {
    h += caseFilesHtml();
  } else if (view === "ThreatIntel") {
    h += threatIntelHtml();
  } else if (view === "CTFBoard") {
    h += threatIntelHtml();
  } else if (view === "DHD") {
    h += dhdHtml();
  } else if (view === "Mesh") {
    // The Mesh page (0.2.0e Build 10, #196): one calm column —
    // posture sentence, Your door, Lending & borrowing, one
    // card per peer, Roads, Health at the foot (meshPageHtml).
    h += meshPageHtml();
  } else if (view === "Federation") {
    h += federationHtml();
  } else if (view === "FediChat") {
    h += fedichatHtml();
  } else if (view === "Seraph") {
    h += seraphHtml();
  } else if (view === "Dossier") {
    h += dossierHtml();
  } else if (view === "RAG") {
    h += "<h3>RAG · knowledge packs</h3>";
    h += "<div class='tabbar' role='tablist' aria-label='RAG sections' style='margin-bottom:10px'><button class='" + (ragTab === "packs" ? "act" : "ghost") + "' data-ragtab='packs' aria-label='Knowledge packs tab'>Packs</button><button class='" + (ragTab === "studio" ? "act" : "ghost") + "' data-ragtab='studio' aria-label='RAG Studio tab'>RAG Studio</button></div>";
    if (CONFIG.backend === "live" && !ragPacksState && !ragPacksFetching) refreshRagPacks(false);
    if (ragTab === "studio") {
      h += "<p style='color:var(--dim)'>Package documents into a knowledge pack: drag .md/.txt/.pdf files into the drop zone (or click it to upload) — or name a folder on this machine — name and describe the pack, and the Studio lays it down in <span class='mono'>rag-packs/</span> with its documents, its built index, and its metadata, active and ready to use immediately. A PDF contributes its extracted text — the binary itself is never stored. Her answers can draw on it the moment it lands; toggle it off here in the list or in Packs whenever you like.</p>";
      if (CONFIG.backend !== "live") h += "<div class='card'>RAG Studio packages through the live bridge — switch the backend to Live in Settings. The form below is the shape of it.</div>";
      if (ragForgeMsg) h += "<p>" + esc(ragForgeMsg) + "</p>";
      if (ragEdit) h += "<div class='card' style='border-color:var(--cy)'><b>Editing · " + esc(ragEditPrefill ? ragEditPrefill.name : ragEdit.slug) + "</b>" + (ragEdit.sourceKey ? " <span class='mono' style='color:#f5b544;font-size:11px'>INSTALLED FROM " + esc(ragEdit.sourceKey) + " — SAVING MARKS IT EDITED LOCALLY</span>" : "") + "<br><span style='color:var(--dim);font-size:12px'>Remove documents, add more below, adjust the name and description — saving writes the new set into this pack's folder and rebuilds its index. Export it from the Packs list to publish your version.</span> <button class='ghost' id='rf-new' aria-label='Stop editing and start a new pack'>New pack instead</button></div>";
      h += "<div class='row'><input class='grow' id='rf-name' type='text' placeholder='Pack name — e.g. Home Notes' aria-label='Pack name'" + (ragEditPrefill ? " value='" + esc(ragEditPrefill.name) + "'" : "") + "></div>";
      h += "<div class='row' style='margin-top:8px'><input class='grow' id='rf-desc' type='text' placeholder='Description — what this pack knows, in one line' aria-label='Pack description'" + (ragEditPrefill ? " value='" + esc(ragEditPrefill.description) + "'" : "") + "></div>";
      h += "<div class='row' style='margin-top:8px'><textarea class='grow mono' id='rf-icon' rows='6' placeholder='Cartridge art (optional) — 20 rows of 20 characters: # accent, o shade, . empty. Your art outranks the book glyph.' aria-label='Pack cartridge art, 20 rows of 20 characters'>" + esc(ragEditPrefill && ragEditPrefill.icon ? ragEditPrefill.icon : "") + "</textarea></div>";
      h += "<div id='rf-drop' tabindex='0' role='button' aria-label='Drop documents here or activate to upload' style='margin-top:10px;border:1px dashed " + (ragDragOver ? "var(--accent)" : "var(--line)") + ";border-radius:8px;padding:18px 12px;text-align:center;cursor:pointer;background:" + (ragDragOver ? "rgba(88,166,255,.12)" : "transparent") + "'><b>" + (ragDragOver ? "Drop the documents to add them" : "Drag documents here") + "</b><br><span style='color:var(--dim);font-size:12px'>.md / .txt / .pdf, as many as you like — or click to upload</span><input id='rf-files' type='file' accept='.md,.txt,.pdf,text/markdown,text/plain,application/pdf' multiple style='display:none' aria-label='Upload documents'></div>";
      if (ragForgeDocs.length && ragEdit) h += ragForgeDocs.map(function (d, i) { return "<div class='card' style='margin-top:6px;padding:8px'><div class='row'><b class='mono grow' style='font-size:12px'>" + esc(d.title) + "</b><span style='color:var(--dim);font-size:11px'>" + (d.pdf_base64 ? "PDF — text extracted on save" : (d.body ? d.body.length + " chars" : "")) + "</span><button class='ghost' data-rfremove='" + i + "' aria-label='Remove document " + esc(d.title) + "'>Remove</button></div>" + (d.pdf_base64 ? "" : "<textarea class='rf-doc-body mono' data-rfdoc='" + i + "' rows='4' style='margin-top:6px;font-size:12px' aria-label='Document text for " + esc(d.title) + "'>" + esc(d.body || "") + "</textarea>") + "</div>"; }).join("");
      else if (ragForgeDocs.length) h += "<p style='color:var(--dim);font-size:12px'>" + ragForgeDocs.length + " selected: " + ragForgeDocs.map(function (d) { return esc(d.title); }).join(", ") + "</p>";
      if (!ragEdit) h += "<div class='row' style='margin-top:8px'><input class='grow' id='rf-dir' type='text' placeholder='…or a folder on this machine — e.g. /home/name/documents (its .md/.txt/.pdf files, subfolders included)' aria-label='Folder of documents on this machine'></div>";
      h += "<button class='act' id='rf-forge' style='margin-top:8px'>" + (ragEdit ? "Save changes · re-index pack" : "Package pack") + "</button>";
      h += "<div class='card' style='margin-top:14px'><b>Publishing a pack</b><ol style='color:var(--dim);margin:8px 0 8px 18px;padding:0'><li>Forge the pack here in RAG Studio.</li><li>On its card in the Packs list, press <b>Edit README</b> if you want your own words on its GitHub page — your edited README is kept with the pack and ships in the export and the upload alike (Restore generated README undoes it).</li><li>Signed in to GitHub? Press <b>Publish to GitHub</b> on the pack's card, check the repo name (pre-filled from the pack's name, editable), and confirm: Navi creates the public repo if it is new, sets the <span class='mono'>navi-rag</span> topic on its own, and uploads the files. An existing repo is updated, and she says so.</li><li>Not signed in? Press <b>Export for GitHub</b> instead and download the ZIP, then create a <b>public GitHub repo</b> yourself, unzip the export at its root, and add the <span class='mono'>navi-rag</span> topic.</li><li>With the ZIP route, the last step is yours: on GitHub, <b>publish the release</b> — the version tag is created with that publish, so use the pack version from the manifest. With Publish to GitHub the release is published for you, knowledge cartridge attached (a pack too big for a picture releases without one, and says so). The Packs catalog finds the repo by its topic, and a newer release is what offers everyone an update.</li></ol><span style='color:var(--dim)'>What's in the export and the upload — the same file set both ways: a <span class='mono'>README.md</span> (the pack's GitHub page; installing skips a root README, so it is never indexed as a document), a <span class='mono'>navi-rag.json</span> manifest (name, version, document list), the built <span class='mono'>index.json</span> as a reference copy (installing Navi rebuilds it locally from the documents), the documents under <span class='mono'>documents/</span>, and a <span class='mono'>.gitignore</span>. The ZIP lands in <span class='mono'>rag-exports/</span>.</span></div>";
    } else if (CONFIG.backend !== "live") {
      h += "<div class='card'>Knowledge packs run through the live bridge — switch the backend to Live in Settings and this tab searches GitHub for the navi-rag topic, with install, delete, update, and per-pack active toggles. The packs folder starts empty in every mode: Navi ships with no pre-installed packs.</div>";
    } else {
      if (!ragCatalog && !ragCatalogBusy) loadRagCatalog();
      h += "<div class='row'><b>On GitHub · topic: navi-rag</b><span style='color:var(--dim)'>" + (ragCatalog && ragCatalog.fetchedAt ? "searched " + esc(String(ragCatalog.fetchedAt).slice(0, 10)) : ragCatalogBusy ? "searching…" : "not searched yet") + "</span><button class='ghost' id='rag-refresh'>Search GitHub</button></div>";
      h += "<p style='color:var(--dim)'>Every public repo tagged navi-rag is a knowledge pack — its .md/.txt/.pdf documents, indexed on this machine at install time. Only <b>active</b> packs feed her answers, and when a pack informs an answer, its pack and document show as a pill under that message — never spoken, and never carried onto messages that did not draw on a pack. Ask her in chat; a bare <span class='mono'>/rag</span> lists what's installed and what's active.</p>";
      h += "<div class='row'><span style='color:var(--dim);font-size:12px'>Have a pack as a picture? A knowledge cartridge installs from the picture alone — its index is built here, on this machine.</span><button class='ghost' id='rag-cart-pick'>Install from a picture…</button><input id='rag-cart-file' type='file' accept='image/png' style='display:none' aria-label='Choose a knowledge cartridge PNG'></div>";
      if (ragMsg) h += "<p>" + esc(ragMsg) + "</p>";
      if (ragCatalogBusy) h += "<p style='color:var(--dim)'>Searching GitHub for navi-rag…</p>";
      else if (ragCatalog && ragCatalog.error) h += "<div class='card' style='border-color:#f87171;color:#f87171'>" + esc(ragCatalog.error) + "</div>";
      else if (ragCatalog) {
        if (ragCatalog.warning) h += "<div class='card' style='border-color:#f5b544;color:#f5b544'>" + esc(ragCatalog.warning) + "</div>";
        // Issue #18: the upper list is the GitHub catalog and
        // nothing else. The payload also carries one summary
        // entry per installed pack the search did not return
        // (inCatalog false — e.g. a forged pack's local/…
        // record), and those rendered the same pack a second
        // time up here, with its own Active toggle and Delete,
        // on top of its full card under "On this machine".
        // They no longer render here — the local card below is
        // the one management surface, and it carries the
        // provenance the summary card used to state.
        // Issue #23: the catalog packs wear the chips' flip
        // cards now (ragCardHtml) — same grid, same frame.
        var catPacks = ragCatalog.packs.filter(function (c) { return c.inCatalog; });
        h += catPacks.length ? "<div class='nur-grid'>" + catPacks.map(ragCardHtml).join("") + "</div>" : "<p style='color:var(--dim)'>No repos carry the navi-rag topic right now.</p>";
      }
      if (ragPacksState && ragPacksState.length) {
        // Issue #23: the local cards sit in the same grid and
        // frame as the catalog flip cards — full-width rows of
        // it, since their editors and publish panels need the
        // room. Presentation only: every control below is the
        // same one, bound the same way, as before the makeover.
        h += "<h3 style='margin-top:16px'>On this machine · rag-packs/</h3><div class='nur-grid'>" + ragPacksState.map(function (p) {
          var exp = ragExp[p.slug];
          var pub = ragPub[p.slug];
          var red = ragReadmeEd[p.slug];
          var expHtml = exp ? (exp.error ? "<br><span style='color:#f5b544;font-size:12px'>" + esc(exp.error) + "</span>" : "<br><span style='font-size:12px'>Repo ZIP ready: <a class='ghost' href='/api/rag/export-file?name=" + encodeURIComponent(exp.file) + "' download='" + esc(exp.file) + "'>" + esc(exp.file) + "</a></span><br><span class='mono' style='color:var(--dim);font-size:11px'>" + exp.files.map(function (f) { return esc(f); }).join(" · ") + "</span>") : "";
          var cart = ragCart[p.slug];
          var cartHtml = cart ? (cart.error ? "<br><span style='color:#f5b544;font-size:12px'>" + esc(cart.error) + "</span>" : "<br><span style='font-size:12px'>Knowledge cartridge ready: <a class='ghost' href='/api/rag/export-file?name=" + encodeURIComponent(cart.file) + "' download='" + esc(cart.file) + "'>" + esc(cart.file) + "</a></span> <span class='mono' style='color:var(--dim);font-size:11px'>" + cart.size + " bytes — one picture; send it inside a .zip so nothing recompresses it</span>") : "";
          return "<div class='card' style='grid-column:1/-1;border-radius:8px;margin-bottom:0'><b>" + esc(p.name) + "</b> <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(p.slug) + "/ · " + (p.source === "github" ? esc(p.fullName) : "local pack" + (p.installedRef && p.installedRef !== "local pack" ? " · " + esc(p.installedRef) : "")) + "</span>" + (p.locallyEdited ? " <span class='mono' style='color:#f5b544;font-size:11px'>EDITED LOCALLY" + (p.source === "github" ? " · DIVERGES FROM " + esc(p.fullName) : "") + "</span>" : "") + (p.description ? "<br><span style='color:var(--dim)'>" + esc(p.description) + "</span>" : "") + "<br><span style='color:var(--dim);font-size:12px'>" + p.docCount + " documents · " + p.chunkCount + " indexed chunks · " + (p.active ? "active" : "off") + "</span><br><span class='mono' style='color:var(--dim);font-size:11px'>" + p.documents.map(function (d) { return esc(d); }).join(" · ") + "</span><br><label style='font-size:12px'><input type='checkbox' data-ragtoggle='" + esc(p.fullName) + "'" + (p.active ? " checked" : "") + "> Active — feeds her answers</label> <button class='ghost' data-ragedit='" + esc(p.slug) + "' aria-label='Edit pack " + esc(p.name) + " in RAG Studio'>Edit in RAG Studio</button> <button class='ghost' data-ragdelete='" + esc(p.fullName) + "'>Delete</button><br><label style='font-size:12px'>Version <input id='ragver-" + esc(p.slug) + "' class='mono' value='0.1.0' style='width:70px' aria-label='Export version for " + esc(p.name) + "'></label> <button class='ghost' data-ragexport='" + esc(p.slug) + "'>Export for GitHub</button> <button class='ghost' data-ragcartridge='" + esc(p.slug) + "' aria-label='Export pack " + esc(p.name) + " as a PNG knowledge cartridge'>Export as PNG</button> <button class='ghost' data-ragpub='" + esc(p.slug) + "' aria-label='Publish pack " + esc(p.name) + " to GitHub'>Publish to GitHub</button> <button class='ghost' data-ragreadme='" + esc(p.slug) + "' aria-label='Edit README for pack " + esc(p.name) + "'>Edit README</button>" + expHtml + cartHtml + pubPanelHtml(pub, "rag", p.slug) + readmeEdHtml(red, "rag", p.slug) + "</div>";
        }).join("") + "</div>";
      } else {
        h += "<h3 style='margin-top:16px'>On this machine · rag-packs/</h3><p style='color:var(--dim)'>No packs yet — the folder starts empty. Install one from the catalog above or forge your own in RAG Studio.</p>";
      }
    }
  } else if (view === "ChipDeck") {
    h += deckHtml();
  } else if (view === "Guardian") {
    h += guardianHtml();
  } else if (view === "GitHub") {
    h += githubViewHtml();
  } else if (view === "Docs") {
    h += "<h3>Documentation</h3>";
    h += docsHomeHtml();
  } else if (view === "Settings") {
    h += "<h3>Settings</h3>";
    // Topic tabs: every setting lives under exactly one.
    h += "<div role='tablist' aria-label='Settings topics' class='tabbar' style='margin-bottom:10px'>" +
      [["operator", "Operator"], ["persona", "Persona"], ["appearance", "Appearance"], ["voice", "Voice"], ["github", "GitHub & Backup"], ["system", "System"]].map(function (t) {
        return "<button type='button' role='tab' aria-selected='" + (settingsTab === t[0]) + "' class='" + (settingsTab === t[0] ? "act" : "ghost") + "' data-settab='" + t[0] + "'>" + t[1] + "</button>";
      }).join("") + "</div>";
    if (settingsTab === "appearance") h += "<div class='card'><label>Layout<br><select id='lay' aria-label='Interface layout'><option value='standard'" + (CONFIG.layout !== "lcars" ? " selected" : "") + ">Standard — the shipped console geometry</option><option value='lcars'" + (CONFIG.layout === "lcars" ? " selected" : "") + ">LCARS Mode — the starship layout: elbow cards, pill-stack rail, sweep headers</option></select></label><p style='color:var(--dim)'>Two layouts, independent of the theme below: every palette works in both. Flipping is a re-render, never a reset — the surface you are on, your place in it, an open case, a half-typed line and every switch all survive the flip, in either direction. The picker lives here in Appearance under both layouts, so the way back is always in its accustomed place. The choice persists in your config like the theme, and rides the settings backup beside it.</p></div>";
    if (settingsTab === "appearance") h += "<div class='card'><label>Theme<br><select id='th' aria-label='Interface theme'><option value='nord'" + (CONFIG.theme === "nord" ? " selected" : "") + ">Nordic dark — Polar Night surfaces, Snow Storm text, Aurora accents, color-coded icons</option><option value='glass'" + (CONFIG.theme === "glass" ? " selected" : "") + ">Dark Liquid Glass — frosted dark glass, cheap blur on the big surfaces</option><option value='terminal'" + (CONFIG.theme === "terminal" ? " selected" : "") + ">Terminal — green on black, the classic console</option><option value='dark'" + (CONFIG.theme === "dark" ? " selected" : "") + ">Signature dark — her console, cyan on near-black</option><option value='light'" + (CONFIG.theme === "light" ? " selected" : "") + ">Light — the same console on pale slate</option><option value='contrast'" + (CONFIG.theme === "contrast" ? " selected" : "") + ">High contrast — pure black, white text, bright signals</option><option value='nocturne'" + (CONFIG.theme === "nocturne" ? " selected" : "") + ">Nocturne Row — rain-slick Rockport night, neon cyan and magenta</option><option value='abyss'" + (CONFIG.theme === "abyss" ? " selected" : "") + ">Abyss — deep-sea black, one bioluminescent lure cyan</option><option value='casefile'" + (CONFIG.theme === "casefile" ? " selected" : "") + ">Case File — warm paper, ink text, evidence-red accents</option><option value='bloodmoon'" + (CONFIG.theme === "bloodmoon" ? " selected" : "") + ">Blood Moon — Sanguinari crimson over black, gold thread</option><option value='phosphor'" + (CONFIG.theme === "phosphor" ? " selected" : "") + ">Phosphor Amber — amber on black, Terminal's warmer sibling</option><option value='lcars'" + (CONFIG.theme === "lcars" ? " selected" : "") + ">LCARS — the starship console: black field, pill bars and elbow cards, the classic accent family with the 2.0 cooler blues</option></select></label><p style='color:var(--dim)'>Twelve palettes over the same style variables — every view, select, and input follows the choice, and it persists in your config like the other settings. Nocturne Row is the default. Nordic dark is the dark Nord family: Polar Night backgrounds, Snow Storm text, Frost and Aurora accents — and every menu icon carries its own color from the palette, in the full menu and the collapsed rail alike. Terminal is green on black. Her markdown answers follow the same palette in Nordic and Terminal too.</p></div>";
    if (settingsTab === "appearance") h += "<div class='card'><h3>Interface motion</h3><div style='display:flex;align-items:center;gap:8px'><button class='switch' id='motion-potato' role='switch' aria-checked='" + (CONFIG.motion === "potato" ? "true" : "false") + "' aria-label='Potato mode'><span class='knob'></span></button><span><b>Potato mode</b></span></div><p style='color:var(--dim)'>For low-power machines — turns off decorative animation across Apex. Menus, panes and views change instantly instead of traveling, and the Dark Liquid Glass theme's backdrop blur is suspended too (its colors stay). Off by default: the full motion language. The choice persists in your config like the theme, and rides the settings backup beside it." + ((window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches) ? " Your system asks for reduced motion — Apex honors that over this switch, either way." : "") + "</p></div>";
    if (settingsTab === "appearance") h += "<div class='card'><h3>Interface size</h3><p style='color:var(--dim)'>One scale for the whole interface — words and layout together, the way an operating system's display scale works, so nothing outgrows its row or its rail. (A separate text-size knob would multiply against this one and squeeze fixed rows at the big end; there is deliberately just the one.) Applies the moment you pick it, persists in your config beside the theme, and rides the settings backup with it.</p><div class='row' role='group' aria-label='UI scale'>" + [[1, "100%"], [1.1, "110%"], [1.25, "125%"], [1.5, "150%"]].map(function (o) { return "<button class='" + (Number(CONFIG.uiScale) === o[0] ? "act" : "ghost") + "' data-uiscale='" + o[0] + "' aria-pressed='" + (Number(CONFIG.uiScale) === o[0] ? "true" : "false") + "'>" + o[1] + "</button>"; }).join("") + "</div></div>";
    if (settingsTab === "system") (function () {
      var b = bootStateCache;
      h += "<div class='card'><h3>Startup \u2014 at boot</h3>";
      if (!b) {
        h += "<p style='color:var(--dim)'>Reading the boot state from the system\u2026</p>";
      } else {
        var line = b.armed ? "Armed \u2014 Apex comes up headless at boot" : "Not armed \u2014 Apex starts when you open it";
        h += "<p style='color:var(--dim)'><b style='color:var(--text)'>" + esc(line) + "</b><br>Bridge service " + bootUnitWord(b.bridgeUnit) + " \u00b7 llamafile service " + bootUnitWord(b.llamafileUnit) + " \u00b7 linger " + (b.linger === true ? "on" : b.linger === false ? "off" : "unknown") + " \u2014 read from this system, never assumed.</p>";
        h += bootButtonHtml("boot-action-settings");
        h += bootMsgHtml();
      }
      h += "</div>";
    })();
    // The Door (Build 2, #173): the trusted-device list lives
    // here in Settings → System AND on the DHD page — one shared
    // renderer (doorDevicesCardHtml), one registry behind it.
    // Sudo Mode (Build 3, #209): the arm switch lives here, and
    // it is a switch-gate surface \u2014 the flip, the bridge's
    // stored state and what a reload shows must agree. Flipping
    // on does not pretend: it reveals the masked prompt ON THIS
    // CARD, and the switch turns on only when the bridge
    // confirms the arm. A refusal (wrong password, cold sudo)
    // degrades honestly right here \u2014 the switch stays off and
    // the reason sits on the same card, never stranded elsewhere.
    if (settingsTab === "system") (function () {
      var st = CONFIG.backend === "live" ? sudoState : (CONFIG.sudo ? { enabled: true, state: "armed" } : { enabled: false, state: "off" });
      var on = !!(st && st.enabled);
      var word = !on ? "Off" : st.state === "armed" ? "Armed \u2014 root" : "On \u2014 timestamp expired";
      h += "<div class='card'><h3>Sudo Mode \u2014 Navi, elevated</h3>";
      h += "<p style='color:var(--dim)'>When armed, the commands Navi runs in chat may run as root. Arming feeds your password <b>once</b> to <span class='mono'>sudo -v</span> \u2014 it is never stored, logged or synced, by the page or the bridge; what persists is sudo's own credential timestamp and one on/off flag. When the timestamp lapses the mode reads <b>Expired</b> and elevated commands refuse honestly until you re-warm. Disarming is instant and total: the flag flips and the timestamp is killed. Every elevated command Navi runs is written to an append-only audit (<span class='mono'>config/sudo-audit.jsonl</span> in the install home), shown read-only below; the audit never rides Sync.</p>";
      h += "<div style='display:flex;align-items:center;gap:8px'><button class='switch' id='set-sudo-sw' role='switch' aria-checked='" + (on ? "true" : "false") + "' aria-label='Sudo mode' " + (sudoArmBusy ? "disabled" : "") + "><span class='knob'></span></button><span><b>" + esc(word) + "</b></span></div>";
      if (on && st.state === "expired") h += "<p style='font-size:12px;color:#f5b544'>sudo's timestamp has lapsed. Elevated commands refuse until you re-warm. <button class='ghost' id='set-sudo-rewarm' style='min-height:44px;padding:6px 10px'>Re-warm</button></p>";
      if (sudoArmOpen && !sudoArmBusy) {
        h += "<div style='margin-top:8px'><label style='color:var(--dim);font-size:12px' for='set-sudo-pw'>Sudo password \u2014 asked once per arm, never kept:</label>" +
          "<div class='row'><input class='grow' id='set-sudo-pw' type='password' autocomplete='off' aria-label='Sudo password for arming sudo mode' placeholder='sudo password'><button class='act' id='set-sudo-arm' style='min-height:44px'>Arm sudo mode</button><button class='ghost' id='set-sudo-cancel' style='min-height:44px'>Cancel</button></div></div>";
      }
      if (sudoArmBusy) h += "<p style='font-size:12px;color:var(--dim)'>Arming\u2026 (a fingerprint reader, if this machine has one, gets its say first.)</p>";
      if (sudoArmError) h += "<p style='font-size:12px;color:#f87171'>" + esc(sudoArmError) + "</p>";
      if (on) {
        h += "<div style='margin-top:10px'><button class='ghost' id='set-sudo-audit' style='min-height:44px;padding:6px 10px'>" + (sudoAuditEntries ? "Refresh the audit" : "Show the audit") + "</button></div>";
        if (sudoAuditEntries) {
          h += sudoAuditEntries.length ? "<div class='mono' style='font-size:12px;margin-top:8px;max-height:220px;overflow:auto'>" + sudoAuditEntries.slice().reverse().map(function (e) { return "<div>" + esc(new Date((e.ts || 0) * 1000).toLocaleString()) + " \u00b7 " + esc(e.command || "") + " \u00b7 chat " + esc(String(e.chat == null ? "\u2014" : e.chat)) + " \u00b7 exit " + esc(String(e.exit)) + "</div>"; }).join("") + "</div>" : "<p style='color:var(--dim);font-size:12px'>No elevated commands on the record yet.</p>";
        }
      }
      h += "</div>";
    })();
    if (settingsTab === "system") h += doorDevicesCardHtml();
    // Hardware profile (build .34): what the installer detected on
    // this machine and the runtime settings it chose from it.
    if (settingsTab === "system") (function () {
      var hp = hardwareState && hardwareState.profile ? hardwareState.profile : null;
      h += "<div class='card'><h3>Hardware profile — her runtime settings</h3>";
      if (!hardwareState && CONFIG.backend === "live") {
        h += "<p style='color:var(--dim)'>Loading the profile the installer wrote…</p>";
      } else if (!hp || !hp.detected) {
        h += "<p style='color:var(--dim)'>" + esc((hardwareState && hardwareState.error) || (hp && hp.note) || "No hardware profile is stored yet. Re-detect to profile this machine.") + "</p>";
      } else {
        var d = hp.detected, st = hp.settings || {};
        h += "<p style='color:var(--dim)'>Profiled at install from this machine — not copied from the developer's. Every launch path (this app, the bridge's Connect, the boot service) reads these flags from <span class='mono'>config/navi.config.json</span>; the full detection log is <span class='mono'>hardware-profile.log</span> beside the app. Changes apply the next time the model server starts.</p>";
        h += "<p class='mono' style='font-size:12px'>DETECTED<br>CPU: " + esc(d.cpuModel) + " — " + d.cpuCores + " logical cores" + (d.performanceCores ? ", " + d.performanceCores + " performance" : "") + "<br>RAM: " + Number(d.totalRamGB).toFixed(1) + " GiB<br>GPU: " + (d.gpuPresent ? esc(d.gpuName) + " (" + esc(d.gpuKind) + (d.gpuVramGB != null ? ", " + d.gpuVramGB + " GiB VRAM" : "") + ")" : "none detected") + "<br>OS: " + esc(d.osLabel) + "</p>";
        h += "<p class='mono' style='font-size:12px'>CHOSEN (" + esc(hp.source === "manual" ? "manual override" : "auto — derived at install") + (hp.tierLabel ? " · " + esc(hp.tierLabel) : "") + ")<br>--ctx-size " + st.ctxSize + " · --threads " + st.threads + " · --batch-size " + st.batchSize + (st.gpuLayers > 0 ? " · --n-gpu-layers " + st.gpuLayers : " · GPU layers 0 (CPU only)") + "</p>";
        if (hp.reasoning && hp.reasoning.length) h += "<ul style='color:var(--dim);font-size:12px;margin:6px 0 0 16px'>" + hp.reasoning.map(function (r) { return "<li>" + esc(r) + "</li>"; }).join("") + "</ul>";
        if (hp.autoSettings && hp.source === "manual") h += "<p style='color:var(--dim);font-size:12px'>Auto-derived for this hardware was: ctx " + hp.autoSettings.ctxSize + " · threads " + hp.autoSettings.threads + " · batch " + hp.autoSettings.batchSize + " · GPU layers " + hp.autoSettings.gpuLayers + ". Re-detect returns to auto.</p>";
      }
      h += "<div class='row' style='flex-wrap:wrap;gap:6px;margin-top:8px'><label style='font-size:12px'>Threads <input id='hw-thr' type='number' min='1' max='32' style='width:64px' aria-label='Threads override'></label><label style='font-size:12px'>Context <input id='hw-ctx' type='number' min='512' max='65536' step='512' style='width:84px' aria-label='Context size override'></label><label style='font-size:12px'>Batch <input id='hw-bat' type='number' min='32' max='4096' step='32' style='width:72px' aria-label='Batch size override'></label><label style='font-size:12px'>GPU layers <input id='hw-gpu' type='number' min='0' max='99' style='width:64px' aria-label='GPU layers override'></label></div>";
      h += "<div style='margin-top:8px'><button class='act' id='hw-save'>Save override</button> <button class='ghost' id='hw-redetect'>Re-detect hardware</button></div>";
      if (CONFIG.backend !== "live") h += "<p style='color:var(--dim);font-size:12px'>Mock mode: the override and re-detect buttons need the live bridge — in the bench in the Apex space they work against simulated machines. Here, switch to Live to manage the real profile.</p>";
      if (hardwareMsg) h += "<p style='font-size:12px'>" + esc(hardwareMsg) + "</p>";
      h += "</div>";
    })();
    if (settingsTab === "operator") h += "<div class='card'><h3>Phone notifications \u2014 this device</h3>" + pushDoorHtml() + "<p style='color:var(--dim);font-size:12px;margin:2px 0 0'>The same door stands at the head of the bell panel. A subscription belongs to the node you are connected to: each of your machines keeps its own list, and its own key.</p></div>";
    if (settingsTab === "operator") {
      var opBadgePayload = CONFIG.operatorBadge ? badgeParse(CONFIG.operatorBadge) : null;
      h += "<div class='card'><h3>Your picture &amp; badge</h3>"
        + "<div style='display:flex;gap:14px;align-items:flex-start;flex-wrap:wrap'>"
        + "<img class='m-av' style='width:64px;height:64px' src='" + esc(operatorPictureSrc()) + "' alt='Your chat picture'>"
        + "<div style='flex:1;min-width:220px'><p style='color:var(--dim);margin-top:0'>Your picture rides beside your messages in chat, the way hers rides beside hers. Upload one and it is kept with your profile on this machine and carried in the settings backup. Until you do, a picture generated from your name stands in.</p>"
        + "<div class='row'><button class='ghost' id='op-pic-pick'>Upload a picture…</button><input id='op-pic-file' type='file' accept='image/*' style='display:none' aria-label='Choose your profile picture'>" + (CONFIG.operatorPicture ? " <button class='ghost' id='op-pic-clear'>Use the generated picture</button>" : "") + "</div>"
        + "<p style='color:var(--dim);font-size:12px'>Uploads are trimmed to a 256&times;256 square on this machine before they are kept — the full original never lands in the profile.</p></div></div>"
        + "<h4 style='margin:14px 0 6px'>Operator Badge</h4>"
        + "<p style='color:var(--dim)'>A badge is your self-portrait in the cartridge format — your face in a card-grade frame, with your name (and this node's name) carried inside the picture itself. It is <b>not</b> a credential: nothing anywhere reads a badge to prove who you are, and a damaged payload simply reads as no badge at all. Pick the face — your uploaded picture, or art generated from your name — and mint it. With a badge minted and no picture uploaded, chat wears the badge's face for you.</p>"
        + (CONFIG.operatorBadge ? "<div style='display:flex;gap:14px;align-items:flex-start;flex-wrap:wrap'><img src='" + esc(CONFIG.operatorBadge) + "' alt='Your Operator Badge' style='width:132px;border-radius:8px'><div style='flex:1;min-width:200px'><p style='color:var(--dim);margin-top:0'>" + (opBadgePayload ? "This badge says: <b>" + esc(opBadgePayload.name || "") + "</b>" + (opBadgePayload.node ? " · node <b>" + esc(opBadgePayload.node) + "</b>" : "") + " — self-asserted, as every badge is." : "This badge's payload doesn't parse — it reads as no badge at all, and it harms nothing.") + "</p><div class='row'><button class='ghost' id='op-badge-export'>Export the badge picture</button> <button class='ghost' id='op-badge-clear'>Remove the badge</button></div></div></div>" : "<p style='color:var(--dim)'>No badge yet — mint one below, or slot a badge picture from a file.</p>")
        + "<div class='row' style='margin-top:8px'><button class='ghost' id='op-badge-slot'>Slot a badge from a file…</button><input id='op-badge-file' type='file' accept='image/png' style='display:none' aria-label='Choose an operator badge picture'><span style='font-size:12px;color:var(--dim)'>Minted on another machine? Its badge picture alone slots it here — the same single badge the account sync uses. This node: <b>" + esc(badgeNodeName || "…") + "</b></span></div>"
        + "<div class='row' style='margin-top:8px'><span style='font-size:12px;color:var(--dim)'>Face:</span> <button class='ghost' id='op-badge-face-photo' aria-pressed='" + (badgeFaceChoice === "photo") + "'>My picture</button> <button class='ghost' id='op-badge-face-art' aria-pressed='" + (badgeFaceChoice === "art") + "'>Generated art</button> <button class='act' id='op-badge-mint'>Mint my badge</button></div>"
        + (badgeMsg ? "<p style='font-size:12px'>" + esc(badgeMsg) + "</p>" : "")
        + "</div>";
    }
    if (settingsTab === "operator") h += "<div class='card'><h3>What Navi calls you</h3><p style='color:var(--dim)'>The name or form of address Navi uses whenever she talks to you — stored in the app config, used by her persona and her greetings every time she addresses you. Blank = Operator, the default on a fresh install.</p><div class='row'><input class='grow' id='op-name' type='text' value='" + esc(CONFIG.operatorName || "") + "' placeholder='e.g. Alex — blank means Operator' aria-label='Name Navi uses for the operator'><button class='act' id='op-save'>Save</button></div></div>";
    // About you — memories for Navi (Build 6, #110): the
    // operator's own file — the entries that exist because the
    // operator put them there (added by hand, or saved from
    // chat at the operator's ask). A view over the one memory
    // list, partitioned by the origin every entry already
    // carries; her side of the partition lives under Persona
    // as "Her file — what she keeps".
    if (settingsTab === "operator") {
      var opMem = memory.map(function (m, i) { return { m: m, i: i }; }).filter(function (x) { return x.m.o === "manual" || x.m.o === "trigger"; });
      h += "<div class='card'><h3>About you — memories for Navi</h3><p style='color:var(--dim)'>Facts you handed Navi about you — added by hand here, or saved from chat because you asked. Yours, in your words: true by your say-so, and forgetting one takes it out. What Navi keeps on her own judgment lives in her file, under Persona — together the two files are everything she remembers from chat.</p><div id='memlist'>"
        + (opMem.length ? opMem.map(function (x) { return "<div class='card'>" + esc(x.m.t) + " <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(MEMORY_ORIGIN_LABELS[x.m.o] || "Added by hand") + "</span> <button class='ghost' data-del='" + x.i + "'>Forget</button></div>"; }).join("") : "<p style='color:var(--dim);font-size:12px'>Nothing here yet — add the first fact below.</p>")
        + "</div><div class='row'><input class='grow' id='mem-in' type='text' placeholder='Something Navi should remember…' aria-label='New memory'><button class='act' id='mem-add'>Remember</button></div></div>";
    }
    // Soul and Memory live on the Persona tab, and on
    // no other Settings tab — the editors have exactly one home.
    // (The operator's name moved to the Operator tab in Build 6,
    // #110, with the operator's own memory file.)
    if (settingsTab === "persona") {
      h += "<div class='card'><h3>Persona — Soul</h3><p style='color:var(--dim)'>Who Navi is — her identity and voice. Lives in config/soul.md; the bridge feeds it to the model as her system prompt.</p><textarea id='soul-doc' rows='12' class='mono' style='font-size:12px' aria-label='Soul document — who Navi is'>" + esc(CONFIG.soul || "") + "</textarea><button class='act' id='soul-save' style='margin-top:8px'>Save Soul</button></div>";
      h += "<div class='card'><h3>Persona — Memory</h3><p style='color:var(--dim)'>What Navi carries between sessions. Lives in config/memory.md, beside the Soul.</p><textarea id='mem-doc' rows='9' class='mono' style='font-size:12px' aria-label='Memory document — what Navi carries'>" + esc(CONFIG.memory || "") + "</textarea><button class='act' id='memdoc-save' style='margin-top:8px'>Save Memory</button></div>";
    }
    if (settingsTab === "voice") {
      h += voiceHomeHtml();
    }
    if (settingsTab === "persona") {
    // What she remembers from chat (Build 6, #110): the leash and
    // the proposal queue moved here from Build 4's short-lived
    // Memories tab, beside the Soul and Memory document editors
    // they govern. The Build 4 pointer card is retired — the
    // document editor is on this very tab now, so there is
    // nothing left to point at.
    // How she remembers from chat (build .81, #72): the mode
    // selector, the receipt promise, and anything waiting on
    // the operator's word in Ask first mode.
    h += "<div class='card'><b>How she remembers from chat</b><br><span style='color:var(--dim)'>Tell her to remember something — or tap <b>Save to memory</b> under any message in Chat — and she writes it down and shows you the receipt, word for word. In the default mode she also saves durable facts on her own judgment, in the same reply, with the same receipt and a one-tap Undo. A save that corrects an existing entry replaces it. Secrets are never saved, on any path.</span><div style='margin-top:8px;display:flex;gap:6px;flex-wrap:wrap'>" + [["autonomous", "She saves on her own"], ["ask", "Ask first"], ["trigger", "Only when I say"]].map(function (o) { var on = (CONFIG.memoryMode || "autonomous") === o[0]; return "<button class='" + (on ? "act" : "ghost") + "' data-memmode='" + o[0] + "' aria-pressed='" + on + "' style='min-height:44px'>" + o[1] + "</button>"; }).join("") + "</div>";
    if (memoryProposals.length) {
      h += "<div style='margin-top:10px'><b>Waiting for your word</b>" + memoryProposals.map(function (p) { return "<div class='card' style='margin-top:6px'>“" + esc(p.text) + "”<div style='margin-top:6px'><button class='act' data-memprop='" + p.id + "' aria-label='Save this proposed memory' style='padding:2px 10px'>Save it</button> <button class='ghost' data-mempropno='" + p.id + "' aria-label='Dismiss this proposed memory' style='padding:2px 10px'>Dismiss</button></div></div>"; }).join("") + "</div>";
    }
    h += "</div>";
    // Her file — what she keeps: the entries that exist by her
    // judgment (saved on her own call, and her corrections) —
    // the other half of the origin partition. No add row: she
    // authors these, in chat, under the leash above.
    var herMem = memory.map(function (m, i) { return { m: m, i: i }; }).filter(function (x) { return x.m.o === "autonomous" || x.m.o === "correction"; });
    h += "<h3 style='margin-top:16px'>Her file — what she keeps</h3><p style='color:var(--dim)'>Entries that exist because Navi judged them worth keeping — saved on her own call in chat, or corrections she made to an earlier line. Facts you handed her live in your file, under Operator.</p>";
    h += "<div id='memlist'>" + (herMem.length ? herMem.map(function (x) { return "<div class='card'>" + esc(x.m.t) + " <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(MEMORY_ORIGIN_LABELS[x.m.o] || "From chat — her call") + "</span> <button class='ghost' data-del='" + x.i + "'>Forget</button></div>"; }).join("") : "<p style='color:var(--dim);font-size:12px'>Nothing here yet — what she saves on her own call in chat will land here.</p>") + "</div>";
    }
    if (settingsTab === "github") {
      if (!githubState && !githubFetching) fetchGithub(true);
      h += githubSettingsHtml();
    }
    if (settingsTab === "system") h += "<div class='card'><h3>Installed commands · autocomplete</h3>";
    if (CONFIG.backend === "live") {
      h += "<p style='color:var(--dim)'>Live mode: autocomplete comes from bridge.py's scan of this machine's real PATH — " + (liveCommands ? liveCommands.length + " commands found." : "scanning…") + " The built-in default set is always offered too.</p>";
    } else {
      h += "<p style='color:var(--dim)'>Mock mode: autocomplete = the built-in default set plus these installed commands. Install one and it starts completing in the chat composer.</p>";
      h += "<div>" + (CONFIG.installed || []).map(function (n) { return "<span class='chipbtn' style='cursor:default'>" + esc(n) + " <button class='ghost' style='padding:0 4px' data-uninst='" + esc(n) + "' aria-label='Uninstall " + esc(n) + "'>×</button></span>"; }).join("") + "</div>";
      var avail = CATALOG.filter(function (c) { return (CONFIG.installed || []).indexOf(c.name) < 0; });
      if (avail.length) h += "<div style='margin-top:8px'>" + avail.map(function (c) { return "<button class='ghost' data-inst='" + c.name + "' style='margin:0 4px 4px 0'>Install " + c.name + "</button>"; }).join("") + "</div>";
      h += "<div class='row' style='margin-top:8px'><input class='grow' id='cmd-custom' type='text' placeholder='Register your own — e.g. theharvester' aria-label='Custom command name'><button class='act' id='cmd-add'>Install custom</button></div>";
    }
    h += "</div>";
    // 0.2.0e Build 10 (#196): the Updater folds into System as
    // its own dropdown, and the Current Version Notes are the
    // final block at the bottom of the System menu.
    if (settingsTab === "system") h += settingsUpdaterHtml();
    if (settingsTab === "system" && CONFIG.backend === "live") h += updaterCurrentHtml();
  }
  if (view === "Docs") docsCaptureNav();
  mainEl.innerHTML = h;
  // Portal the article reader to <body> while it is open
  // (Build 10, #196): outside the view root's scroll and
  // animation ancestry, its fixed sheet anchors to the
  // viewport on every door. The Build 9 shape rendered it
  // inside the page-flow column — on a phone the shade
  // painted the screen while the panel sat mid-document,
  // off-screen: gray, and no article.
  var _wrp = document.getElementById("wire-reader");
  if (_wrp && _wrp.parentNode !== document.body) {
    var _wrs = document.getElementById("wire-reader-shade");
    if (_wrs) document.body.appendChild(_wrs);
    document.body.appendChild(_wrp);
    bindWireReaderClose();
  }
  document.body.classList.toggle("reader-open", !!_wrp);
  // The FediChat overlays (#270) ride the same road: the tray
  // and the About & settings window are generated inside the
  // view's markup and anchor fixed to <body>, like the reader.
  ["fc-shade", "fc-tray", "fc-win"].forEach(function (id) {
    var n = document.getElementById(id);
    if (n && n.parentNode !== document.body) document.body.appendChild(n);
  });
  if (document.getElementById("fc-tray") || document.getElementById("fc-win")) bindFedichatOverlays();
  // The Vault's create sheet (#279) rides the same road.
  ["vault-shade", "vault-sheet"].forEach(function (id) {
    var n = document.getElementById(id);
    if (n && n.parentNode !== document.body) document.body.appendChild(n);
  });
  if (document.getElementById("vault-sheet")) bindVaultSheet();
  if (document.getElementById("seraph-root")) bindSeraph();
  // Replay the view-entrance motion on every render so switching
  // views (and tabs) glides instead of snapping.
  mainEl.classList.remove("view-in");
  void mainEl.offsetWidth;
  mainEl.classList.add("view-in");
  // Chat view: the sidebars stay fixed and only the thread scrolls —
  // main becomes a flex frame and #msgs is its scrolling region.
  mainEl.classList.toggle("chat-view", view === "Chat");
  mainEl.classList.toggle("docs-view", view === "Docs");
  mainEl.classList.toggle("vault-view", view === "Vault");
  mainEl.classList.toggle("vault-reading", view === "Vault" && vaultReader && !!vaultSel);
  mainEl.classList.toggle("fedichat-view", view === "FediChat");
  wire();
  if (view === "Docs") docsRestoreNav();
  if (view === "Chat") {
    watchShellSession();
    wireShellUi();
    wireOperationCards(mainEl);
    messages.forEach(function (m) { if (m.operation && (m.operation.state === "running" || m.operation.state === "waiting")) watchOperation(m.operation.id); });
    var msgsEl = document.getElementById("msgs");
    if (msgsEl) msgsEl.scrollTop = msgsEl.scrollHeight;
    if (keepChatFocus) {
      var chatIn = document.getElementById("chat-in");
      if (chatIn) {
        chatIn.focus();
        try { chatIn.setSelectionRange(chatIn.value.length, chatIn.value.length); } catch (e) {}
      }
      chatFocusWanted = false;
    }
  }
}
function sendChat(text) {
  if (!text && !pendingUpload) return;
  if (pendingUpload) {
    var up = pendingUpload;
    pendingUpload = null;
    uploads.push({ name: up.name, kind: up.kind, size: up.size, at: new Date().toISOString(), chat: activeChat().title });
    persistUploads();
    if (up.kind === "text") {
      var body = up.text.length > 20000 ? up.text.slice(0, 20000) + "\n… (truncated at 20,000 characters)" : up.text;
      text = (text ? text + "\n\n" : "") + "[Attached file: " + up.name + "]\n" + body;
    } else {
      text = (text ? text + " " : "") + "[Attached image: " + up.name + " — stored with this chat for reference. The local model is text-only, so it is referenced by name, not seen.]";
    }
  }
  if (!text) return;
  // Firing a line hands the composer its focus back (see render()):
  // the operator keeps typing without clicking into the bar again.
  chatFocusWanted = true;
  // /cls (0.1.9e Build 11, issue #133): clearing the chat is a
  // composer command now — the retired Clear chat button's work,
  // done identically: the thread empties, this chat's fold state
  // goes with it, the store persists, the view re-renders. It is
  // intercepted here, before the latch flips and before the line
  // is pushed, so a /cls never lands in the thread it clears,
  // never reaches the model or the shell, and never enters the
  // command history — and it works in either latch, model
  // running or not. The whole line must be the command: a longer
  // line that merely starts with it routes as it always has.
  if (/^\s*\/(cls|clear)\s*$/i.test(text)) {
    messages.length = 0;
    // v0.6.3e (#318): a clear of the MAIN thread is a synced
    // act, not a local shrug — stamp the clear epoch on the
    // record before the persist carries it, so convergence on
    // every seat honors the emptied state over any copy of the
    // pre-clear thread, however new it claims to be. (Other
    // threads keep their existing semantics.)
    var _ac = activeChat();
    if (_ac && String(_ac.id) === "1" && !_ac.case) { _ac.clearedAt = Date.now(); _ac.clearedBy = ""; }
    Object.keys(chatFoldState).forEach(function (k) { if (k.indexOf(String(activeChatId) + ":") === 0) delete chatFoldState[k]; });
    persist(); render();
    return;
  }
  // A $ / > prefix flips the latch to match, so the quick way to
  // switch and the toggles never disagree afterwards.
  if (text.charAt(0) === "$") chatMode = "cli";
  else if (text === ">" || text.indexOf("> ") === 0) chatMode = "navi";
  persistChatMode();
  if (CONFIG.backend === "live" && chatMode === "cli" && !/^\s*\/(chip|rag)\b/i.test(text) && !isDhdChatIntent(text)) {
    // Build 4 (#214): CLI Mode \u2014 a CLI-latched line types
    // into this chat's standing shell session instead of
    // running as a one-shot command. Chip invocations and DHD
    // intents keep their own roads, exactly as they did under
    // the old latch; a > prefix already flipped the latch back
    // to Navi above, so it never reaches here.
    sendLineToShellSession(text);
    return;
  }
  messages.push({ who: "YOU", text: text, exec: null, ts: Date.now() });
  persist(); render();
  try {
    var _isChipLine = /^\s*\/(chip|rag)\b/i.test(text);
    if (!_isChipLine) {
      var _rr = routeOfLine(text, chatMode);
      if (llama.status !== "running" || _rr.kind === "cli") cliHistoryAppend(_rr.command || String(text).replace(/^\$\s*/, ""));
    }
  } catch (e) {}
  if (CONFIG.backend === "live" && isDhdChatIntent(text)) {
    // DHD chat operations are bridge work, not model work: they
    // route to /api/chat even while the model is disconnected, so
    // command-line mode cannot swallow the request as a shell line.
    naviThinking = true; render();
    liveChat(text).then(function (r) { naviThinking = false; messages.push({ who: "NAVI", text: r.text, exec: r.exec, sources: r.sources || null, operation: r.operation || null, ts: Date.now() }); persist(); render(); })
      .catch(function () { naviThinking = false; messages.push({ who: "NAVI", text: "The bridge could not be reached to run that DHD operation.", exec: null, ts: Date.now() }); persist(); render(); });
    return;
  }
  if (llama.status !== "running") {
    // Command-line mode: no model connected — the text IS the command.
    // Chips are local code, though: an invocation still runs its chip,
    // with or without the model.
    if (CONFIG.backend === "live" && (/^\s*\/chip\b/i.test(text) || /^\s*run\b/i.test(text))) {
      naviThinking = true; render();
      liveChat(text).then(function (r) { naviThinking = false; messages.push({ who: "NAVI", text: r.text, exec: r.exec, ts: Date.now() }); persist(); render(); })
        .catch(function () { naviThinking = false; messages.push({ who: "NAVI", text: "The bridge could not be reached to run that chip.", exec: null, ts: Date.now() }); persist(); render(); });
      return;
    }
    var mockChip = CONFIG.backend === "mock" ? matchChipText(text, CHIP_DEFS) : null;
    if (mockChip) {
      naviThinking = true; render();
      setTimeout(function () { naviThinking = false; messages.push({ who: "NAVI", text: "Slotted in " + mockChip.name + " and ran it — here's what it gave me:", exec: { command: "chip " + mockChip.id, output: mockChip.output, exit_code: 0 }, ts: Date.now() }); persist(); render(); }, 350);
      return;
    }
    var cmd = text.replace(/^\$\s*/, "");
    var eff = CONFIG.sudo && cmd.indexOf("sudo") !== 0 ? "sudo " + cmd : cmd;
    if (CONFIG.backend === "live") liveShell(cmd).then(function (r) { messages.push({ who: "SHELL", text: "", exec: { command: r.command, output: r.output, exit_code: r.exit_code }, ts: Date.now() }); persist(); render(); });
    else setTimeout(function () { messages.push({ who: "SHELL", text: "", exec: { command: eff, output: mockShell(cmd), exit_code: 0 }, ts: Date.now() }); persist(); render(); }, 100);
    return;
  }
  naviThinking = true; render();
  var done = function (r) {
    naviThinking = false;
    // A CLI-routed line files as a bare terminal block — the raw
    // output, no model framing around it.
    if (r.mode === "cli" || r.mode === "command-line") messages.push({ who: "SHELL", text: "", exec: r.exec, ts: Date.now() });
    else {
      var nm = { who: "NAVI", text: r.text, exec: r.exec, sources: r.sources || null, operation: r.operation || null, mesh: r.mesh || null };
      // Navi Mesh (build .85): the first mesh-served reply of a
      // session raises the notice card under the message — the
      // operator should never learn borrowing happened only from
      // a footnote.
      if (r.mesh && r.mesh.servedBy === "mesh" && !meshBorrowNoticed) { nm.meshFirst = true; meshBorrowNoticed = true; }
      // Memory from chat (build .81, #72): a reply can carry
      // memory work as payload data — file it against the store
      // and hang the receipts on this message.
      if (r.memorySaves || r.memoryRefused) { var rcs = applyMemorySaves(r.memorySaves, r.memoryRefused); if (rcs.length) nm.memoryReceipts = rcs; }
      // Her own case filings (Build 15, #161): the bridge filed
      // them before answering; the receipt hangs on this message,
      // in the thread where the filing happened.
      if (r.caseFilings && r.caseFilings.length) nm.caseFilings = r.caseFilings;
      // Her case logging (Build 2, #199): the receipt for what
      // she logged into the bound case's registers hangs on
      // this message the same way.
      if (r.caseLog) nm.caseLog = r.caseLog;
      // Starting a case from chat (v0.4.5e, #241): the
      // intake receipt — the case she opened, what the bridge
      // filed, and the door into the case's own chat — hangs
      // on this message the same way.
      if (r.caseIntake) nm.caseIntake = r.caseIntake;
      messages.push(nm);
    }
    persist(); render(); if (CONFIG.backend !== "live" && CONFIG.speakReplies && r.text) mockSpeak(r.text);
  };
  if (CONFIG.backend === "live") liveChat(text, CONFIG.speakReplies).then(done).catch(function (e) { done({ text: "Live backend error: " + e.message + (bridgeStale ? " — the bridge answering on port " + BRIDGE_PORT + " is an older build than this dashboard (see BRIDGE OUT OF DATE in the status column): fully quit the app and relaunch it so the current bridge takes over." : " — is bridge.py and the llamafile server running?"), exec: null }); });
  else setTimeout(function () { var mr = mockChat(text); if (!mr.exec && !mr.memorySaves && !mr.memoryRefused) { var cap = jsMemoryCapture(text); if (cap) { mr.memorySaves = cap.saves; mr.memoryRefused = cap.refused; } } done(mr); }, 450);
}
function wire() {
  wireNotifications();
  wireDay();
  mainEl.querySelectorAll("[data-docpage]").forEach(function (b) { b.onclick = function () { docsPage = b.getAttribute("data-docpage") || "getting-started"; try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {} if (view !== "Docs") view = "Docs"; render(); }; });
  wireDocTopics(mainEl);
  var docsPicker = document.getElementById("docs-picker");
  if (docsPicker) docsPicker.onchange = function () { docsPage = docsPicker.value || "getting-started"; try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {} if (view !== "Docs") view = "Docs"; render(); };
  var docsOut = document.getElementById("docs-pop-out");
  if (docsOut) docsOut.onclick = function () { popOutDocs(); render(); };
  var docsIn = document.getElementById("docs-pop-in");
  if (docsIn) docsIn.onclick = function () { dockDocs(); };
  if (view === "Chat") {
    // Case intake receipts (v0.4.5e, #241): the "Open the case
    // chat" button on a born case's receipt walks the same
    // jump the case page uses (jumpToCaseChat) and lands the
    // operator in the bound chat. If the bridge door somehow
    // refuses, a record already in the local store still
    // lands — the button is rigged to open, never decorative.
    mainEl.querySelectorAll("[data-caseintake-open]").forEach(function (b) { b.onclick = function () {
      var slug = b.getAttribute("data-caseintake-slug") || "";
      var cid = b.getAttribute("data-caseintake-open") || "";
      var land = function () {
        var c = chats.filter(function (x) { return String(x.id) === String(cid); })[0];
        if (c) { activeChatId = c.id; messages = c.messages; persist(); landOnChat(); return true; }
        return false;
      };
      if (slug) jumpToCaseChat(slug, function () { if (!land()) hydrateChatsFromServer(); });
      else land();
    }; });
    // Memory receipts (build .81, #72): Undo and proposal
    // decisions act on the receipt object stored on the message,
    // then persist and re-render so the store, the receipt, and
    // the Settings memory files all agree.
    mainEl.querySelectorAll("[data-memundo]").forEach(function (b) { b.onclick = function () { var parts = (b.getAttribute("data-memundo") || "").split(":"); var mm = messages[Number(parts[0])]; var rc = mm && mm.memoryReceipts && mm.memoryReceipts[Number(parts[1])]; if (rc && undoMemoryReceipt(rc)) { persist(); render(); } }; });
    mainEl.querySelectorAll("[data-memsave]").forEach(function (b) { b.onclick = function () { var parts = (b.getAttribute("data-memsave") || "").split(":"); var mm = messages[Number(parts[0])]; var rc = mm && mm.memoryReceipts && mm.memoryReceipts[Number(parts[1])]; if (rc) { resolveMemoryProposal(rc, true); persist(); render(); } }; });
    mainEl.querySelectorAll("[data-memdismiss]").forEach(function (b) { b.onclick = function () { var parts = (b.getAttribute("data-memdismiss") || "").split(":"); var mm = messages[Number(parts[0])]; var rc = mm && mm.memoryReceipts && mm.memoryReceipts[Number(parts[1])]; if (rc) { resolveMemoryProposal(rc, false); persist(); render(); } }; });
    // File into this case (v0.5.2e, #261): the deterministic
    // door on Navi's replies in a case chat. The bridge files
    // the exact reply and appends its own receipt message to
    // the stored chat — on success the room rehydrates and the
    // receipt is simply there; on failure the refusal is named
    // in the room, never swallowed.
    mainEl.querySelectorAll("[data-file-reply]").forEach(function (b) { b.onclick = function () {
      var mm = messages[Number(b.getAttribute("data-file-reply"))];
      if (!mm || !mm.text) return;
      b.disabled = true;
      casePost("/api/cases/file-reply", { chatId: activeChatId, text: mm.text, part: b.getAttribute("data-file-part") || undefined }, function (j) {
        if (j && j.ok) {
          hydrateChatsFromServer();
        } else {
          b.disabled = false;
          messages.push({ who: "NAVI", ts: Date.now(), text: "That filing did not land — " + ((j && j.error) || "the bridge could not file it.") + " Nothing was written to the case.", exec: null });
          persist(); render();
        }
      });
    }; });
    // Start a case from this (v0.5.14e, #289): the
    // deterministic door on Navi's replies in a chat that is
    // not a case room. The bridge composes the case from the
    // exact reply and appends its own receipt message to the
    // stored chat — on success the room rehydrates and the
    // receipt is simply there; on failure the refusal is
    // named in the room, never swallowed.
    mainEl.querySelectorAll("[data-start-case]").forEach(function (b) { b.onclick = function () {
      var mm = messages[Number(b.getAttribute("data-start-case"))];
      if (!mm || !mm.text) return;
      b.disabled = true;
      casePost("/api/cases/start-from-reply", { chatId: activeChatId, text: mm.text, part: b.getAttribute("data-start-part") || undefined }, function (j) {
        if (j && j.ok) {
          hydrateChatsFromServer();
        } else {
          b.disabled = false;
          messages.push({ who: "NAVI", ts: Date.now(), text: "That case did not start \u2014 " + ((j && j.error) || "the bridge could not start it.") + " Nothing was created.", exec: null });
          persist(); render();
        }
      });
    }; });
    // Reference shelf removal (v0.5.10e, #283): shelf-local —
    // the bridge drops the ref from this case's shelf only,
    // and the room rehydrates with the shelf as it now stands.
    // Reference shelf collapse (v0.6.7e, #333): the header
    // is the toggle — per-case view state, re-rendered in
    // place; the shelf's contents are untouched by folding.
    mainEl.querySelectorAll("[data-shelf-toggle]").forEach(function (b) { b.onclick = function () {
      shelfToggle(b.getAttribute("data-shelf-toggle"));
    }; });
    mainEl.querySelectorAll("[data-shelf-remove]").forEach(function (b) { b.onclick = function () {
      b.disabled = true;
      casePost("/api/cases/shelf", { case: b.getAttribute("data-shelf-case"), act: "remove", ref: b.getAttribute("data-shelf-remove") }, function (j) {
        if (j && j.ok) hydrateChatsFromServer();
        else { b.disabled = false; }
      });
    }; });
    mainEl.querySelectorAll("[data-memsg]").forEach(function (b) { b.onclick = function () {
      var mm = messages[Number(b.getAttribute("data-memsg"))];
      if (!mm || !mm.text) return;
      var fact = jsNormalizeFact(String(mm.text).replace(/^>\s+/, ""));
      if (!fact) return;
      if (jsMemorySecret(fact)) {
        messages.push({ who: "NAVI", ts: Date.now(), text: "I didn't save that one — it looks like a secret, and secrets never go into my memory, whichever door they knock on.", exec: null, memoryReceipts: [{ kind: "refused", text: "", replacedText: null, origin: "trigger", undone: false, proposalId: null }] });
      } else {
        var rc2 = applyMemorySave({ text: fact, origin: "trigger" });
        messages.push({ who: "NAVI", ts: Date.now(), text: rc2.kind === "corrected" ? "Saved from your message — and it corrects what I had, so the old line was replaced, not duplicated. The receipt shows both." : rc2.kind === "already" ? "That one's already in my memory, word for word — the receipt below is the entry as it stands." : "Saved from your message — the receipt below is exactly what I wrote down.", exec: null, memoryReceipts: [rc2] });
      }
      persist(); render();
    }; });
    var input = document.getElementById("chat-in");
    attachAutocomplete(input, { rich: true, hintEl: document.getElementById("chat-hint"), history: chatCliHistory });
    // While a call is live, the composer talks to the call —
    // the same routing the Voice panel's own input uses.
    var route = function (t) { if (!t) return; chatFocusWanted = true; if (call) callNaviTurn(t); else sendChat(t); };
    // Build 6 (#223): a submitted line leaves the composer. The
    // CLI/session path never re-renders the composer, so the
    // fired line simply sat in the bar after Enter — looking
    // unsent and one stray Enter away from firing twice. The
    // clear happens here, at the one funnel every Enter-submit
    // passes through, so the session path and the Navi path
    // clear identically; the input event re-arms the
    // autocomplete layer on the empty bar. (When the completion
    // dropdown is open, Enter accepts a completion instead —
    // that path returns above and clears nothing.)
    var send = function () { if (input.dataset.acOpen === "1") return; var t = input.value.trim(); if (!t) return; input.value = ""; try { input.dispatchEvent(new Event("input", { bubbles: true })); } catch (e) {} route(t); };
    // No Send button (build .49): Enter sends, exactly as it always
    // could — the button was a second way to do the same thing.
    input.onkeydown = function (e) { if (e.key === "Enter") send(); };
    // New case from chat (v0.4.5e, #241): the composer's case
    // door and its form's controls.
    var ncBtn = document.getElementById("chat-newcase");
    if (ncBtn) ncBtn.onclick = function () { openNewCaseForm(); };
    mainEl.querySelectorAll("[data-nc-cancel]").forEach(function (b) { b.onclick = function () { newCaseState = null; render(); }; });
    var ncShade = document.getElementById("nc-shade");
    if (ncShade) ncShade.onclick = function () { newCaseState = null; render(); };
    var ncSubmit = document.getElementById("nc-submit");
    if (ncSubmit) ncSubmit.onclick = function () { submitNewCaseForm(); };
    var upBtn = document.getElementById("chat-upload");
    var fileIn = document.getElementById("chat-file");
    if (upBtn && fileIn) {
      upBtn.onclick = function () { fileIn.click(); };
      fileIn.onchange = function () {
        var f = fileIn.files && fileIn.files[0];
        if (!f) return;
        var isImage = (f.type || "").indexOf("image/") === 0;
        if (isImage) {
          if (f.size > 1500000) { naviToast("That image is too large to store in the chat (over ~1.5 MB). Pick a smaller one."); return; }
          var rd = new FileReader();
          rd.onload = function () { pendingUpload = { name: f.name, kind: "image", size: f.size, dataUrl: String(rd.result || "") }; render(); };
          rd.onerror = function () { naviToast("That image could not be read — nothing was attached."); };
          rd.readAsDataURL(f);
        } else {
          var rt = new FileReader();
          rt.onload = function () { pendingUpload = { name: f.name, kind: "text", size: f.size, text: String(rt.result || "") }; render(); };
          rt.onerror = function () { naviToast("That file could not be read — nothing was attached."); };
          rt.readAsText(f);
        }
      };
    }
    var upClear = document.getElementById("chat-upload-clear");
    if (upClear) upClear.onclick = function () { pendingUpload = null; render(); };
    if (CONFIG.backend === "live" && !wfData && !naviThinking) {
      fetch("/api/workflows").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { wfData = j; if (view === "Chat") render(); }).catch(function () {});
    }
    mainEl.querySelectorAll("details[data-fold]").forEach(function (d) { d.addEventListener("toggle", function () { chatFoldState[d.getAttribute("data-fold")] = d.open; }); });
    var routePillEl = document.getElementById("chat-route-pill");
    if (routePillEl) routePillEl.onclick = function () { cycleChatRoute(); };
    if (CONFIG.backend === "live" && String(chatRouteChatId) !== String(activeChatId)) refreshChatRoute();
    var micBtn = document.getElementById("chat-mic");
    if (micBtn) micBtn.onclick = function () { pttToggle("chat"); };
    var chatCallBtn = document.getElementById("chat-call");
    if (chatCallBtn) chatCallBtn.onclick = function () {
      var why = callBlockReason();
      if (why) { voiceMsg = why; render(); return; }
      warmVoice();
      startCall();
      render();
    };
    // Chat tools (v0.4.7e, #246, as amended): the burger at the
    // end of the composer row toggles the tools list; the shade
    // dismisses it; Esc dismisses it; and a re-render that lands
    // while the list is open repaints its rows from the live DOM.
    var toolsBtn = document.getElementById("chat-tools");
    if (toolsBtn) toolsBtn.onclick = function () { if (chatToolsOpen) closeChatTools(); else openChatTools(); };
    var toolsShade = document.getElementById("tools-shade");
    if (toolsShade) toolsShade.onclick = function () { closeChatTools(); };
    ensureToolsEsc();
    if (chatToolsOpen) paintToolsMenu();
    var chatEndBtn = document.getElementById("call-end");
    if (chatEndBtn) chatEndBtn.onclick = function () { endCall(); };
    var chatPttBtn = document.getElementById("call-ptt");
    if (chatPttBtn) chatPttBtn.onclick = function () { pttToggle("call"); };
  } else if (view === "Chips") {
    mainEl.querySelectorAll("[data-chip]").forEach(function (b) { b.onclick = function () { var c = CHIP_DEFS.filter(function (x) { return x.id === b.getAttribute("data-chip"); })[0]; var pre = document.getElementById("chip-" + c.id); pre.style.display = "block"; if (CONFIG.backend === "mock") { pre.textContent = c.output; return; } pre.textContent = "Running " + c.name + " on the bridge…"; fetch("/api/chips/run", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ query: c.id }) }).then(function (r) { return r.json(); }).then(function (j) { pre.textContent = j.ok ? j.output : (j.error || "The chip could not be run."); }).catch(function () { pre.textContent = "The bridge could not be reached."; }); }; });
    mainEl.querySelectorAll("[data-lchip]").forEach(function (b) { b.onclick = function () { var id = b.getAttribute("data-lchip"); var pre = document.getElementById("lchip-" + id); if (!pre) return; pre.style.display = "block"; pre.textContent = "Running on the bridge…"; fetch("/api/chips/run", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ query: id }) }).then(function (r) { return r.json(); }).then(function (j) { pre.textContent = j.ok ? j.output : (j.error || "The chip could not be run."); }).catch(function () { pre.textContent = "The bridge could not be reached."; }); }; });
    mainEl.querySelectorAll("[data-ctab]").forEach(function (b) { b.onclick = function () { chipsTab = b.getAttribute("data-ctab"); chipMsg = ""; render(); }; });
    if (CONFIG.backend === "live" && chipsTab === "management") {
      var rf = document.getElementById("cat-refresh");
      if (rf) rf.onclick = loadChipCatalog;
      if (!chipCatalog && !chipCatalogBusy) loadChipCatalog();
      var chipPost = function (path, full, done) {
        fetch(path, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ full_name: full }) })
          .then(function (r) { return r.json(); })
          .then(function (j) { done(j); chipCatalog = null; wfData = null; if (view === "Chips") render(); })
          .catch(function () { chipMsg = "The bridge could not be reached."; render(); });
      };
      // Edit in Studio, from a chip's Local Chips card (the
      // catalog cards are discovery-only since issue #21): load
      // the chip through the Studio's own single path, then land
      // on the Studio tab with it open. A failed read stays on
      // this tab and says why in the view's message line — never
      // a silent nothing. The data-cupdate / data-cdelete
      // bindings inside bindNurCards below query the whole view,
      // so they bind the Local Chips cards' new Update and
      // Delete buttons too — one binding set, both card kinds.
      var openChipInStudio = function (file) {
        loadChipIntoStudio(file, function () { chipsTab = "studio"; chipMsg = ""; }, function (msg) { chipMsg = msg; });
      };
      // The catalog cards' buttons, bound through one helper so
      // the Community search can rebind after re-rendering just
      // the list (typing must not lose the input's focus to a
      // full render).
      var bindNurCards = function () {
        // Flip on a card click — but never when the click lands
        // on a button, a link, or the README scroll area (those
        // do their own thing); the README/Front buttons flip on
        // purpose.
        mainEl.querySelectorAll("[data-flipcard]").forEach(function (card) {
          card.onclick = function (ev) {
            var t = ev.target;
            var flipBtn = t.closest ? t.closest("[data-flipbtn]") : null;
            if (!flipBtn && t.closest && t.closest("button, a, .flip-scroll")) return;
            card.classList.toggle("flipped");
            if (card.classList.contains("flipped")) loadNurReadme(card.getAttribute("data-flipcard"));
          };
        });
        mainEl.querySelectorAll("[data-cinstall]").forEach(function (b) { b.onclick = function () { var full = b.getAttribute("data-cinstall"); chipMsg = "Downloading " + full + " into the chips folder…"; render(); chipPost("/api/chips/install", full, function (j) { chipMsg = j.ok ? "Installed " + full + " — its chip is in the chips folder, on the pill row, and /chip runs it." : (j.error || "Install failed."); }); }; });
        mainEl.querySelectorAll("[data-cupdate]").forEach(function (b) { b.onclick = function () { var full = b.getAttribute("data-cupdate"); chipPost("/api/chips/update", full, function (j) { chipMsg = !j.ok ? (j.error || "Update check failed.") : (j.updated ? "Updated " + full + " to the latest on GitHub." : full + " is already up to date."); }); }; });
        mainEl.querySelectorAll("[data-cdelete]").forEach(function (b) { b.onclick = function () { var full = b.getAttribute("data-cdelete"); var lc = wfData && wfData.chips ? wfData.chips.filter(function (x) { return x.sourceKey === full || ("local/" + String(x.file || "").replace(/.png$/i, "")) === full; })[0] : null; if (lc && lc.locallyEdited && !window.confirm("This chip has local edits — deleting removes the cartridge and your edits with it. Delete anyway?")) return; chipPost("/api/chips/delete", full, function (j) { chipMsg = j.ok ? "Deleted " + full + " from the installed chips." : (j.error || "Delete failed."); }); }; });
        mainEl.querySelectorAll("[data-cuninstall]").forEach(function (b) { b.onclick = function () { var full = b.getAttribute("data-cuninstall"); var cat = chipCatalog && chipCatalog.chips ? chipCatalog.chips.filter(function (x) { return x.fullName === full; })[0] : null; if (cat && cat.locallyEdited && !window.confirm("This chip has local edits — uninstalling removes the cartridge and your edits with it. Uninstall anyway?")) return; chipPost("/api/chips/delete", full, function (j) { chipMsg = j.ok ? "Uninstalled " + full + " — its cartridge and install record are gone." : (j.error || "Uninstall failed."); }); }; });
        mainEl.querySelectorAll("[data-chipedit]").forEach(function (b) { b.onclick = function () { openChipInStudio(b.getAttribute("data-chipedit")); }; });
      };
      bindNurCards();
      mainEl.querySelectorAll("[data-nurtab]").forEach(function (b) { b.onclick = function () { nurTab = b.getAttribute("data-nurtab"); render(); }; });
      var nurSearchEl = document.getElementById("nur-search");
      if (nurSearchEl) nurSearchEl.oninput = function () {
        nurQuery = nurSearchEl.value;
        var listEl = document.getElementById("nur-list");
        if (listEl) { listEl.innerHTML = nurCardsHtml(); bindNurCards(); }
        var metaEl = document.getElementById("nur-meta");
        if (metaEl) metaEl.textContent = nurMetaText();
        var clrEl = document.getElementById("nur-clear");
        if (clrEl) clrEl.style.display = nurQuery ? "" : "none";
      };
      var nurClearEl = document.getElementById("nur-clear");
      if (nurClearEl) nurClearEl.onclick = function () { nurQuery = ""; render(); };
      mainEl.querySelectorAll("[data-chipexport]").forEach(function (b) { b.onclick = function () {
        var file = b.getAttribute("data-chipexport");
        var stem = String(file || "").replace(/.png$/i, "");
        var verEl = document.getElementById("chipver-" + stem);
        var version = (verEl && verEl.value.trim()) || "0.1.0";
        chipExp[file] = { error: "Building the repo ZIP…" };
        render();
        fetch("/api/chips/export", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ file: file, version: version }) })
          .then(function (r) { return r.json(); })
          .then(function (j) { chipExp[file] = j.ok ? { file: j.file, files: j.files || [] } : { error: j.error || "Export failed." }; render(); })
          .catch(function () { chipExp[file] = { error: "The bridge didn't answer." }; render(); });
      }; });
    }
    if (CONFIG.backend === "live" && !wfData) {
      fetch("/api/workflows").then(gateGuard).then(function (r) { return r.json(); }).then(function (j) { wfData = j; if (view === "Chips") render(); }).catch(function () {});
    }
    if (chipsTab === "studio") {
      var ta = document.getElementById("st-code");
      var d = studioDraft;
      var lineH = function () { return studioZoom * 1.55; };
      var charW = function () { return studioZoom * 0.602; };
      var lineIndexOf = function (text, offset) { return text.slice(0, offset).split("\n").length - 1; };
      var offsetOfLine = function (text, idx) { var ls = text.split("\n"); var off = 0; for (var i = 0; i < idx && i < ls.length; i++) off += ls[i].length + 1; return off; };
      var gutterHtml = function (text) {
        var lines = text.split("\n");
        return lines.map(function (ln, i) {
          var btn = "";
          if (studioIsFoldLine(ln)) btn = "<button data-unfold='" + i + "' aria-label='Unfold the block at line " + (i + 1) + "'>▸</button>";
          else if (studioBlockStart(lines, i) > i) btn = "<button data-fold='" + i + "' aria-label='Fold the block at line " + (i + 1) + "'>▾</button>";
          return "<div>" + btn + (i + 1) + "</div>";
        }).join("");
      };
      var rootChecksFail = function () {
        if (!d || d.active !== "root.py") return [];
        var rc = studioRootCode(d);
        var chk = [["imports navi_internal", /^\s*import\s+navi_internal/m.test(rc)], ["declares command", /^\s*command\s*[:=]/m.test(rc)], ["declares use", /^\s*use\s*[:=]/m.test(rc)], ["defines run(arguments)", /^\s*def\s+run\s*\(/m.test(rc)]];
        return chk.filter(function (c) { return !c[1]; }).map(function (c) { return c[0]; });
      };
      var syncEd = function () {
        if (!d || !ta) return;
        d.code = ta.value;
        var pre = document.getElementById("ed-pre");
        if (pre) pre.innerHTML = highlightPy(ta.value) + "\n";
        var gut = document.getElementById("ed-gutter");
        if (gut) gut.innerHTML = gutterHtml(ta.value);
        var mini = document.getElementById("ed-mini-pre");
        if (mini) mini.textContent = ta.value;
        var lines = ta.value.split("\n");
        var guides = document.getElementById("ed-guides");
        if (guides) {
          if (studioWrap) guides.innerHTML = "";
          else {
            var gh = "";
            lines.forEach(function (ln, i) {
              var ind = (ln.match(/^\s*/) || [""])[0].length;
              for (var c = 2; c <= ind; c += 2) gh += "<i style='left:" + (12 + (c - 1) * charW()) + "px;top:" + i * lineH() + "px;height:" + lineH() + "px'></i>";
            });
            guides.innerHTML = gh;
          }
        }
        var squigs = document.getElementById("ed-squigs");
        if (squigs) {
          var fails = rootChecksFail();
          squigs.innerHTML = fails.length && !studioWrap ? "<i style='left:12px;top:" + (10 + lineH() - 3) + "px;width:" + Math.max(60, lines[0].length * charW()) + "px'></i>" : "";
        }
        var extrasEl = document.getElementById("ed-extras");
        if (extrasEl) {
          extrasEl.innerHTML = studioExtras.map(function (off) {
            var li = lineIndexOf(ta.value, off);
            var col = off - offsetOfLine(ta.value, li);
            return "<i style='left:" + (12 + col * charW()) + "px;top:" + (10 + li * lineH()) + "px;height:" + lineH() + "px'></i>";
          }).join("");
        }
        var st = document.getElementById("ed-status");
        if (st) {
          var caret = ta.selectionStart || 0;
          var li = lineIndexOf(ta.value, caret);
          var crumbs = "";
          for (var i = li; i >= 0; i--) { var m = /^(\s*)(def|class)\s+(\w+)/.exec(lines[i] || ""); if (m) { crumbs = m[2] + " " + m[3]; break; } }
          var fails2 = rootChecksFail();
          st.textContent = (studioAnyDirty(d) ? "● unsaved · " : "✓ saved · ") + "Ln " + (li + 1) + ", Col " + (caret - offsetOfLine(ta.value, li) + 1) + (crumbs ? " · " + crumbs : "") + (studioExtras.length ? " · " + (studioExtras.length + 1) + " cursors" : "") + (fails2.length ? " · ⚠ root.py: missing — it " + fails2[0] : "") + " · " + lines.length + " lines";
        }
        if (d.file && studioAnyDirty(d)) {
          if (studioSaveTimer) clearTimeout(studioSaveTimer);
          studioSaveTimer = setTimeout(function () { saveStudio(true); }, 1600);
        }
        syncAc();
      };
      var saveStudio = function (silent) {
        if (!d) return;
        if (CONFIG.backend !== "live") { if (!silent) { studioMsg = "Chip Studio forges chips through the live bridge — switch the backend to Live in Settings."; render(); } return; }
        var fs = studioFilesOf(d);
        var root = fs.filter(function (f) { return f.path === "root.py"; })[0];
        var iconRows = (d.icon || "").split("\n").map(function (s) { return s.replace(/\s+$/, ""); }).filter(function (s) { return s.length; });
        var payload = { name: (d.name || "").trim(), description: d.description || "", code: root ? root.content : "", output: d.output || "", files: fs, icon: iconRows };
        if (!payload.name || !payload.code.trim()) { if (!silent) { studioMsg = "A chip needs a name and code."; render(); } return; }
        if (d.file) {
          fetch("/api/chips/save", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ file: d.file, name: payload.name, description: payload.description, code: payload.code, output: payload.output, files: payload.files, icon: payload.icon }) })
            .then(function (r) { return r.json(); })
            .then(function (j) {
              if (j.ok) { d.snapshot = studioSnapshot(d); if (!silent) { studioMsg = "Saved — " + d.file + " was re-forged with the new code inside."; wfData = null; render(); } else { var st2 = document.getElementById("ed-status"); if (st2) st2.textContent = "✓ saved · autosaved just now"; } }
              else if (!silent) { studioMsg = j.error || "Save failed."; render(); }
            })
            .catch(function () { if (!silent) { studioMsg = "The bridge could not be reached."; render(); } });
        } else {
          fetch("/api/chips/create", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) })
            .then(function (r) { return r.json(); })
            .then(function (j) { if (j.ok) { studioMade = j.filename; chipCatalog = null; studioMsg = ""; d.file = j.filename; d.snapshot = studioSnapshot(d); wfData = null; } else studioMsg = j.error || "Forge failed."; render(); })
            .catch(function () { studioMsg = "The bridge could not be reached."; render(); });
        }
      };
      var STUDIO_AC_WORDS = ["navi_internal", "navi_instance", "print_message", "llm_chat", "process_message", "get_user", "set_user", "clear_terminal", "chat_with_navi", "arguments", "command", "use", "def", "return", "import", "from", "class", "if", "else", "elif", "for", "while", "try", "except", "with", "as", "print", "len", "str", "int", "list", "dict", "True", "False", "None"];
      var syncAc = function () {
        // Build 13 (#146): the phone's autocomplete row — completes
        // the word at the caret from the chip API's own vocabulary
        // plus Python's keywords; tapping a suggestion replaces
        // the partial word. Recomputed on every editor sync, the
        // same heartbeat as the status line.
        var row = document.getElementById("st-ac-row");
        if (!row || !ta) return;
        var caret = ta.selectionStart || 0;
        var m = /[A-Za-z_][A-Za-z0-9_]*$/.exec(ta.value.slice(0, caret));
        var word = m ? m[0] : "";
        var words = word.length >= 2 ? STUDIO_AC_WORDS.filter(function (x) { return x !== word && x.indexOf(word) === 0; }).slice(0, 6) : [];
        row.innerHTML = words.map(function (x) { return "<button class='ghost' data-stac='" + x + "' data-stacword='" + word + "'>" + esc(x) + "</button>"; }).join("");
        row.querySelectorAll("[data-stac]").forEach(function (b) { b.onclick = function () {
          var w0 = b.getAttribute("data-stacword") || "";
          var c0 = ta.selectionStart || 0;
          ta.focus();
          ta.setRangeText(b.getAttribute("data-stac"), Math.max(0, c0 - w0.length), c0, "end");
          syncEd();
        }; });
      };
      var gotoLine = function (n) {
        if (!ta || !n || n < 1) return;
        var off = offsetOfLine(ta.value, n - 1);
        ta.focus(); ta.setSelectionRange(off, off);
        var sc = mainEl.querySelector(".ed-scroll");
        if (sc) sc.scrollTop = Math.max(0, (n - 4) * lineH());
        syncEd();
      };
      var switchFile = function (path) {
        if (!d || path === d.active) return;
        if (ta) d.code = ta.value;
        var full = studioUnfold(d);
        d.files = studioFilesOf(d);
        d.active = path;
        var hit = d.files.filter(function (f) { return f.path === path; })[0];
        d.code = hit ? hit.content : "";
        d.folds = [];
        studioExtras = [];
        render();
      };
      if (ta) {
        ta.addEventListener("input", syncEd);
        ta.addEventListener("click", function (e) {
          if (e.altKey) {
            var off = ta.selectionStart || 0;
            if (studioExtras.indexOf(off) < 0) { studioExtras.push(off); studioExtras.sort(function (a, b) { return a - b; }); }
          }
          syncEd();
        });
        ta.addEventListener("keyup", syncEd);
        ta.addEventListener("keydown", function (e) {
          var start = ta.selectionStart || 0, end = ta.selectionEnd || 0;
          var mod = e.ctrlKey || e.metaKey;
          if (e.key === "Escape") {
            if (studioPal) { studioPal = false; render(); return; }
            if (studioFocus) { studioFocus = false; render(); return; }
            if (studioExtras.length) { studioExtras = []; syncEd(); }
            return;
          }
          if (mod && e.key.toLowerCase() === "s") { e.preventDefault(); saveStudio(false); return; }
          if (mod && e.shiftKey && e.key.toLowerCase() === "p") { e.preventDefault(); studioPal = true; studioPalQuery = ""; render(); setTimeout(function () { var q = document.getElementById("ed-pal-q"); if (q) q.focus(); }, 0); return; }
          if (mod && e.key.toLowerCase() === "g") { e.preventDefault(); var gi = document.getElementById("ed-goto"); if (gi) { gi.focus(); gi.select(); } return; }
          if (mod && (e.key === "=" || e.key === "+")) { e.preventDefault(); studioZoom = Math.min(20, studioZoom + 1); render(); return; }
          if (mod && e.key === "-") { e.preventDefault(); studioZoom = Math.max(10, studioZoom - 1); render(); return; }
          if (mod && e.shiftKey && e.key.toLowerCase() === "d") {
            e.preventDefault();
            var li1 = lineIndexOf(ta.value, start);
            var ls = ta.value.split("\n");
            ls.splice(li1, 0, ls[li1]);
            var off1 = offsetOfLine(ls.join("\n"), li1 + 1) + (start - offsetOfLine(ta.value, li1));
            ta.value = ls.join("\n"); ta.setSelectionRange(off1, off1); syncEd(); return;
          }
          if (e.altKey && (e.key === "ArrowUp" || e.key === "ArrowDown")) {
            e.preventDefault();
            var li2 = lineIndexOf(ta.value, start);
            var ls2 = ta.value.split("\n");
            var to = e.key === "ArrowUp" ? li2 - 1 : li2 + 1;
            if (to < 0 || to >= ls2.length) return;
            var tmp = ls2[to]; ls2[to] = ls2[li2]; ls2[li2] = tmp;
            var joined = ls2.join("\n");
            var off2 = offsetOfLine(joined, to) + (start - offsetOfLine(ta.value, li2));
            ta.value = joined; ta.setSelectionRange(off2, off2); syncEd(); return;
          }
          if (mod && e.key === "/") {
            e.preventDefault();
            var ls3 = ta.value.split("\n");
            var a = lineIndexOf(ta.value, start), b = lineIndexOf(ta.value, end);
            var allCommented = true;
            for (var i = a; i <= b; i++) { var t = ls3[i].replace(/^\s+/, ""); if (t && t.indexOf("#") !== 0) { allCommented = false; break; } }
            for (var j = a; j <= b; j++) {
              if (!ls3[j].trim()) continue;
              if (allCommented) ls3[j] = ls3[j].replace(/^(\s*)# ?/, "$1");
              else { var ind = (ls3[j].match(/^\s*/) || [""])[0]; ls3[j] = ind + "# " + ls3[j].slice(ind.length); }
            }
            ta.value = ls3.join("\n"); syncEd(); return;
          }
          if (studioExtras.length && !mod && !e.altKey) {
            var isText = e.key.length === 1, isBack = e.key === "Backspace", isDel = e.key === "Delete", isEnter = e.key === "Enter";
            if (isText || isBack || isDel || isEnter) {
              e.preventDefault();
              var carets = studioExtras.concat([start]).sort(function (x, y) { return x - y; });
              var val = ta.value;
              var inserts = carets.map(function () { return isText ? e.key : isEnter ? "\n" : ""; });
              for (var ci = carets.length - 1; ci >= 0; ci--) {
                var co = carets[ci];
                if (isBack) { if (co > 0) val = val.slice(0, co - 1) + val.slice(co); }
                else if (isDel) { val = val.slice(0, co) + val.slice(co + 1); }
                else val = val.slice(0, co) + inserts[ci] + val.slice(co);
              }
              var shift = 0, next = [];
              carets.forEach(function (co, idx) {
                var at = co + shift + (isText || isEnter ? 1 : 0);
                if (idx > 0) next.push(at);
                shift += isBack || isDel ? -1 : 1;
              });
              studioExtras = next;
              ta.value = val;
              ta.setSelectionRange(carets[0] + (isText || isEnter ? 1 : 0) - (isBack ? 1 : 0), carets[0] + (isText || isEnter ? 1 : 0) - (isBack ? 1 : 0));
              syncEd(); return;
            }
          }
          if (e.key === "Tab") {
            e.preventDefault();
            ta.setRangeText("  ", start, end, "end");
            syncEd();
          } else if (e.key === "Enter" && start === end) {
            var lsx = ta.value.lastIndexOf("\n", start - 1) + 1;
            var line = ta.value.slice(lsx, start);
            var ind = (line.match(/^\s*/) || [""])[0];
            if (/:\s*$/.test(line)) ind += "  ";
            e.preventDefault();
            ta.setRangeText("\n" + ind, start, end, "end");
            syncEd();
          } else if (start === end && "([{".indexOf(e.key) >= 0) {
            var close = { "(": ")", "[": "]", "{": "}" }[e.key];
            e.preventDefault();
            ta.setRangeText(e.key + close, start, end, "end");
            ta.setSelectionRange(start + 1, start + 1);
            syncEd();
          } else if (start === end && (e.key === "\"" || e.key === "'")) {
            e.preventDefault();
            ta.setRangeText(e.key + e.key, start, end, "end");
            ta.setSelectionRange(start + 1, start + 1);
            syncEd();
          }
        });
        syncEd();
      }
      var gutEl = document.getElementById("ed-gutter");
      if (gutEl) gutEl.addEventListener("click", function (e) {
        var foldBtn = e.target.closest ? e.target.closest("[data-fold]") : null;
        var unfoldBtn = e.target.closest ? e.target.closest("[data-unfold]") : null;
        if (foldBtn && d) {
          var i = Number(foldBtn.getAttribute("data-fold"));
          var lines = d.code.split("\n");
          var endLine = studioBlockStart(lines, i);
          if (endLine > i) {
            var block = lines.slice(i, endLine + 1);
            var placeholder = lines[i] + "  # ⋯ folded block · " + block.length + " lines";
            d.folds = d.folds.concat([{ placeholder: placeholder, lines: block }]);
            lines.splice(i, endLine - i + 1, placeholder);
            d.code = lines.join("\n");
            render();
          }
        } else if (unfoldBtn && d) {
          var j = Number(unfoldBtn.getAttribute("data-unfold"));
          var lines2 = d.code.split("\n");
          var ph = lines2[j];
          var fi = d.folds.findIndex(function (f) { return f.placeholder === ph; });
          if (fi >= 0) {
            lines2.splice(j, 1, ...d.folds[fi].lines);
            d.folds.splice(fi, 1);
            d.code = lines2.join("\n");
            render();
          }
        }
      });
      var miniEl = document.getElementById("ed-mini");
      if (miniEl) miniEl.addEventListener("click", function (e) {
        var sc = mainEl.querySelector(".ed-scroll");
        if (!sc) return;
        var rect = miniEl.getBoundingClientRect();
        var ratio = (e.clientY - rect.top) / Math.max(1, rect.height);
        sc.scrollTop = ratio * sc.scrollHeight;
      });
      var nm = document.getElementById("st-name"); if (nm) nm.addEventListener("input", function () { if (d) { d.name = nm.value; syncEd(); } });
      var ds = document.getElementById("st-desc"); if (ds) ds.addEventListener("input", function () { if (d) d.description = ds.value; });
      var si = document.getElementById("st-icon"); if (si) si.addEventListener("input", function () { if (d) d.icon = si.value; });
      var ou = document.getElementById("st-output"); if (ou) ou.addEventListener("input", function () { if (d) d.output = ou.value; });
      var un = document.getElementById("ed-undo"); if (un) un.onclick = function () { ta.focus(); document.execCommand("undo"); syncEd(); };
      var re2 = document.getElementById("ed-redo"); if (re2) re2.onclick = function () { ta.focus(); document.execCommand("redo"); syncEd(); };
      var fi2 = document.getElementById("ed-find"); if (fi2) fi2.addEventListener("input", function () { studioFind = fi2.value; });
      var fn = document.getElementById("ed-find-next");
      if (fn) fn.onclick = function () {
        if (!ta || !studioFind) return;
        var from = (ta.selectionEnd || 0);
        var idx = ta.value.indexOf(studioFind, from);
        if (idx < 0) idx = ta.value.indexOf(studioFind);
        if (idx >= 0) { ta.focus(); ta.setSelectionRange(idx, idx + studioFind.length); }
      };
      var ol = document.getElementById("ed-outline"); if (ol) ol.onchange = function () { if (ol.value) gotoLine(Number(ol.value)); };
      var gb = document.getElementById("ed-goto-btn"); if (gb) gb.onclick = function () { var gi = document.getElementById("ed-goto"); gotoLine(gi ? parseInt(gi.value, 10) : NaN); };
      var gi2 = document.getElementById("ed-goto"); if (gi2) gi2.addEventListener("keydown", function (e) { if (e.key === "Enter") { e.preventDefault(); gotoLine(parseInt(gi2.value, 10)); } });
      var zi = document.getElementById("ed-zoom-in"); if (zi) zi.onclick = function () { studioZoom = Math.min(20, studioZoom + 1); render(); };
      var zo = document.getElementById("ed-zoom-out"); if (zo) zo.onclick = function () { studioZoom = Math.max(10, studioZoom - 1); render(); };
      var wr = document.getElementById("ed-wrap"); if (wr) wr.onclick = function () { studioWrap = !studioWrap; render(); };
      var fsBtn = document.getElementById("ed-fs"); if (fsBtn) fsBtn.onclick = function () { studioFocus = !studioFocus; render(); };
      var palBtn = document.getElementById("ed-pal"); if (palBtn) palBtn.onclick = function () { studioPal = true; studioPalQuery = ""; render(); setTimeout(function () { var q = document.getElementById("ed-pal-q"); if (q) q.focus(); }, 0); };
      var palQ = document.getElementById("ed-pal-q"); if (palQ) palQ.addEventListener("input", function () { studioPalQuery = palQ.value; render(); var q2 = document.getElementById("ed-pal-q"); if (q2) { q2.focus(); q2.setSelectionRange(q2.value.length, q2.value.length); } });
      mainEl.querySelectorAll("[data-pal]").forEach(function (b) { b.onclick = function () {
        var cmd = b.getAttribute("data-pal");
        studioPal = false;
        if (cmd === "save") { saveStudio(false); return; }
        if (cmd === "focus") { studioFocus = !studioFocus; render(); return; }
        if (cmd === "wrap") { studioWrap = !studioWrap; render(); return; }
        if (cmd === "zoomin") { studioZoom = Math.min(20, studioZoom + 1); render(); return; }
        if (cmd === "zoomout") { studioZoom = Math.max(10, studioZoom - 1); render(); return; }
        if (cmd === "goto") { render(); var gi3 = document.getElementById("ed-goto"); if (gi3) { gi3.focus(); gi3.select(); } return; }
        if (cmd === "find") { render(); var fi3 = document.getElementById("ed-find"); if (fi3) { fi3.focus(); fi3.select(); } return; }
        if (cmd === "findall") { render(); var fa = document.getElementById("st-findall-q"); if (fa) { fa.focus(); fa.select(); } return; }
        if (cmd === "clearcursors") { studioExtras = []; render(); return; }
        if (!ta || !d) { render(); return; }
        render();
        setTimeout(function () {
          var t2 = document.getElementById("st-code");
          if (!t2 || !d) return;
          var lines = t2.value.split("\n");
          if (cmd === "foldall") {
            var out = [], i = 0;
            d.folds = [];
            while (i < lines.length) {
              var e2 = studioBlockStart(lines, i);
              if (e2 > i) { var block = lines.slice(i, e2 + 1); var ph = lines[i] + "  # ⋯ folded block · " + block.length + " lines"; d.folds.push({ placeholder: ph, lines: block }); out.push(ph); i = e2 + 1; }
              else { out.push(lines[i]); i++; }
            }
            d.code = out.join("\n"); render(); return;
          }
          if (cmd === "unfoldall") { d.code = studioUnfold(d); d.folds = []; render(); return; }
          t2.focus();
          var li = lineIndexOf(t2.value, t2.selectionStart || 0);
          if (cmd === "duplicate") { lines.splice(li, 0, lines[li]); t2.value = lines.join("\n"); }
          if (cmd === "moveup" && li > 0) { var t3 = lines[li - 1]; lines[li - 1] = lines[li]; lines[li] = t3; t2.value = lines.join("\n"); }
          if (cmd === "movedown" && li < lines.length - 1) { var t4 = lines[li + 1]; lines[li + 1] = lines[li]; lines[li] = t4; t2.value = lines.join("\n"); }
          if (cmd === "comment") {
            if (/^\s*#/.test(lines[li])) lines[li] = lines[li].replace(/^(\s*)# ?/, "$1");
            else { var ind = (lines[li].match(/^\s*/) || [""])[0]; lines[li] = ind + "# " + lines[li].slice(ind.length); }
            t2.value = lines.join("\n");
          }
          d.code = t2.value;
          var se = document.getElementById("st-code"); if (se) se.dispatchEvent(new Event("input"));
        }, 0);
      }; });
      mainEl.querySelectorAll("[data-stfile]").forEach(function (b) { b.onclick = function () { switchFile(b.getAttribute("data-stfile")); }; });
      mainEl.querySelectorAll("[data-strename]").forEach(function (b) { b.onclick = function () { studioRename = b.getAttribute("data-strename"); studioRenameText = studioRename; render(); }; });
      var rgo = document.getElementById("st-rename-go");
      if (rgo) rgo.onclick = function () {
        var inp = document.getElementById("st-rename-in");
        var to = inp ? inp.value.trim().replace(/^\/+/, "") : "";
        if (!d || !studioRename) return;
        if (!/^[A-Za-z0-9_][A-Za-z0-9_./-]*\.py$/.test(to) || to.indexOf("..") >= 0 || studioFilesOf(d).some(function (f) { return f.path === to; })) { studioMsg = "That new name won't work — use a plain Python path that isn't already taken."; studioRename = null; render(); return; }
        d.files = studioFilesOf(d).map(function (f) { return f.path === studioRename ? { path: to, content: f.content } : f; });
        if (d.active === studioRename) d.active = to;
        studioRename = null; render();
      };
      mainEl.querySelectorAll("[data-stdelete]").forEach(function (b) { b.onclick = function () {
        if (!d) return;
        var path = b.getAttribute("data-stdelete");
        d.files = studioFilesOf(d).filter(function (f) { return f.path !== path; });
        if (d.active === path) { d.active = "root.py"; var root = d.files.filter(function (f) { return f.path === "root.py"; })[0]; d.code = root ? root.content : ""; d.folds = []; }
        render();
      }; });
      var addBtn2 = document.getElementById("st-addfile");
      if (addBtn2) addBtn2.onclick = function () {
        var inp = document.getElementById("st-newfile");
        var path = inp ? inp.value.trim().replace(/^\/+/, "") : "";
        if (!d) return;
        if (!/^[A-Za-z0-9_][A-Za-z0-9_./-]*\.py$/.test(path) || path.indexOf("..") >= 0) { studioMsg = "A file name must be a plain Python path like helpers.py or tools/scan.py."; render(); return; }
        if (studioFilesOf(d).some(function (f) { return f.path === path; })) { studioMsg = path + " is already in this chip."; render(); return; }
        d.files = studioFilesOf(d).concat([{ path: path, content: "# " + path + " — a module root.py can import.\n" }]);
        switchFile(path);
      };
      var faBtn = document.getElementById("st-findall-btn");
      if (faBtn) faBtn.onclick = function () {
        var q = document.getElementById("st-findall-q");
        studioFindAll = q ? q.value : "";
        if (!d || !studioFindAll) { studioFindAllResults = null; render(); return; }
        var out = [];
        studioFilesOf(d).forEach(function (f) { f.content.split("\n").forEach(function (ln, i) { if (ln.indexOf(studioFindAll) >= 0) out.push({ path: f.path, line: i + 1, text: ln.trim().slice(0, 90) }); }); });
        studioFindAllResults = out.slice(0, 60);
        render();
      };
      mainEl.querySelectorAll("[data-findall-jump]").forEach(function (b) { b.onclick = function () {
        var parts = (b.getAttribute("data-findall-jump") || "").split("|");
        if (!d || parts.length !== 2) return;
        var targetLine = Number(parts[1]);
        if (parts[0] === d.active) { gotoLine(targetLine); return; }
        switchFile(parts[0]);
        setTimeout(function () { gotoLine(targetLine); }, 0);
      }; });
      var apiDocsBtn = document.getElementById("studio-api-docs");
      if (apiDocsBtn) apiDocsBtn.onclick = function () {
        // Land the wiki on the chip API page, then open the docs
        // the way the app opens them: the pop-out window (the
        // shell's own window, or a browser window) reads the page
        // from storage as it boots. If the pop-out is refused
        // outright, fall back to the Docs view itself — same page.
        docsPage = "navi-functions";
        try { localStorage.setItem("navi.docs.page", docsPage); } catch (e) {}
        popOutDocs();
        if (!docsInShell() && docsPopBlocked) { docsPopBlocked = false; view = "Docs"; render(); }
      };
      var tp = document.getElementById("st-template");
      if (tp) tp.onchange = function () {
        var t = CHIP_TEMPLATES.filter(function (x) { return x.id === tp.value; })[0];
        if (t && d) {
          d.files = studioFilesOf(d).map(function (f) { return f.path === "root.py" ? { path: f.path, content: t.code } : f; });
          if (d.active === "root.py") { d.code = t.code; d.folds = []; }
          render();
        }
      };
      var nw = document.getElementById("st-new");
      if (nw) nw.onclick = function () { studioDraft = defaultStudioDraft(); studioMsg = ""; studioMade = null; studioExtras = []; studioFindAllResults = null; render(); };
      var ob = document.getElementById("st-open-btn");
      if (ob) ob.onclick = function () {
        var sel = document.getElementById("st-open");
        var file = sel ? sel.value : "";
        if (!file) { studioMsg = "Pick a chip from the folder first."; render(); return; }
        loadChipIntoStudio(file, null, function (msg) { studioMsg = msg; });
      };
      var fg = document.getElementById("st-forge");
      // Build 13 (#146): the phone Studio's controls — the
      // segmented pane switch, the face strip's toggle, the
      // snippet shelf, and the thumb action bar. The switch and
      // the strip share one truth: choosing Face opens the
      // strip; the strip's own toggle can open it over any pane.
      mainEl.querySelectorAll("[data-stpane]").forEach(function (b) { b.onclick = function () {
        var pane = b.getAttribute("data-stpane");
        // Face is not a pane: its segment opens the strip (and
        // the strip's own toggle closes it again). Code and
        // Test are the panes — one at a time in the region.
        if (pane === "face") studioFaceOpen = true;
        else studioPane = pane;
        render();
      }; });
      var faceTgl = document.getElementById("st-face-toggle");
      if (faceTgl) faceTgl.onclick = function () { studioFaceOpen = !studioFaceOpen; render(); };
      mainEl.querySelectorAll("[data-stsnip]").forEach(function (b) { b.onclick = function () {
        if (!ta) return;
        var snip = STUDIO_SNIPPETS[b.getAttribute("data-stsnip")] || "";
        var s0 = ta.selectionStart || 0, e0 = ta.selectionEnd || 0;
        ta.focus();
        ta.setRangeText(snip, s0, e0, "end");
        syncEd();
      }; });
      var runStudioChip = function () {
        // Run tests the FORGED chip — the real .png in the chips
        // folder, through the bridge's own run route — and its
        // output lands in the Test pane.
        if (!d || !d.file) { studioRunOut = "Forge the chip first — Run tests the forged chip, the real .png in the chips folder."; studioPane = "test"; render(); return; }
        if (CONFIG.backend !== "live") { studioRunOut = "Run needs the live bridge — switch the backend to Live in Settings."; studioPane = "test"; render(); return; }
        studioRunOut = "Running " + d.file + " …";
        studioPane = "test"; render();
        fetch("/api/chips/run", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ query: String(d.file).replace(/\.png$/i, "") }) })
          .then(function (r) { return r.json(); })
          .then(function (j) { studioRunOut = (j && (j.output || j.error)) || "The chip ran and said nothing."; if (view === "Chips" && chipsTab === "studio") render(); })
          .catch(function () { studioRunOut = "The bridge didn't answer — the run never started."; if (view === "Chips" && chipsTab === "studio") render(); });
      };
      var actRun = document.getElementById("st-act-run"); if (actRun) actRun.onclick = runStudioChip;
      var actForge = document.getElementById("st-act-forge"); if (actForge) actForge.onclick = function () { saveStudio(false); };
      var actSave = document.getElementById("st-act-save"); if (actSave) actSave.onclick = function () { saveStudio(false); };
      if (fg) fg.onclick = function () { saveStudio(false); };
    }
    if (chipsTab === "workflows") {
      var stepsOf = function () { return CONFIG.backend === "live" ? wfDraftSteps : mockWfSteps; };
      var setSteps = function (next) { if (CONFIG.backend === "live") wfDraftSteps = next; else mockWfSteps = next; };
      var addBtn = document.getElementById("wf-add-btn");
      if (addBtn) addBtn.onclick = function () { var sel = document.getElementById("wf-add"); if (sel && sel.value) { setSteps(stepsOf().concat([sel.value])); render(); } };
      mainEl.querySelectorAll("[data-wf-up]").forEach(function (b) { b.onclick = function () { var i = Number(b.getAttribute("data-wf-up")); var s = stepsOf().slice(); if (i > 0) { var t = s[i - 1]; s[i - 1] = s[i]; s[i] = t; setSteps(s); render(); } }; });
      mainEl.querySelectorAll("[data-wf-down]").forEach(function (b) { b.onclick = function () { var i = Number(b.getAttribute("data-wf-down")); var s = stepsOf().slice(); if (i < s.length - 1) { var t = s[i + 1]; s[i + 1] = s[i]; s[i] = t; setSteps(s); render(); } }; });
      mainEl.querySelectorAll("[data-wf-rm]").forEach(function (b) { b.onclick = function () { var i = Number(b.getAttribute("data-wf-rm")); setSteps(stepsOf().filter(function (_, j) { return j !== i; })); render(); }; });
      var saveBtn = document.getElementById("wf-save");
      if (saveBtn) saveBtn.onclick = function () {
        var name = document.getElementById("wf-name").value.trim();
        var desc = document.getElementById("wf-desc").value.trim();
        var steps = stepsOf();
        if (!name || steps.length < 2) { chipMsg = "A workflow needs a name and at least two chip steps."; render(); return; }
        if (CONFIG.backend === "live") {
          fetch("/api/workflows/create", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ name: name, description: desc, steps: steps }) })
            .then(function (r) { return r.json(); })
            .then(function (j) { chipMsg = j.ok ? "" : (j.error || "The workflow could not be saved."); if (j.ok) { wfDraftSteps = []; wfData = null; } render(); })
            .catch(function () { chipMsg = "The bridge could not be reached."; render(); });
        } else {
          mockWorkflows.push({ id: Date.now(), name: name, description: desc, steps: steps.slice() });
          mockWfSteps = [];
          persistWorkflows();
          chipMsg = "";
          render();
        }
      };
      mainEl.querySelectorAll("[data-wfrun]").forEach(function (b) { b.onclick = function () {
        var id = Number(b.getAttribute("data-wfrun"));
        if (CONFIG.backend === "live") {
          chipMsg = ""; render();
          fetch("/api/workflows/run", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ id: id }) })
            .then(function (r) { return r.json(); })
            .then(function (j) { wfRuns[id] = j; render(); })
            .catch(function () { wfRuns[id] = { ok: false, error: "The bridge could not be reached.", steps: [], finalOutput: null }; render(); });
        } else {
          var w = mockWorkflows.filter(function (x) { return x.id === id; })[0];
          if (!w) return;
          var incoming = "", steps = [], error = null;
          for (var i = 0; i < w.steps.length; i++) {
            var def = CHIP_DEFS.filter(function (c) { return c.id === w.steps[i]; })[0];
            if (!def) { error = "Step chip \"" + w.steps[i] + "\" is no longer in the folder — the chain stops here."; break; }
            steps.push({ chipId: def.id, name: def.name, input: incoming, output: def.output });
            incoming = def.output;
          }
          mockWfRuns[id] = { ok: !error, error: error, steps: steps, finalOutput: error ? null : incoming };
          render();
        }
      }; });
      mainEl.querySelectorAll("[data-wfdel]").forEach(function (b) { b.onclick = function () {
        var id = Number(b.getAttribute("data-wfdel"));
        if (CONFIG.backend === "live") {
          fetch("/api/workflows/delete", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ id: id }) })
            .then(function () { wfData = null; delete wfRuns[id]; render(); })
            .catch(function () { chipMsg = "The bridge could not be reached."; render(); });
        } else {
          mockWorkflows = mockWorkflows.filter(function (x) { return x.id !== id; });
          delete mockWfRuns[id];
          persistWorkflows();
          render();
        }
      }; });
    }
  } else if (view === "CaseFiles") {
    wireCaseFiles();
  } else if (view === "ThreatIntel") {
    wireThreatIntel();
  } else if (view === "CTFBoard") {
    wireThreatIntel();
  } else if (view === "DHD") {
    wireDhd();
  } else if (view === "Mesh") {
    wireMeshCard();
  } else if (view === "Federation") {
    wireFederation();
  } else if (view === "FediChat") {
    wireFedichat();
  } else if (view === "Dossier") {
    wireDossier();
  } else if (view === "Vault") {
    var vq = function (sel) { return mainEl.querySelector(sel); };
    var vqa = function (sel) { return mainEl.querySelectorAll(sel); };
    vqa("[data-vault-open]").forEach(function (b) { b.onclick = function () { vaultOpenPath(decodeURIComponent(b.getAttribute("data-vault-open"))); }; });
    vqa("[data-vault-wiki]").forEach(function (b) { b.onclick = function () { vaultOpenWiki(decodeURIComponent(b.getAttribute("data-vault-wiki"))); }; });
    vqa("[data-wikilink]").forEach(function (b) { b.onclick = function (ev) { ev.preventDefault(); vaultOpenWiki(b.getAttribute("data-wikilink")); }; });
    vqa("[data-vault-fold]").forEach(function (b) { b.onclick = function () { var f = decodeURIComponent(b.getAttribute("data-vault-fold")); vaultCollapsed[f] = !vaultCollapsed[f]; render(); }; });