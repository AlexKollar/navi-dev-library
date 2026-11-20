# app/index.html — part 7 of 9

Source: `app/index.html` in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41, seq 41), reproduced verbatim below.
Lines 11339–13791 of app/index.html.

      var inp = document.getElementById("mesh-tunnel-confirm-in");
      fetch("/api/mesh/peers/tunnel-authorize", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: nodeId, action: "authorize", confirm: inp ? inp.value : "", badge: CONFIG.operatorBadge || "" }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) { meshState = j.mesh; meshTunnelConfirmFor = null; meshTunnelAuthMsg = { nodeId: nodeId, ok: true, text: j.message || "Saved." }; } else meshTunnelAuthMsg = { nodeId: nodeId, ok: false, text: (j && (j.error || j.message)) || "The bridge refused." }; render(); })
        .catch(function () { meshTunnelAuthMsg = { nodeId: nodeId, ok: false, text: "The bridge did not answer — is it running?" }; render(); });
    };
  });
  mainEl.querySelectorAll("[data-mesh-tunnel-cancel]").forEach(function (b) {
    b.onclick = function () { meshTunnelConfirmFor = null; render(); };
  });
  mainEl.querySelectorAll("[data-mesh-repair-clear]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-repair-clear");
      if (CONFIG.backend !== "live") { meshHealthMsg = "The repair needs the live bridge."; render(); return; }
      fetch("/api/mesh/peers/repair", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: nodeId }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) { meshState = j.mesh; meshHealth = j.health; meshHealthMsg = j.message || "Repaired."; } else meshHealthMsg = (j && j.error) || "The bridge refused the repair."; render(); })
        .catch(function () { meshHealthMsg = "The bridge did not answer — is it running?"; render(); });
    };
  });
  mainEl.querySelectorAll("[data-mesh-repair-import]").forEach(function (b) {
    b.onclick = function () {
      meshPngMsg = "Import the fresh card here — it replaces the dead addresses on the old record; the pairing, its pin, and its history are kept.";
      var zone = document.getElementById("mesh-png-zone");
      if (zone) { try { zone.scrollIntoView({ behavior: "smooth", block: "center" }); } catch (e) { try { zone.scrollIntoView(); } catch (e2) {} } }
    };
  });
  mainEl.querySelectorAll("[data-mesh-remove]").forEach(function (b) {
    b.onclick = function () {
      var nodeId = b.getAttribute("data-mesh-remove");
      if (CONFIG.backend !== "live") { meshMsg = "Unpairing needs the live bridge."; render(); return; }
      fetch("/api/mesh/peers/remove", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ nodeId: nodeId }) })
        .then(function (r) { return r.json(); })
        .then(function (j) { if (j && j.ok) { meshState = j.mesh; meshMsg = "Peer removed."; } else meshMsg = (j && j.error) || "The bridge refused."; render(); })
        .catch(function () { meshMsg = "The bridge did not answer — is it running?"; render(); });
    };
  });
}
function wireLocalNetwork() {
  var det = document.getElementById("local-net-details");
  if (det) det.addEventListener("toggle", function () { localNetOpen = det.open; });
  var copyWith = function (text, which) {
    var done = function () { localNetCopied = which; render(); setTimeout(function () { if (localNetCopied === which) { localNetCopied = ""; render(); } }, 2000); };
    if (navigator.clipboard && navigator.clipboard.writeText) navigator.clipboard.writeText(text).then(done).catch(function () {});
  };
  var lanBtn = document.getElementById("local-net-copy-lan");
  if (lanBtn) lanBtn.onclick = function () { var ps = publicState || CONFIG.publicServer || {}; if (ps.lanUrl) copyWith(ps.lanUrl, "lan"); };
  var codeBtn = document.getElementById("local-net-copy-code");
  if (codeBtn) codeBtn.onclick = function () { var ps = publicState || CONFIG.publicServer || {}; if (ps.accessCode) copyWith(ps.accessCode, "code"); };
  var setBtn = document.getElementById("local-net-settings");
  if (setBtn) setBtn.onclick = function () { view = "DHD"; dockOpen = false; render(); };
}
var sudoState = null;   // live: the bridge's sudo truth from /api/status (Build 3, #209) — {enabled, state}
var sudoArmOpen = false;    // the Settings card's masked prompt is revealed
var sudoArmBusy = false;    // an arm request is in flight — the switch will not double-flip
var sudoArmError = null;    // a refusal, shown on the same card that was flipped
var sudoAuditEntries = null; // the audit, loaded on demand (read-only)
function sudoWarnHtml() {
  return "<details id='sudo-warn'" + (sudoWarnOpen ? " open" : "") + " style='margin-top:6px'>" +
    "<summary style='color:#f87171;font-size:12px;cursor:pointer'>Sudo Mode Warning</summary>" +
    "<div style='color:#f87171;font-size:12px;margin-top:4px'>" + esc(SUDO_WARNING_LINE) + "</div></details>";
}
function renderSudo(live) {
  // Build 3 (#209): the status column READS sudo mode; it no
  // longer owns it. Arming lives in Settings \u2192 System \u2014 a
  // masked prompt feeding "sudo -v" once; no password is ever
  // held by this page. Here the column shows the bridge's own
  // state (Armed / Expired) with a Disarm button, and per the
  // #204 law the row is ABSENT when the feature is off or its
  // state is not yet known \u2014 never greyed, never guessed.
  // Mock mode reads the simulated CONFIG flag instead.
  if (CONFIG.backend !== "live") {
    if (!CONFIG.sudo) return "";
    return "<div class='stat'><span class='dot' style='background:#f87171'></span><span style='color:var(--dim)'>SUDO MODE</span><br>" +
      "<span class='mono' style='font-weight:700;color:#f87171'>On \u2014 root (simulated)</span> " +
      "<button class='ghost' id='sudo-disarm-status' style='min-height:44px;padding:6px 10px'>Disarm</button>" +
      sudoWarnHtml() + "</div>";
  }
  if (live && live.sudo) sudoState = live.sudo;
  if (!sudoState || !sudoState.enabled) return "";
  var armed = sudoState.state === "armed";
  return "<div class='stat'><span class='dot' style='background:" + (armed ? "#f87171" : "#f5b544") + "'></span><span style='color:var(--dim)'>SUDO MODE</span><br>" +
    "<span class='mono' style='font-weight:700;color:" + (armed ? "#f87171" : "#f5b544") + "'>" + (armed ? "Armed \u2014 root" : "Expired \u2014 re-warm in Settings") + "</span> " +
    "<button class='ghost' id='sudo-disarm-status' style='min-height:44px;padding:6px 10px'>Disarm</button>" +
    sudoWarnHtml() + "</div>";
}
function wireSudo() {
  var warn = document.getElementById("sudo-warn");
  if (warn) warn.addEventListener("toggle", function () { sudoWarnOpen = warn.open; });
  var dis = document.getElementById("sudo-disarm-status");
  if (dis) dis.onclick = function () {
    if (CONFIG.backend !== "live") { CONFIG.sudo = false; saveCfg(); render(); return; }
    dis.disabled = true; dis.textContent = "Disarming\u2026";
    fetch("/api/sudo/disarm", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
      .then(function (r) { return r.json(); })
      .then(function (j) { if (j && j.sudo) sudoState = j.sudo; CONFIG.sudo = false; render(); })
      .catch(function () { render(); });
  };
}
function wireSudoCard() {
  var sw = document.getElementById("set-sudo-sw");
  if (sw) sw.onclick = function () {
    if (sudoArmBusy) return;
    var on = CONFIG.backend === "live" ? !!(sudoState && sudoState.enabled) : !!CONFIG.sudo;
    if (on) { disarmSudo(); return; }
    // Flipping on reveals the prompt on this same card; the
    // switch itself turns on only when the bridge confirms.
    sudoArmOpen = true; sudoArmError = null; render();
    var f = document.getElementById("set-sudo-pw"); if (f) f.focus();
  };
  var arm = document.getElementById("set-sudo-arm");
  if (arm) arm.onclick = submitSudoArm;
  var pw = document.getElementById("set-sudo-pw");
  if (pw) pw.onkeydown = function (e) { if (e.key === "Enter") { e.preventDefault(); submitSudoArm(); } };
  var cancel = document.getElementById("set-sudo-cancel");
  if (cancel) cancel.onclick = function () { sudoArmOpen = false; sudoArmError = null; render(); };
  var rewarm = document.getElementById("set-sudo-rewarm");
  if (rewarm) rewarm.onclick = function () { sudoArmOpen = true; sudoArmError = null; render(); var f = document.getElementById("set-sudo-pw"); if (f) f.focus(); };
  var aud = document.getElementById("set-sudo-audit");
  if (aud) aud.onclick = function () {
    if (CONFIG.backend !== "live") { sudoAuditEntries = []; render(); return; }
    fetch("/api/sudo/audit").then(function (r) { return r.json(); }).then(function (j) { sudoAuditEntries = (j && j.entries) || []; render(); }).catch(function () { sudoAuditEntries = []; render(); });
  };
}
function submitSudoArm() {
  if (sudoArmBusy) return;
  var f = document.getElementById("set-sudo-pw");
  var pw = f ? f.value : "";
  if (!pw) { sudoArmError = "Enter the sudo password to arm sudo mode."; render(); return; }
  if (CONFIG.backend !== "live") { CONFIG.sudo = true; sudoArmOpen = false; sudoArmError = null; saveCfg(); render(); return; }
  sudoArmBusy = true; sudoArmError = null; render();
  fetch("/api/sudo/arm", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ password: pw }) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      sudoArmBusy = false;
      if (j && j.sudo) sudoState = j.sudo;
      if (j && j.ok) { sudoArmOpen = false; sudoArmError = null; CONFIG.sudo = true; }
      else { sudoArmError = (j && j.error) || "sudo mode was not armed."; sudoArmOpen = true; }
      render();
    })
    .catch(function () { sudoArmBusy = false; sudoArmError = "Could not reach the bridge to arm sudo mode."; sudoArmOpen = true; render(); });
}
function disarmSudo() {
  if (CONFIG.backend !== "live") { CONFIG.sudo = false; sudoAuditEntries = null; saveCfg(); render(); return; }
  fetch("/api/sudo/disarm", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
    .then(function (r) { return r.json(); })
    .then(function (j) { if (j && j.sudo) sudoState = j.sudo; CONFIG.sudo = false; sudoAuditEntries = null; render(); })
    .catch(function () { render(); });
}
/* ------------------------------- voice ------------------------------- */
/* Mock mode's simulated voice catalog. Live mode never uses these:
   the bridge enumerates the installed engine's real voices and the
   picker is built from that list (see the Voice view). */
var KOKORO_VOICES = [["bf_isabella", "Isabella (UK female)"], ["af_heart", "Heart — warm, natural (US female)"], ["af_bella", "Bella (US female)"], ["af_sarah", "Sarah (US female)"], ["af_nicole", "Nicole (US female)"], ["am_michael", "Michael (US male)"], ["bf_emma", "Emma (UK female)"]];
var PIPER_VOICES = [["en_US-lessac-medium", "Lessac (US female)"], ["en_US-ryan-high", "Ryan (US male)"], ["en_GB-alan-medium", "Alan (UK male)"], ["en_GB-alba-medium", "Alba (UK female)"]];
var pttRecording = false;
var voiceCalls = null;
function fmtCallTime(sec) { var m = Math.floor(sec / 60), s = sec % 60; return (m < 10 ? "0" : "") + m + ":" + (s < 10 ? "0" : "") + s; }
function uploadSvg() { return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M21.44 11.05l-9.19 9.19a6 6 0 0 1-8.49-8.49l8.57-8.57A4 4 0 1 1 18 8.84l-8.59 8.57a2 2 0 0 1-2.83-2.83l8.49-8.48'/></svg>"; }
function micSvg() { return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M12 2a3 3 0 0 0-3 3v7a3 3 0 0 0 6 0V5a3 3 0 0 0-3-3Z'/><path d='M19 10v2a7 7 0 0 1-14 0v-2'/><path d='M12 19v3'/></svg>"; }
function caseSvg() { return "<svg aria-hidden='true' focusable='false' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M20 20a2 2 0 0 0 2-2V8a2 2 0 0 0-2-2h-7.9a2 2 0 0 1-1.69-.9L9.6 3.9A2 2 0 0 0 7.93 3H4a2 2 0 0 0-2 2v13a2 2 0 0 0 2 2Z'/><path d='M12 10v6'/><path d='M9 13h6'/></svg>"; }
// The phone glyph (the same Lucide path as the Voice nav icon) for
// the Call Navi buttons: the Voice view's call button, the Call
// button beside the chat composer, and the status column's Call
// Navi button all carry it.
function phoneSvg(size) { var s = size || 16; return "<svg aria-hidden='true' focusable='false' width='" + s + "' height='" + s + "' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'><path d='M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07 19.5 19.5 0 0 1-6-6 19.79 19.79 0 0 1-3.07-8.67A2 2 0 0 1 4.11 2h3a2 2 0 0 1 2 1.72c.127.96.361 1.903.7 2.81a2 2 0 0 1-.45 2.11L8.09 9.91a16 16 0 0 0 6 6l1.27-1.27a2 2 0 0 1 2.11-.45c.907.339 1.85.573 2.81.7A2 2 0 0 1 22 16.92z'/></svg>"; }
function refreshVoice(thenRender) {
  if (CONFIG.backend !== "live" || voiceFetching) return;
  voiceFetching = true;
  fetch("/api/voice").then(function (r) { return r.json(); }).then(function (j) {
      if (!j || !j.config || !j.stt || !j.tts) throw new Error(j && j.error ? j.error : "the bridge answered the voice check with an unreadable response");
      voiceState = j; voiceFetching = false;
      // One source of truth: this response is the same probe the
      // pipeline strip and the status column render. When it says
      // the engines are here, a stale "not installed" line left by
      // an earlier failed speak clears instead of bannering over a
      // working voice in both views.
      if (j.stt.available && j.tts.available && /not installed/i.test(voiceMsg)) voiceMsg = "";
      if (thenRender) render();
    })
    .catch(function (e) { var why = e && e.message ? e.message : "no answer"; voiceState = { enabled: true, config: { stt_model: "small.en", tts_engine: "kokoro", tts_voice: "bf_isabella", rate: 1, pitch: 1 }, stt: { available: false, detail: "The voice check itself failed: " + why + "." }, tts: { available: false, detail: "If the rest of this page works, the bridge is up and its /api/voice endpoint is the failing piece — a bridge and dashboard from different builds can do that; fully relaunching the app lines them up." }, voices: [], voices_source: "current", call: null, offline: true }; voiceFetching = false; if (thenRender) render(); });
}
function refreshVoiceCalls() {
  if (CONFIG.backend !== "live") return;
  fetch("/api/voice/calls").then(function (r) { return r.json(); }).then(function (j) { voiceCalls = j.calls || []; if (view === "Settings") render(); }).catch(function () {});
}
function warmVoice() {
  // Warm her engines — once per session, fire-and-forget — so a
  // call never pays the model load inside the pickup. Opening
  // the Voice view asks, and so does starting a call from the
  // chat window. Never at bridge boot: on a small machine those
  // models are real RAM.
  if (CONFIG.backend !== "live" || voiceWarmed) return;
  voiceWarmed = true;
  fetch("/api/voice/warmup", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).catch(function () {});
}
function callBlockReason() {
  // Why a call can't start right now, from the same voice probe
  // everything else reads — "" when she can be called. It backs
  // the chat window's Call button the way the Voice view's own
  // disabled state backs its panel.
  if (CONFIG.backend !== "live" || !voiceState) return "";
  if (!voiceState.enabled) return "Voice is turned off — switch it on in Settings → Voice.";
  if (!voiceState.stt.available || !voiceState.tts.available) return "Her speech engines aren't installed yet — Settings → Voice names the install step.";
  return "";
}
function renderVoiceStatus() {
  // Build 2 (#204): the status-bar law — voice off in
  // Settings means no VOICE row in the status column at all.
  if (CONFIG.backend === "live" && voiceState && !voiceState.enabled) return "";
  var inCall = !!call;
  var avail = CONFIG.backend === "live" && voiceState ? (voiceState.stt.available && voiceState.tts.available) : null;
  var label = CONFIG.backend !== "live" ? "Simulated (mock mode)" : inCall ? "In a call with you" : voiceState && !voiceState.enabled ? "Off" : avail === false ? "Engines not installed" : "On — she can be called";
  var dot = inCall ? "#34d399" : avail === false ? "#f5b544" : "#22d3ee";
  var sub = CONFIG.backend === "live" && voiceState ? "hears: faster-whisper " + esc(voiceState.config.stt_model) + " · speaks: " + esc(voiceState.config.tts_engine) + " " + esc(voiceState.config.tts_voice) : "local speech — faster-whisper hears, Kokoro speaks";
  return "<div class='stat'><span class='dot" + (inCall ? " pdot" : "") + "' style='background:" + dot + "'></span><span style='color:var(--dim)'>VOICE</span><br>" +
    "<span class='v mono'>" + esc(label) + "</span><br><span style='color:var(--dim);font-size:12px'>" + sub + "</span></div>";
}
function playWavB64(b64) {
  return new Promise(function (resolve) {
    try {
      var audio = new Audio("data:audio/wav;base64," + b64);
      audio.onended = function () { resolve(); };
      audio.onerror = function () { resolve(); };
      var p = audio.play();
      if (p && p.catch) p.catch(function () { resolve(); });
    } catch (e) { resolve(); }
  });
}
function mockSpeak(text) {
  // Mock mode's stand-in voice: the browser's own speech synthesis.
  // It is a simulation and the UI says so — the real engines are
  // faster-whisper + Kokoro/Piper through the live bridge.
  return new Promise(function (resolve) {
    try {
      if (!("speechSynthesis" in window)) { resolve(); return; }
      var u = new SpeechSynthesisUtterance(text);
      u.rate = 1.02;
      u.onend = function () { resolve(); };
      u.onerror = function () { resolve(); };
      speechSynthesis.speak(u);
    } catch (e) { resolve(); }
  });
}
function splitSentences(text) {
  var BT = String.fromCharCode(96);
  var fence = new RegExp(BT + "{3}[\s\S]*?" + BT + "{3}", "g");
  var clean = String(text || "").replace(fence, " (code on your screen) ").split(BT).join("").replace(/[#>*_]/g, "").replace(/\s+/g, " ").trim();
  if (!clean) return [];
  var raw = clean.match(/[^.!?]+[.!?]+["')\]]*|\S[^.!?]*$/g) || [clean];
  var out = [];
  raw.forEach(function (s) { s = s.trim(); if (s) out.push(s); });
  return out.length ? out : [clean];
}
function speakText(text) {
  if (CONFIG.backend !== "live") return mockSpeak(text);
  // Build 6 (#224): one sentence's synthesis is bounded — a
  // hung TTS request used to strand the call panel at "Navi is
  // speaking…" forever, because the speaking state only lifts
  // when this promise settles. Ninety seconds is far beyond a
  // sentence's honest synthesis time on CPU; past it, the line
  // is named unsaid and the chain moves on.
  var ctl = null, timer = null;
  try {
    if (window.AbortController) {
      ctl = new AbortController();
      timer = setTimeout(function () { try { ctl.abort(); } catch (e) {} }, 90000);
    }
  } catch (e) {}
  var opts = { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ text: text }) };
  if (ctl) opts.signal = ctl.signal;
  return fetch("/api/voice/tts", opts)
    .then(function (r) { return r.json(); })
    .then(function (j) { if (j.ok && j.audio_b64) return playWavB64(j.audio_b64); if (!j.ok) voiceMsg = j.error || "She couldn't speak that line."; })
    .catch(function () { voiceMsg = "That line didn't get spoken — the speech request didn't finish."; })
    .then(function () { if (timer) clearTimeout(timer); });
}
function speakSentences(text) {
  // Her reply, spoken sentence by sentence in order — she starts
  // talking as soon as the first sentence exists instead of waiting
  // for the whole reply to synthesize.
  var parts = splitSentences(text);
  var chain = Promise.resolve();
  parts.forEach(function (p) { chain = chain.then(function () { return speakText(p); }); });
  return chain;
}
function recordCallTurn(who, text, exec) {
  if (CONFIG.backend !== "live") return;
  fetch("/api/voice/call/turn", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ who: who, text: text, exec: exec || null }) }).catch(function () {});
}
function startCall() {
  if (call || callStarting) return;
  voiceMsg = "";
  var begin = function () {
    callStarting = false;
    voiceMsg = "";
    call = { startedAt: Date.now(), turns: [] };
    callSeconds = 0; callState = "listening";
    if (callTimer) clearInterval(callTimer);
    callTimer = setInterval(function () { callSeconds++; var el = document.getElementById("call-clock"); if (el) el.textContent = fmtCallTime(callSeconds); }, 1000);
    // She picks up: a greeting in her voice, recorded as the call's
    // first turn and filed into the conversation like the rest.
    var greeting = "Hey " + opName() + " — it's me, Navi. Good to hear you. What's on your mind?";
    call.turns.push({ who: "NAVI", text: greeting, exec: null, ts: Date.now() });
    messages.push({ who: "NAVI", text: greeting, exec: null, ts: Date.now() });
    recordCallTurn("navi", greeting, null);
    persist(); render();
    callState = "speaking"; render();
    speakText(greeting).then(function () { if (call) { callState = "listening"; render(); } }, function () { if (call) { callState = "listening"; render(); } });
  };
  if (CONFIG.backend === "live") {
    callStarting = true;
    fetch("/api/voice/call/start", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (!j.ok) { callStarting = false; voiceMsg = j.error || "The call couldn't start."; render(); return; }
        // The bridge answered and the call is open on its side.
        // Opening the local panel is a separate step: an error in
        // it must never be reported as a bridge failure, so begin()
        // runs outside this chain's rejection path.
        try { begin(); } catch (e) { callStarting = false; if (window.console) console.error(e); }
      }, function () {
        // Only a genuine failure to reach the bridge lands here —
        // and it never banners over a call that has since begun.
        callStarting = false;
        if (!call) voiceMsg = "Could not reach the bridge to start the call.";
        render();
      });
  } else begin();
}
function callNaviTurn(text) {
  if (!call) return;
  call.turns.push({ who: "YOU", text: text, exec: null, ts: Date.now() });
  messages.push({ who: "YOU", text: text, exec: null, ts: Date.now() });
  recordCallTurn("you", text, null);
  persist(); callState = "thinking"; render();
  var done = function (r) {
    if (!call) return;
    call.turns.push({ who: "NAVI", text: r.text, exec: r.exec, sources: r.sources || null, ts: Date.now() });
    messages.push({ who: "NAVI", text: r.text, exec: r.exec, sources: r.sources || null, ts: Date.now() });
    recordCallTurn("navi", r.text, r.exec);
    persist(); callState = "speaking"; render();
    // A CLI-routed turn carries no reply text — what she says on
    // the call is the terminal output itself.
    // Build 6 (#224): the return to listening is unconditional —
    // a rejected speech chain lifts the speaking state too.
    speakSentences(r.text || (r.exec ? r.exec.output : "")).then(function () { if (call) { callState = "listening"; render(); } }, function () { if (call) { callState = "listening"; render(); } });
  };
  // Build 6 (#224): a failed model request is a SNAG, not a
  // reply. It used to be fed through done() as if it were her
  // answer — the panel entered "speaking" for an error sentence
  // and, when the speech chain then hung, never left it. The
  // snag path leaves the speaking/thinking state at once, shows
  // the reason in the panel, and files the turn in the thread
  // and the call record as what it was: unanswered.
  var fail = function (e) {
    if (!call) return;
    var why = (e && e.message) ? e.message : "the model didn't answer";
    var note = "— that turn didn't get an answer (" + why + ") —";
    call.turns.push({ who: "NAVI", text: note, exec: null, ts: Date.now() });
    messages.push({ who: "NAVI", text: note, exec: null, ts: Date.now() });
    recordCallTurn("navi", note, null);
    callState = "listening";
    voiceMsg = "That one didn't get an answer — " + why + " The call is still open: say it again, or type below.";
    persist(); render();
  };
  if (CONFIG.backend === "live") {
    if (llama.status !== "running" && !/^\s*\/chip\b/i.test(text)) {
      // No model home: on a call her fallback is the same as typed
      // chat's — what you say runs as a command and she reads the
      // system output back to you.
      var cmd = text.replace(/^\$\s*/, "");
      liveShell(cmd).then(function (r) { done({ text: "Here's what that gave us:", exec: r }); })
        .catch(function () { done({ text: "That didn't run — the bridge didn't answer.", exec: null }); });
    } else {
      liveChat(text).then(done, fail);
    }
  } else {
    setTimeout(function () { done(mockChat(text)); }, 450);
  }
}
function endCall() {
  if (!call) return;
  var finished = call;
  var secs = callSeconds;
  call = null; callState = "idle"; callSeconds = 0;
  if (callTimer) { clearInterval(callTimer); callTimer = null; }
  stopPtt();
  try { if ("speechSynthesis" in window) speechSynthesis.cancel(); } catch (e) {}
  var finish = function () {
    var n = finished.turns.filter(function (t) { return t.who === "YOU"; }).length;
    messages.push({ who: "NAVI", ts: Date.now(), text: "— voice call ended · " + n + " exchange" + (n === 1 ? "" : "s") + " over " + fmtCallTime(secs) + " — the call is recorded in voice/calls.json and stays part of this conversation.", exec: null });
    persist(); refreshVoiceCalls(); render();
  };
  if (CONFIG.backend === "live") {
    fetch("/api/voice/call/end", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).then(finish).catch(finish);
  } else finish();
}
function blobToB64(blob) {
  return new Promise(function (resolve, reject) {
    var reader = new FileReader();
    reader.onload = function () { var s = String(reader.result || ""); resolve(s.slice(s.indexOf(",") + 1)); };
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}
function markPttButtons() {
  var btns = document.querySelectorAll("[data-ptt]");
  btns.forEach(function (b) {
    if (pttRecording) { b.classList.add("rec"); b.textContent = "● Listening… click to send"; }
    else if (b.id === "call-ptt") { b.classList.remove("rec"); b.textContent = "Push to talk"; }
    else { b.classList.remove("rec"); b.innerHTML = micSvg(); }
  });
}
function reportMicState() {
  // Build 6 (#6, telemetry half): every push-to-talk attempt
  // reports this window's own mic reading to the bridge, which
  // logs it — the reading the Voice tab shows has existed only
  // on this screen since 2026-11-07, which is exactly why the
  // app-window capture question stayed open. Fire and forget:
  // a failed report changes nothing about the attempt.
  if (CONFIG.backend !== "live") return;
  try {
    fetch("/api/voice/mic-report", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ state: micState }) }).catch(function () {});
  } catch (e) {}
}
function startPtt(target) {
  if (!navigator.mediaDevices || !window.MediaRecorder) { micState = "unavailable"; reportMicState(); voiceMsg = "This browser can't record audio — type what you'd say instead."; render(); return; }
  navigator.mediaDevices.getUserMedia({ audio: true }).then(function (stream) {
    micState = "ok"; reportMicState();
    mediaStream = stream; mediaChunks = []; pttTarget = target; pttRecording = true;
    mediaRecorder = new MediaRecorder(stream);
    mediaRecorder.ondataavailable = function (e) { if (e.data && e.data.size) mediaChunks.push(e.data); };
    mediaRecorder.onstop = finishPtt;
    mediaRecorder.start();
    markPttButtons();
  }).catch(function () { micState = "refused"; reportMicState(); voiceMsg = "Microphone access was refused — Navi can't hear you until the mic is allowed. In the standalone window, check the OS privacy/sound settings for Apex (her shell grants the dashboard's own mic requests; the OS prompt is the gate). In a browser, allow the mic for this page. Or type instead."; render(); });
}
function stopPtt() {
  if (mediaRecorder && mediaRecorder.state !== "inactive") { try { mediaRecorder.stop(); } catch (e) {} }
  else if (mediaStream) { mediaStream.getTracks().forEach(function (t) { t.stop(); }); mediaStream = null; }
}
function finishPtt() {
  var target = pttTarget; pttTarget = null; pttRecording = false;
  if (mediaStream) { mediaStream.getTracks().forEach(function (t) { t.stop(); }); mediaStream = null; }
  var blob = new Blob(mediaChunks, { type: mediaRecorder && mediaRecorder.mimeType ? mediaRecorder.mimeType : "audio/webm" });
  mediaChunks = [];
  if (!blob.size) { render(); return; }
  if (CONFIG.backend !== "live") { voiceMsg = "Mock mode simulates the call — type what you'd say; real transcription runs through the live bridge."; render(); return; }
  voiceMsg = "Transcribing…"; if (view === "Settings") render();
  blobToB64(blob).then(function (b64) {
    return fetch("/api/voice/stt", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ audio_b64: b64 }) }).then(function (r) { return r.json(); });
  }).then(function (j) {
    voiceMsg = "";
    if (!j.ok) { voiceMsg = j.error || "Transcription failed."; render(); return; }
    var text = (j.text || "").trim();
    if (!text) { voiceMsg = "I didn't catch any words in that — try again, a little closer to the mic."; render(); return; }
    if (target === "call" && call) callNaviTurn(text);
    else if (target === "chat") { if (call) callNaviTurn(text); else sendChat(text); }
    else render();
  }).catch(function () { voiceMsg = "Transcription failed — the bridge didn't answer."; render(); });
}
function pttToggle(target) {
  if (pttRecording) { stopPtt(); return; }
  reportMicState();
  startPtt(target);
}
function applyBackgroundUpdate(u) {
  // Background re-check (build .86): the bridge re-checks on its
  // own while it runs and /api/status carries the verdict. It
  // folds into the Updater's PASSIVE states only — a check in
  // flight, an apply, or a finished/failed state on screen is
  // never overridden by the background word.
  if (!u || CONFIG.backend !== "live") return;
  if (updater.state === "checking" || updater.state === "applying" || updater.state === "updated" || updater.state === "failed" || updater.state === "applyfailed") return;
  if (u.updateAvailable && u.latest) {
    if (updater.state !== "available" || updater.latest !== u.latest) {
      updater.state = "available";
      updater.latest = u.latest;
      updater.bgTag = u.latest;
      if (view === "Settings" && settingsTab === "system") render();
    }
  } else if (updater.state === "available" && updater.bgTag && updater.bgTag === updater.latest) {
    // The background word now reports no update: a stale
    // "available" for that same tag clears.
    updater.state = "uptodate";
    updater.bgTag = "";
    if (view === "Settings" && settingsTab === "system") render();
  }
}
function refreshUpdaterUi() {
  if (view === "Settings" && settingsTab === "system") render();
  else renderStatus(null);
}
function checkForUpdates() {
  if (CONFIG.backend !== "live") return;
  if (updater.state === "checking" || updater.state === "applying") return;
  updater.state = "checking";
  updater.error = "";
  refreshUpdaterUi();
  fetch("/api/update/check?channel=" + encodeURIComponent(updater.channel)).then(function (r) { return r.json(); }).then(function (j) {
    if (!j || j.ok !== true) {
      updater.state = "failed";
      updater.error = (j && j.error) || "The bridge answered the update check with an unreadable response.";
    } else {
      updater.latest = j.latest || "";
      updater.releaseName = j.releaseName || "";
      updater.releaseUrl = j.releaseUrl || "";
      updater.publishedAt = j.publishedAt || "";
      updater.assetName = (j.asset && j.asset.name) || "";
      updater.assetSize = (j.asset && j.asset.size) || 0;
      // The release notes ride the check result (issue #47):
      // the panel below the version line renders them before
      // the operator ever presses Download & install, and no
      // second fetch happens at Apply time.
      updater.notes = j.notes || "";
      updater.notesSource = j.notesSource || "";
      if (j.channel) updater.channel = j.channel;
      if (typeof j.channelExplicit === "boolean") updaterChannelExplicit = j.channelExplicit;
      updater.prerelease = !!j.prerelease;
      // Edge build sequence (build .86): the candidate's Build
      // number and this build's own, from the check payload.
      updater.buildSeq = (typeof j.buildSeq === "number") ? j.buildSeq : null;
      if (typeof j.currentBuildSeq === "number") updater.currentBuildSeq = j.currentBuildSeq;
      // The running version's own release page (build .83):
      // the "About this release" dropdown renders it — name,
      // tag, date, link, and the full patch notes.
      updater.currentRelease = j.currentRelease || null;
      // A just-confirmed update keeps its confirmation when the
      // check agrees we're current — the "Updated" line is the
      // news; "up to date" would only bury it.
      updater.state = j.updateAvailable ? "available" : (updater.state === "updated" ? "updated" : "uptodate");
    }
    refreshUpdaterUi();
  }).catch(function () {
    updater.state = "failed";
    updater.error = "The bridge didn't answer — the update check runs through it.";
    refreshUpdaterUi();
  });
}
function applyUpdate() {
  if (CONFIG.backend !== "live" || updater.state !== "available") return;
  updater.state = "applying";
  updater.step = "downloading";
  updater.stageError = "";
  updater.applyError = "";
  render();
  fetch("/api/update/apply", { method: "POST", headers: { "Content-Type": "application/json" }, body: "{}" }).then(function (r) { return r.json(); }).then(function (j) {
    if (!j || j.ok !== true) {
      updater.state = "available";
      updater.stageError = (j && j.error) || "The update could not be started — the bridge answered with an unreadable response.";
      render();
    } else {
      pollUpdateResult();
    }
  }).catch(function () {
    // The bridge may already be going down for the restart — the
    // record poll is the source of truth either way.
    pollUpdateResult();
  });
}
function pollUpdateResult() {
  if (updater.pollTimer) return;
  var tick = function () {
    updater.pollTimer = null;
    fetchUpdateResult(true);
  };
  updater.pollTimer = setTimeout(tick, 1500);
}
function fetchUpdateResult(keepPolling) {
  fetch("/api/update/result").then(function (r) { return r.json(); }).then(function (j) {
    if (j && j.state && j.state !== "none") applyUpdateRecord(j);
    if (updater.state === "applying" && keepPolling) pollUpdateResult();
    refreshUpdaterUi();
  }).catch(function () {
    // While the app restarts, the bridge does not answer — that
    // silence is the restart happening, not a failure. Keep
    // polling; the relaunched bridge serves the same record.
    if (updater.state === "applying" && keepPolling) pollUpdateResult();
  });
}
function applyUpdateRecord(j) {
  if (j.beforeLabel) updater.fromLabel = j.beforeLabel;
  if (j.stagedDir) updater.stagedDir = j.stagedDir;
  if (j.command) updater.command = j.command;
  if (j.diagnoseCommand) updater.diagnoseCommand = j.diagnoseCommand;
  if (j.dhdPrereqs) updater.dhdPrereqs = j.dhdPrereqs;
  if (j.tailFailure) updater.tailFailure = j.tailFailure;
  if (j.state === "working") {
    updater.state = "applying";
    // When the current step started (build .81, #70): the
    // progress card's "Taking a while?" note keys on how long
    // the Installing step has been running.
    if ((j.step || "downloading") !== updater.step) updater.stepSince = Date.now();
    updater.step = j.step || "downloading";
  } else if (j.state === "done") {
    updater.state = "updated";
    updater.step = "";
    if (j.afterLabel) updater.toLabel = j.afterLabel;
    if (updater.pollTimer) { clearTimeout(updater.pollTimer); updater.pollTimer = null; }
  } else if (j.state === "failed") {
    // A failure from an apply this page started — or, on the
    // relaunched app's first paint, the failure that the restart
    // interrupted: it surfaces while the startup check is still
    // running, or whenever the release on offer is the very one
    // that failed. A stale failure for a different release never
    // buries a fresh check's offer.
    if (updater.state === "applying" || updater.state === "applyfailed" || updater.state === "idle" || updater.state === "updated" || updater.state === "checking" || (j.tag && updater.latest && j.tag === updater.latest)) {
      updater.state = "applyfailed";
      updater.step = j.step || "";
      updater.applyError = j.error || "The update failed at a step it did not name.";
    }
    if (updater.pollTimer) { clearTimeout(updater.pollTimer); updater.pollTimer = null; }
  }
}
var UPDATE_STEPS = [["downloading", "Downloading"], ["verifying", "Verifying"], ["staging", "Staging"], ["installing", "Installing"], ["dhd-prereqs", "DHD prerequisites"], ["restarting", "Restarting"]];
function updaterProgressHtml() {
  var cur = 0;
  UPDATE_STEPS.forEach(function (s, i) { if (s[0] === updater.step) cur = i; });
  var parts = UPDATE_STEPS.map(function (s, i) {
    if (i < cur) return "<span style='color:#34d399'>✓ " + s[1] + "</span>";
    if (i === cur) return "<b style='color:var(--text)'>" + s[1] + "…</b>";
    return "<span style='color:var(--dim)'>" + s[1] + "</span>";
  });
  // Slim progress bar across the top of the chain card (build
  // .81, #70): the bar is for the glance, the chain below it is
  // for the truth — step receipts, not a bare spinner.
  var pct = Math.round(((cur + 0.5) / UPDATE_STEPS.length) * 100);
  // The rebuild note (build .81, #70): once Installing has been
  // underway a while, say plainly what the time is going on —
  // the UI rebuild — and that it is working, not stuck. No ETA:
  // on modest hardware an honest note beats a lying estimate.
  var slowNote = "";
  if (updater.step === "installing" && updater.stepSince && (Date.now() - updater.stepSince) > 20000) {
    slowNote = "<br><span style='color:var(--dim)'><b>Taking a while?</b> This step is likely rebuilding the UI — that can take a few minutes on modest hardware. It's working, not stuck. We're working on making it faster.</span>";
  }
  return "<div class='card' style='margin-top:10px'>" +
    "<div role='progressbar' aria-label='Update progress' aria-valuemin='0' aria-valuemax='100' aria-valuenow='" + pct + "' style='height:6px;border-radius:999px;background:var(--panel2);overflow:hidden;margin-bottom:8px'><div style='height:100%;width:" + pct + "%;background:var(--cy);transition:width var(--motion-med) ease'></div></div>" +
    "<b>Updating Apex…</b><br>" +
    "<span style='font-size:13px'>" + parts.join(" <span style='color:var(--dim)'>→</span> ") + "</span><br>" +
    "<span style='color:var(--dim)'>Apex will restart itself when the install finishes — this page will go quiet while it does, then come back on the new version with the confirmation.</span>" + slowNote + "</div>";
}
function updaterFallbackHtml() {
  return "<div class='card' style='margin-top:10px'><b>Manual fallback.</b><br>" +
    "<span style='color:var(--dim)'>The update package is staged and unpacked at</span><br><span class='mono' style='font-size:12px'>" + esc(updater.stagedDir) + "</span><br>" +
    "<span style='color:var(--dim)'>One command finishes the job by hand — run this in a terminal:</span>" +
    "<pre class='exec mono' style='margin:8px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-all'>" + esc(updater.command) + "</pre>" +
    (updater.diagnoseCommand ? "<span style='color:var(--dim)'>See what happened:</span>" +
    "<pre class='exec mono' style='margin:8px 0;padding:8px 10px;font-size:12px;white-space:pre-wrap;word-break:break-all'>" + esc(updater.diagnoseCommand) + "</pre>" : "") +
    "<span style='color:var(--dim)'>The installer keeps everything that's yours — your chips, knowledge packs and their indexes, the model file, Soul and Memory, her voice, and your GitHub settings — and Apex restarts into the new version when it finishes.</span></div>";
}
// --- Quick Sync card (0.2.0e Build 3, #176) -------------------------
// The Sync control in Utilities opens nothing: it pops this
// card and runs the round (pull + converge + push over the backup
// repo, on the Operator identity). Essentials only — the state,
// what moved, the route actually used, at most one action. A
// success dismisses itself after five seconds (tap-away sooner);
// a held, failed, or Needs-your-eyes card NEVER auto-dismisses.
var syncCardState = { last: null, resting: null, timer: null, direction: "" };
function ensureSyncCard() {
  if (document.getElementById("syncCard")) return;
  var shade = document.createElement("div");
  shade.id = "syncShade";
  shade.style.display = "none";
  shade.addEventListener("click", closeSyncCard);
  var card = document.createElement("div");
  card.id = "syncCard";
  card.style.display = "none";
  card.setAttribute("role", "dialog");
  card.setAttribute("aria-label", "Sync");
  card.innerHTML = "<div class='sc-head'><div class='sc-title'>Sync</div><button type='button' class='sc-x' id='scClose' aria-label='Close the sync card'>×</button></div><div id='scBody'></div>";
  document.body.appendChild(shade);
  document.body.appendChild(card);
  document.getElementById("scClose").addEventListener("click", closeSyncCard);
}
function syncAgoText(epoch) {
  if (!epoch) return "";
  var s = Math.max(0, Math.floor(Date.now() / 1000 - epoch));
  if (s < 45) return "just now";
  if (s < 3600) return Math.max(1, Math.round(s / 60)) + " min ago";
  if (s < 86400) return Math.round(s / 3600) + " h ago";
  return Math.round(s / 86400) + " d ago";
}
// --- The seat (v0.3.0e Build 1, #200) ------------------------------
// A window notes itself to the bridge when it opens. Its seat
// state — her memory entries, the operator's picture and badge,
// the things that have always lived in this page's own storage —
// rides up; the bridge merges it into the account's seat state
// and, under the same session-gap + debounce discipline the DHD
// door uses, fires the refresh pull a long-lived bridge would
// otherwise only ever run at its own boot. When a pull lands
// (the sync state's clock advances), the window re-reads its
// chats and cases, so a conversation started at another seat is
// simply there — the desk side of picking it up on the phone,
// and the phone side of picking it up at the desk. The bridge
// answers every note with the converged seat state, and the
// window adopts it.
var seatKey = "w" + Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
var seatAck = { operatorPicture: "", operatorBadge: "", operatorName: "" };
var seatPendingStamps = { pictureAt: 0, badgeAt: 0, nameAt: 0 };
// v0.5.1e (#256): the facts this window has forgotten since the
// last seat push — the bridge records each as a tombstone, so
// the forget lands on every seat and a stale copy cannot
// resurrect it. Cleared only when a push carries them.
var seatForgotten = [];
var seatWatchTimer = null;

function seatFactKey(t) { return String(t || "").toLowerCase().replace(/\s+/g, " ").trim(); }

function seatStateOut() {
  var pic = CONFIG.operatorPicture || "";
  var badge = CONFIG.operatorBadge || "";
  var name = CONFIG.operatorName || "";
  if (pic && pic !== seatAck.operatorPicture && !seatPendingStamps.pictureAt) seatPendingStamps.pictureAt = Date.now() / 1000;
  if (badge && badge !== seatAck.operatorBadge && !seatPendingStamps.badgeAt) seatPendingStamps.badgeAt = Date.now() / 1000;
  if (name && name !== seatAck.operatorName && !seatPendingStamps.nameAt) seatPendingStamps.nameAt = Date.now() / 1000;
  return {
    entries: memory.map(function (m) { return { t: m.t, o: m.o }; }),
    operatorPicture: pic, operatorBadge: badge,
    pictureAt: seatPendingStamps.pictureAt, badgeAt: seatPendingStamps.badgeAt,
    operatorName: name, nameAt: seatPendingStamps.nameAt,
    forgotten: seatForgotten.map(function (t) { return { t: t }; })
  };
}

function seatOrigin(o) {
  // The page's origin vocabulary, with the bridge's legacy
  // stamps folded back into it — an entry's origin is part of
  // the fact, and it survives the road in its own words.
  if (o === "manual" || o === "trigger" || o === "autonomous" || o === "correction") return o;
  if (o === "auto") return "autonomous";
  return "manual";
}

function adoptSeat(seat) {
  if (!seat || typeof seat !== "object") return;
  var changed = false;
  if (Array.isArray(seat.entries)) {
    // v0.5.1e (#256): the bridge's seat state is the truth and
    // this list is its cache. Every adopt answers a push that
    // carried this window's whole list up first, so the
    // converged answer may REPLACE the cache outright —
    // arrivals land, and a fact forgotten on another seat
    // leaves here too, instead of unioning forever under a
    // stale local copy. (Through v0.4.10e the adopt only ever
    // added, which is why each seat kept its own list.)
    var converged = [];
    seat.entries.forEach(function (e) {
      if (e && e.t) converged.push({ t: String(e.t), o: seatOrigin(e.o) });
    });
    var before = JSON.stringify(memory.map(function (m) { return [m.t, m.o]; }));
    var after = JSON.stringify(converged.map(function (m) { return [m.t, m.o]; }));
    if (before !== after) { memory = converged; changed = true; }
  }
  if (typeof seat.operatorName === "string" && seat.operatorName && seat.operatorName !== (CONFIG.operatorName || "")) {
    // The name is a single value, last-writer-wins by stamp —
    // adopt the converged name unless this window holds a
    // newer unsent change of its own.
    if (!seatPendingStamps.nameAt || (typeof seat.nameAt === "number" && seat.nameAt >= seatPendingStamps.nameAt)) {
      CONFIG.operatorName = seat.operatorName; seatPendingStamps.nameAt = 0; changed = true;
    }
  }
  if (seat.operatorName && seat.operatorName === (CONFIG.operatorName || "")) { seatAck.operatorName = seat.operatorName; seatPendingStamps.nameAt = 0; }
  if (typeof seat.operatorPicture === "string" && seat.operatorPicture && seat.operatorPicture !== (CONFIG.operatorPicture || "")) { CONFIG.operatorPicture = seat.operatorPicture; changed = true; }
  if (typeof seat.operatorBadge === "string" && seat.operatorBadge && seat.operatorBadge !== (CONFIG.operatorBadge || "")) { CONFIG.operatorBadge = seat.operatorBadge; changed = true; }
  if (seat.operatorPicture) seatAck.operatorPicture = seat.operatorPicture;
  if (seat.operatorBadge) seatAck.operatorBadge = seat.operatorBadge;
  if (seatPendingStamps.pictureAt && seatAck.operatorPicture === (CONFIG.operatorPicture || "")) seatPendingStamps.pictureAt = 0;
  if (seatPendingStamps.badgeAt && seatAck.operatorBadge === (CONFIG.operatorBadge || "")) seatPendingStamps.badgeAt = 0;
  if (changed) { saveCfg(); persist(); render(); }
}

var seatPushTimer = null;
var seatPushSig = "";
function seatWriteThrough() {
  if (CONFIG.backend !== "live") return;
  if (seatPushTimer) return;
  seatPushTimer = setTimeout(function () {
    seatPushTimer = null;
    var out = seatStateOut();
    var sig = JSON.stringify([out.entries, out.operatorName, out.operatorPicture ? "pic" : "", out.operatorBadge ? "badge" : ""]);
    if (sig === seatPushSig && !seatForgotten.length) return;
    var sentForgotten = seatForgotten.length;
    fetch("/api/sync/seat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ seat: out, seatKey: seatKey }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        seatPushSig = sig;
        // Only the forgets this push actually carried are
        // cleared — one recorded mid-flight rides the next.
        seatForgotten.splice(0, sentForgotten);
        if (seatPendingStamps.nameAt && seatAck.operatorName === (CONFIG.operatorName || "")) seatPendingStamps.nameAt = 0;
        if (j && j.seat) adoptSeat(j.seat);
      }).catch(function () {});
  }, 900);
}

function noteSeat() {
  if (CONFIG.backend !== "live") return;
  fetch("/api/sync").then(function (r) { return r.json(); }).then(function (before) {
    var epoch0 = (before && before.lastSyncEpoch) || 0;
    fetch("/api/sync/seat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ seat: seatStateOut(), seatKey: seatKey }) })
      .then(function (r) { return r.json(); })
      .then(function (j) {
        if (j && j.seat) adoptSeat(j.seat);
        // Watch the sync clock for the pull this note set in
        // motion: a handful of checks over a few minutes, then
        // stop — a window, not a polling daemon.
        var tries = 0;
        if (seatWatchTimer) clearInterval(seatWatchTimer);
        seatWatchTimer = setInterval(function () {
          tries += 1;
          if (tries > 9) { clearInterval(seatWatchTimer); seatWatchTimer = null; return; }
          fetch("/api/sync").then(function (r) { return r.json(); }).then(function (s) {
            if (s && (s.lastSyncEpoch || 0) > epoch0) {
              clearInterval(seatWatchTimer); seatWatchTimer = null;
              hydrateChatsFromServer();
              casesState = null; caseDetailState = null;
              fetch("/api/sync/seat", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ seat: seatStateOut(), seatKey: seatKey }) })
                .then(function (r) { return r.json(); })
                .then(function (j2) { if (j2 && j2.seat) adoptSeat(j2.seat); })
                .catch(function () {});
              render();
            }
          }).catch(function () {});
        }, 20000);
      }).catch(function () {});
  }).catch(function () {});
}

function openSyncCard() {
  ensureSyncCard();
  document.getElementById("syncShade").style.display = "block";
  document.getElementById("syncCard").style.display = "block";
  if (CONFIG.backend !== "live") { runSyncRound(""); return; }
  // v0.6.2e (#303): the override — with the setting on (its
  // default), a press offers Send / Receive before anything
  // moves; with it off, the press runs the legacy full round
  // at once, exactly as it always did. The setting is the
  // bridge's word, read fresh on every open.
  fetch("/api/sync").then(function (r) { return r.json(); }).then(function (j) {
    syncCardState.resting = j;
    if (j && j.choiceOnPress === false) runSyncRound("");
    else renderSyncCard({ state: "choice" });
  }).catch(function () { renderSyncCard({ state: "choice" }); });
}
// The override setting's page half (v0.6.2e, #303): the
// bridge's stored choice, fetched from GET /api/sync and
// flipped only through its settings door — the page never
// holds a private copy of the truth.
var syncChoiceState = null, syncChoiceFetching = false, syncChoiceMsg = "";
function fetchSyncChoice() {
  if (CONFIG.backend !== "live") { syncChoiceState = true; return; }
  syncChoiceFetching = true;
  fetch("/api/sync").then(function (r) { return r.json(); }).then(function (j) {
    syncChoiceFetching = false;
    if (j && typeof j.choiceOnPress === "boolean") { syncChoiceState = j.choiceOnPress; if (view === "Settings") render(); }
  }).catch(function () { syncChoiceFetching = false; });
}
function syncChoiceFlip() {
  var want = !(syncChoiceState !== false);
  if (CONFIG.backend !== "live") { syncChoiceState = want; syncChoiceMsg = ""; render(); return; }
  fetch("/api/settings/sync-choice", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ choiceOnPress: want }) })
    .then(function (r) { return r.json(); })
    .then(function (j) {
      if (j && j.ok) { syncChoiceState = j.choiceOnPress; syncChoiceMsg = ""; }
      else syncChoiceMsg = (j && j.error) || "The bridge refused the change — the setting stands as it was.";
      render();
    })
    .catch(function () { syncChoiceMsg = "The bridge did not answer — the setting stands as it was."; render(); });
}
function closeSyncCard() {
  if (syncCardState.timer) { clearTimeout(syncCardState.timer); syncCardState.timer = null; }
  var shade = document.getElementById("syncShade");
  var card = document.getElementById("syncCard");
  if (shade) shade.style.display = "none";
  if (card) card.style.display = "none";
}
function runSyncRound(direction) {
  if (syncCardState.timer) { clearTimeout(syncCardState.timer); syncCardState.timer = null; }
  syncCardState.direction = direction || "";
  if (CONFIG.backend !== "live") {
    renderSyncCard({ state: "mock" });
    return;
  }
  renderSyncCard({ state: "working", direction: syncCardState.direction });
  var body = { openChatId: (typeof activeChatId !== "undefined" ? activeChatId : ""), seat: seatStateOut() };
  if (syncCardState.direction) body.direction = syncCardState.direction;
  fetch("/api/sync/run", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body) }).then(function (r) { return r.json(); }).then(function (j) {
    syncCardState.last = j;
    if (j && j.seat) adoptSeat(j.seat);
    renderSyncCard(j || { state: "held", error: "The bridge answered without a verdict — the round is held." });
    if (j && (j.state === "partial" || (j.ok && ((j.pulledChats || 0) > 0 || (j.caseFilesPulled || 0) > 0 || (j.deckFilesPulled || 0) > 0 || (j.docsPulled || 0) > 0 || (j.entriesPulled || 0) > 0 || (j.cardsImported || 0) > 0 || j.seatChanged || (j.needsEyes || []).length)))) {
      hydrateChatsFromServer();
      if (typeof view !== "undefined" && view === "CaseFiles") { casesState = null; caseDetailState = null; render(); }
    }
  }).catch(function () {
    renderSyncCard({ state: "held", error: "Can't reach the bridge — the round is held; the next trigger retries it." });
  });
}
function syncMovedHtml(j) {
  function n(count, one, many) { return count + " " + (count === 1 ? one : many); }
  var bits = [];
  if (j.pulledChats) bits.push(n(j.pulledChats, "chat", "chats") + " pulled");
  if (j.pushedChats) bits.push(n(j.pushedChats, "chat", "chats") + " pushed");
  if (j.filesWritten) bits.push(n(j.filesWritten, "file", "files") + " published");
  if (j.caseFilesPulled) bits.push(n(j.caseFilesPulled, "case file", "case files") + " pulled");
  if (j.deckFilesPulled) bits.push(n(j.deckFilesPulled, "deck file", "deck files") + " pulled");
  if (j.cardsImported) bits.push(n(j.cardsImported, "card", "cards") + " taken in");
  if (j.docsPulled) bits.push(n(j.docsPulled, "document", "documents") + " pulled");
  if (j.entriesPulled) bits.push(n(j.entriesPulled, "memory entry", "memory entries") + " pulled");
  if (j.entriesRemoved) bits.push(n(j.entriesRemoved, "memory entry", "memory entries") + " forgotten");
  if (j.seatChanged) bits.push("your picture or badge updated");
  if (j.removed) bits.push(n(j.removed, "deletion", "deletions") + " carried");
  var s = bits.length ? bits.join(" · ") : "Nothing moved — already in step.";
  if (j.openIncluded) s += "<br>The thread on screen is in this round.";
  return s;
}
function renderSyncCard(j) {
  ensureSyncCard();
  var body = document.getElementById("scBody");
  if (!body) return;
  if (syncCardState.timer) { clearTimeout(syncCardState.timer); syncCardState.timer = null; }
  var h = "";
  if (j.state === "choice") {
    h = "<div class='sc-state'>Sync — your call</div><div class='sc-moved'>Move now, in the direction you name — not at the cadence's pace. The full round still runs on its own occasions either way.</div>" +
      "<div class='row' style='margin-top:8px;gap:8px'><button type='button' class='sc-act' data-sync-dir='send' style='min-height:44px'>Send a sync</button><button type='button' class='sc-act' data-sync-dir='receive' style='min-height:44px'>Receive a sync</button></div>" +
      "<div class='sc-moved' style='margin-top:8px'><b>Send</b> publishes this seat's set and pulls nothing down. <b>Receive</b> pulls the converged set and publishes nothing.</div>";
  } else if (j.state === "working") {
    if (j.direction === "send") h = "<div class='sc-state'>Sending…</div><div class='sc-moved'>Publishing this seat's set — nothing comes down.</div>";
    else if (j.direction === "receive") h = "<div class='sc-state'>Receiving…</div><div class='sc-moved'>Pulling the converged set — nothing goes up.</div>";
    else h = "<div class='sc-state'>Syncing…</div><div class='sc-moved'>Pulling, converging, pushing — one round, both ways.</div>";
  } else if (j.state === "mock") {
    h = "<div class='sc-state'>Sync runs over the live bridge</div><div class='sc-moved'>This window is in Mock backend mode (Settings → System), so no round runs from here. Switch to Live and one tap moves your chats, Case Files, the ChipDeck, her Soul and memory, and your seat between your machines.</div>";
  } else if (j.state === "synced") {
    var dirWord = j.direction === "send" ? "Sent" : (j.direction === "receive" ? "Received" : "Synced");
    var moved = (j.direction && j.message) ? esc(j.message) : syncMovedHtml(j);
    h = "<div class='sc-state'>" + dirWord + " · just now</div><div class='sc-moved'>" + moved + "</div>";
  } else if (j.state === "eyes") {
    h = "<div class='sc-state' style='color:#f5b544'>Needs your eyes</div><div class='sc-moved'>" + esc(j.message || "A pair parked both copies.") + "</div><div class='sc-moved'>Both copies are kept — nothing was overwritten. Open the thread or case to settle it.</div>";
  } else if (j.state === "superseded") {
    // v0.6.3e (#317): this round was overridden by the
    // operator's own newer choice — the card says the override
    // landed, never that the round was "held" behind itself.
    h = "<div class='sc-state'>Overridden — your call runs now</div><div class='sc-moved'>" + esc(j.message || "This round stopped at a phase boundary; the direction you chose runs instead.") + "</div>";
  } else if (j.state === "partial") {
    // v0.6.3e (#318): the verdict honesty gate — the main
    // thread disagrees, and the card names it instead of
    // printing a clean "converged" over it.
    h = "<div class='sc-state' style='color:#f5b544'>Mostly synced</div><div class='sc-moved'>" + esc(j.message || "The main thread disagrees with the converged set.") + "</div><button type='button' class='sc-act' id='scRetry'>Retry</button>";
  } else if (j.state === "auth") {
    h = "<div class='sc-state' style='color:#f5b544'>Held — sign-in needed</div><div class='sc-moved'>" + esc(j.error || "Not signed in to GitHub.") + "</div>" +
      "<button type='button' class='sc-act' id='scSignIn'>Sign in to GitHub</button>";
  } else {
    h = "<div class='sc-state' style='color:#f5b544'>Held</div><div class='sc-moved'>" + esc(j.error || "The round could not finish.") + "</div>" +
      "<button type='button' class='sc-act' id='scRetry'>Retry</button>";
  }
  h += "<div class='sc-route'>Route: " + esc(j.route || (syncCardState.resting && syncCardState.resting.route) || "Backup repo") + "</div>";
  if (syncCardState.resting && syncCardState.resting.lastSyncEpoch && j.state !== "synced") {
    h += "<div class='sc-route'>Last round: " + esc(syncAgoText(syncCardState.resting.lastSyncEpoch)) + (syncCardState.resting.lastResult ? " — " + esc(syncCardState.resting.lastResult) : "") + "</div>";
  }
  body.innerHTML = h;
  body.querySelectorAll("[data-sync-dir]").forEach(function (b) {
    b.addEventListener("click", function () { runSyncRound(b.getAttribute("data-sync-dir")); });
  });
  var retry = document.getElementById("scRetry");
  if (retry) retry.addEventListener("click", function () { runSyncRound(syncCardState.direction); });
  var signin = document.getElementById("scSignIn");
  if (signin) signin.addEventListener("click", function () { closeSyncCard(); goView("Settings"); });
  if (j.state === "synced") {
    // Alex's ruling: a success lingers five seconds, then
    // dismisses itself. Held and Needs-your-eyes cards stay
    // until they are dealt with — this timer is armed ONLY here.
    syncCardState.timer = setTimeout(closeSyncCard, 5000);
  }
}
// --- Notifications (v0.4.7e, #242) ---------------------------------
// The bridge keeps the list and the read state; this page holds
// only the last payload it was handed — on /api/status, on
// GET /api/notifications, and as the answer to every read POST,
// all the one shape: {unread, items}, unread first, newest
// first. The bell's badge, the panel, and Home's Needs-your-eyes
// card all paint from this one state, so the surfaces can never
// tell different stories.
var notifState = { unread: 0, items: [] };
var notifSig = "";
var notifPanelOpen = false;
function notifUnreadSig() {
  return (notifState.items || []).filter(function (it) { return !it.read; }).map(function (it) { return String(it.id); }).join(",") + ":" + notifState.unread;
}
function paintNotifBadge() {
  var badge = document.getElementById("notif-badge");
  if (!badge) return;
  badge.textContent = notifState.unread > 99 ? "99+" : String(notifState.unread || 0);
  badge.style.display = notifState.unread > 0 ? "" : "none";
  var bell = document.getElementById("notif-bell");
  if (bell) bell.setAttribute("aria-label", notifState.unread > 0 ? "Notifications \u2014 " + notifState.unread + " unread" : "Notifications");
}
function applyNotifications(p) {
  if (!p || typeof p !== "object") return;
  var prevSig = notifSig;
  notifState = {
    unread: typeof p.unread === "number" ? p.unread : (p.items || []).filter(function (it) { return !it.read; }).length,
    items: Array.isArray(p.items) ? p.items : []
  };
  notifSig = notifUnreadSig();
  paintNotifBadge();
  if (notifPanelOpen) renderNotifPanel();
  // Home's eyes card is a resident of the board: when the unread
  // set changes under an operator standing on Home, re-render so
  // the card is fresh rather than a stale count.
  if (prevSig !== notifSig && view === "ThreatIntel" && intelTab === "home") render();
}
function notifApplyResponse(j) {
  if (!j || typeof j !== "object") return;
  if (j.notifications) applyNotifications(j.notifications);
  else if (Array.isArray(j.items) || typeof j.unread === "number") applyNotifications(j);
}
function notifFetch() {
  fetch("/api/notifications").then(function (r) { return r.json(); }).then(notifApplyResponse).catch(function () {});
}
function notifMarkRead(body) {
  fetch("/api/notifications/read", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(body || {}) })
    .then(function (r) { return r.json(); }).then(notifApplyResponse).catch(function () {});
}
function notifWhen(created) {
  if (created === null || created === undefined || created === "") return "";
  var n = Number(created);
  if (isFinite(n) && n > 1000000000) { try { return new Date(n * 1000).toLocaleString(); } catch (e) { return ""; } }
  var d = new Date(created);
  if (!isNaN(d.getTime())) { try { return d.toLocaleString(); } catch (e) {} }
  return String(created);
}
function ensureNotifPanel() {
  if (document.getElementById("notifPanel")) return;
  var shade = document.createElement("div");
  shade.id = "notifShade";
  shade.style.display = "none";
  shade.addEventListener("click", closeNotifPanel);
  var panel = document.createElement("div");
  panel.id = "notifPanel";
  panel.style.display = "none";
  panel.setAttribute("role", "dialog");
  panel.setAttribute("aria-label", "Notifications \u2014 needs your eyes");
  panel.innerHTML = "<div class='notif-head'><div class='notif-title'>Needs your eyes</div><button type='button' class='ghost' id='notifMarkAll' style='min-height:44px'>Mark all read</button><button type='button' class='notif-x' id='notifClose' aria-label='Close notifications'>\u00d7</button></div><div id='notifBody'></div>";
  document.body.appendChild(shade);
  document.body.appendChild(panel);
  document.getElementById("notifClose").addEventListener("click", closeNotifPanel);
  document.getElementById("notifMarkAll").addEventListener("click", function () { notifMarkRead({ all: true }); });
  document.addEventListener("keydown", function (e) { if (e.key === "Escape" && notifPanelOpen) closeNotifPanel(); });
}
function renderNotifPanel() {
  var body = document.getElementById("notifBody");
  if (!body) return;
  var items = notifState.items || [];
  // v0.5.14e (#272): the phone door stands at the panel's
  // head, and every row carries its own phone delivery line
  // — the outcome the bridge recorded when the record was
  // born, named, never silent.
  var head = pushDoorHtml();
  if (!items.length) { body.innerHTML = head + "<div class='notif-empty'>Nothing needs your eyes.</div>"; return; }
  body.innerHTML = head + items.map(function (it) {
    return "<div class='notif-row" + (it.read ? " read" : "") + "' data-notif-row='" + esc(it.id) + "'" + (pushFocusId && it.id === pushFocusId ? " style='outline:2px solid var(--ac);outline-offset:-2px'" : "") + "><div class='notif-main'><div class='notif-item-title'>" + esc(it.title || "") + (it.count > 1 ? " <span class='mono' style='font-size:11px;color:var(--dim)'>\u00d7" + esc(it.count) + "</span>" : "") + "</div>" + (it.body ? "<div class='notif-item-body'>" + esc(it.body) + "</div>" : "") + (it.created ? "<div class='notif-when mono'>" + esc(notifWhen(it.created)) + "</div>" : "") + (it.push ? "<div class='notif-when mono'>Phone: " + esc(it.push.detail || it.push.state || "") + "</div>" : "") + "</div><button type='button' class='ghost' data-notif-go='" + esc(it.id) + "' style='min-height:44px'>Go</button></div>";
  }).join("");
  wireNotifications();
}
function openNotifPanel() {
  ensureNotifPanel();
  notifPanelOpen = true;
  document.getElementById("notifShade").style.display = "block";
  document.getElementById("notifPanel").style.display = "block";
  renderNotifPanel();
  pushRefresh();
  if (CONFIG.backend === "live") notifFetch();
}
function closeNotifPanel() {
  notifPanelOpen = false;
  var shade = document.getElementById("notifShade");
  var panel = document.getElementById("notifPanel");
  if (shade) shade.style.display = "none";
  if (panel) panel.style.display = "none";
}
// --- Phone notifications: the tray's doorbell (v0.5.14e, #272) ------
// Parity is the law: whatever earns a tray record rings the
// subscribed phones with the tray record's own summary, sent
// by the operator's own node — the bridge holds the keys,
// signs and encrypts every delivery itself, and the only
// other parties are the platform's own push services. The
// door below names its own state on a four-rung ladder and
// never renders a button that cannot work.
var pushState = { vapidPublicKey: "", subscriptions: [], here: false, hereEndpoint: "", busy: false, note: "" };
var pushFocusId = "";
function pushEnv() {
  var supported = ("serviceWorker" in navigator) && ("PushManager" in window) && ("Notification" in window);
  var installed = false;
  try { installed = window.matchMedia("(display-mode: standalone)").matches || window.navigator.standalone === true; } catch (e) {}
  return { supported: supported, installed: installed,
           permission: ("Notification" in window) ? Notification.permission : "unsupported",
           ios: /iphone|ipad|ipod/i.test(navigator.userAgent || "") };
}
function pushLadderState(env) {
  // The four rungs, by name: a device that cannot be rung
  // says so; an iPhone in a browser tab gets the install
  // card, never a dead button; a blocked permission names
  // itself and its fix; anything else is live.
  if (!env || !env.supported) return "unsupported";
  if (env.ios && !env.installed) return "install-first";
  if (env.permission === "denied") return "denied";
  return "live";
}
function pushDoorHtml() {
  var env = pushEnv();
  var state = pushLadderState(env);
  var h = "<div class='push-door' data-push-state='" + state + "' style='border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-bottom:10px'><div style='font-weight:700;font-size:13px'>Phone notifications</div>";
  if (pushState.note) h += "<div style='font-size:12px;color:var(--dim);margin-top:2px'>" + esc(pushState.note) + "</div>";
  if (state === "unsupported") {
    h += "<p style='font-size:12px;color:var(--dim);margin:6px 0 0'>This browser has no Push API, so it cannot be rung. The tray still holds everything \u2014 it just cannot reach this device. (On iPhone, push exists only for Apex added to the Home Screen and opened from its icon.)</p>";
  } else if (state === "install-first") {
    h += "<p style='font-size:12px;color:var(--dim);margin:6px 0 0'>On this phone, push only exists for the <b>installed</b> app \u2014 a browser tab cannot subscribe, and no setting changes that. Tap <b>Share</b>, then <b>Add to Home Screen</b>, open Apex from the new icon, and this door lights up here.</p>";
  } else if (state === "denied") {
    h += "<p style='font-size:12px;color:var(--dim);margin:6px 0 0'>Notifications are <b>blocked</b> for Apex in this browser's site settings, so this device cannot be rung. Allow notifications there and reopen this panel \u2014 the door lights up on its own. Nothing nags you meanwhile.</p>";
  } else if (pushState.here) {
    h += "<p style='font-size:12px;color:var(--dim);margin:6px 0 0'>This device rings when a tray item lands \u2014 the tray's own summary, on the lock screen, sent by your own node. Tapping it opens the tray at that item; reading the item on any seat clears the phone copy.</p>"
      + "<div style='margin-top:8px'><button type='button' class='ghost' data-push-disable='1' style='min-height:44px'>Stop ringing this device</button></div>";
  } else {
    h += "<p style='font-size:12px;color:var(--dim);margin:6px 0 0'>Ring this device when a tray item lands \u2014 the tray's own summary, on the lock screen, sent by your own node (no relay, no account). Tapping it opens the tray at that item.</p>"
      + "<div style='margin-top:8px'><button type='button' class='act' data-push-enable='1' style='min-height:44px'" + (pushState.busy ? " disabled" : "") + ">Ring this device</button></div>";
  }
  if (pushState.subscriptions && pushState.subscriptions.length) {
    // v0.6.7e (#332): the roster can be told apart and
    // cleaned — each record shows WHEN it subscribed, and
    // any seat can remove a stale one (that record stops
    // ringing; the tray is untouched).
    h += "<div style='font-size:12px;color:var(--dim);margin-top:8px'>Ringing:</div>" + pushState.subscriptions.map(function (s) {
      var when = "";
      try { if (s.created) when = " · since " + new Date(s.created).toLocaleDateString(); } catch (e) {}
      return "<div class='row' style='gap:8px;align-items:center;margin-top:4px'><span style='font-size:12px'>" + esc(s.label || "a phone") + esc(when) + "</span><button type='button' class='ghost' data-push-remove='" + esc(s.id || "") + "' style='min-height:32px;padding:2px 10px;font-size:12px'>Remove</button></div>";
    }).join("");
  }
  return h + "</div>";
}
function pushRenderDoors() {
  if (notifPanelOpen) renderNotifPanel();
  if (view === "Settings") render();
}
function pushRefresh() {
  fetch("/api/push/state").then(function (r) { return r.json(); }).then(function (j) {
    pushState.vapidPublicKey = (j && j.vapidPublicKey) || "";
    pushState.subscriptions = (j && j.subscriptions) || [];
    if (!("serviceWorker" in navigator)) { pushRenderDoors(); return; }
    navigator.serviceWorker.getRegistration().then(function (reg) {
      if (!reg) { pushState.here = false; pushRenderDoors(); return; }
      reg.pushManager.getSubscription().then(function (sub) {
        pushState.here = !!sub;
        pushState.hereEndpoint = sub ? sub.endpoint : "";
        pushRenderDoors();
      }).catch(function () { pushRenderDoors(); });
    }).catch(function () { pushRenderDoors(); });
  }).catch(function () {});
}
function pushUrlB64ToUint8(b64) {
  var pad = "=".repeat((4 - (b64.length % 4)) % 4);
  var raw = atob((b64 + pad).replace(/-/g, "+").replace(/_/g, "/"));
  var out = new Uint8Array(raw.length);
  for (var i = 0; i < raw.length; i++) out[i] = raw.charCodeAt(i);
  return out;
}
function pushDeviceLabel() {
  var ua = navigator.userAgent || "";
  if (/iphone/i.test(ua)) return "iPhone";
  if (/ipad/i.test(ua)) return "iPad";
  if (/android/i.test(ua)) return "Android phone";
  return "This browser";
}
function pushEnable() {
  if (!("serviceWorker" in navigator)) return;
  pushState.busy = true; pushState.note = ""; pushRenderDoors();
  navigator.serviceWorker.register("/sw.js").then(function (reg) {
    return Notification.requestPermission().then(function (perm) {
      if (perm !== "granted") {
        pushState.busy = false;
        pushState.note = "Permission was not granted \u2014 the door stays dark until it is.";
        pushRenderDoors();
        return null;
      }
      return fetch("/api/push/state").then(function (r) { return r.json(); }).then(function (j) {
        pushState.vapidPublicKey = (j && j.vapidPublicKey) || pushState.vapidPublicKey;
        return reg.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey: pushUrlB64ToUint8(pushState.vapidPublicKey) });
      });
    });
  }).then(function (sub) {
    if (!sub) return;
    return fetch("/api/push/subscribe", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ subscription: sub.toJSON(), label: pushDeviceLabel() }) }).then(function (r) { return r.json(); }).then(function (j) {
      pushState.busy = false;
      pushState.note = (j && j.ok) ? "This device will ring for tray items now." : ("The bridge refused the subscription \u2014 " + ((j && j.error) || "unnamed refusal."));
      pushRefresh();
    });
  }).catch(function (e) {
    pushState.busy = false;
    pushState.note = "Subscribing failed on this device \u2014 " + String((e && e.message) || e);
    pushRenderDoors();
  });
}
function pushRemove(id) {
  if (!id) return;
  fetch("/api/push/remove", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ id: id }) }).then(function (r) { return r.json(); }).then(function (j) {
    pushState.note = (j && j.ok) ? "That record was removed \u2014 it will not ring again. The tray is untouched." : ("The removal was refused \u2014 " + ((j && j.error) || "unnamed refusal."));
    pushRefresh();
  }).catch(function (e) {
    pushState.note = "Removing that record failed \u2014 " + String((e && e.message) || e);
    pushRenderDoors();
  });
}
function pushDisable() {
  if (!("serviceWorker" in navigator)) return;
  navigator.serviceWorker.getRegistration().then(function (reg) {
    if (!reg) return null;
    return reg.pushManager.getSubscription();
  }).then(function (sub) {
    if (!sub) { pushRefresh(); return; }
    var endpoint = sub.endpoint;
    return fetch("/api/push/unsubscribe", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ endpoint: endpoint }) }).then(function () {
      return sub.unsubscribe();
    }).then(function () {
      pushState.here = false;
      pushState.note = "This device no longer rings. The tray is untouched.";
      pushRefresh();
    });
  }).catch(function (e) {
    pushState.note = "Unsubscribing failed on this device \u2014 " + String((e && e.message) || e);
    pushRenderDoors();
  });
}
document.addEventListener("click", function (e) {
  var b = (e.target && e.target.closest) ? e.target.closest("[data-push-enable],[data-push-disable],[data-push-remove]") : null;
  if (!b) return;
  if (b.getAttribute("data-push-enable")) pushEnable();
  else if (b.getAttribute("data-push-remove")) pushRemove(b.getAttribute("data-push-remove"));
  else pushDisable();
});
window.addEventListener("load", function () {
  if ("serviceWorker" in navigator) { try { navigator.serviceWorker.register("/sw.js").catch(function () {}); } catch (e) {} }
  var m = /#notice=([A-Za-z0-9:._\-]+)/.exec(location.hash || "");
  if (m) { pushFocusId = m[1]; setTimeout(function () { openNotifPanel(); }, 500); }
  pushRefresh();
});

function notifGo(item) {
  if (!item) return;
  // Going is seeing: the item is marked read on the way out, and
  // the fresh payload the POST returns repaints badge and card.
  notifMarkRead({ ids: [item.id] });
  closeNotifPanel();
  var t = item.target || {};
  if (t.view === "FediChat") {
    // v0.5.4e (#267): a FediChat tray record lands IN the
    // room it names — the room is selected before the view
    // opens, and the room's state refreshes on the way in.
    if (t.fedinetId) fcFed = t.fedinetId;
    fcRoomId = t.roomId || "";
    goView("FediChat");
    refreshFedichat(false);
    return;
  }
  if (t.view === "Federation") { goView("Federation"); return; }
  if (t.view === "DHD") { goView("DHD"); return; }
  if (t.view === "Settings") { goView("Settings"); return; }
  if (t.view === "CaseFiles") {
    goView("CaseFiles");
    if (t.case) {
      // The case opens through the case page's own mechanism —
      // refreshCaseDetail, the same road the list's
      // [data-case-open] buttons walk. If the loaded case list
      // knows this slug is gone, the landing on Case Files is
      // the whole answer; nothing is faked open.
      var known = casesState ? (casesState || []).filter(function (c) { return c.slug === t.case; })[0] : null;
      if (!casesState || known) refreshCaseDetail(t.case);
    }
    return;
  }
  if (t.view) goView(t.view);
}
function homeEyesHtml() {
  // Home's Needs-your-eyes card (v0.4.7e, #242): the first
  // resident of the board whenever anything is unread — up to
  // four items, title and a one-line body, each with its Go —
  // and nothing at all when the list is quiet.
  var unread = (notifState.items || []).filter(function (it) { return !it.read; });
  if (!unread.length) return "";
  var h = "<div class='card eyes-card'><div class='eyes-title'>Needs your eyes</div>";
  unread.slice(0, 4).forEach(function (it) {
    h += "<div class='eyes-row'><div class='eyes-main'><b>" + esc(it.title || "") + "</b>" + (it.count > 1 ? " <span class='mono' style='font-size:11px;color:var(--dim)'>\u00d7" + esc(it.count) + "</span>" : "") + "<div class='eyes-body'>" + esc(it.body || "") + "</div></div><button type='button' class='ghost' data-notif-go='" + esc(it.id) + "' style='min-height:44px'>Go</button></div>";
  });
  if (unread.length > 4) h += "<div class='eyes-more'>+ " + (unread.length - 4) + " more \u2014 the bell carries them all.</div>";
  return h + "</div>";
}
function wireNotifications() {
  // The wireFederation() idiom (v0.4.7e, #242): bind every Go
  // button wherever it exists — the Home eyes card rides mainEl,
  // the panel rides <body> — by resolving its id against
  // notifState at click time. The bell binds once, at creation.
  document.querySelectorAll("[data-notif-go]").forEach(function (b) {
    b.onclick = function () {
      var id = b.getAttribute("data-notif-go");
      var it = (notifState.items || []).filter(function (x) { return String(x.id) === String(id); })[0];
      if (it) notifGo(it);
    };
  });
}
// --- Chat tools list (v0.4.7e, #246, final shape ruled
// 2026-11-16) ------------------------------------------------------
// On a phone the composer bar is the input and one burger: the
// three tools — New case, Attach, Call — live in the list the
// burger opens. The rows are built at OPEN time from the buttons
// actually in the DOM, and choosing a row closes the list and
// performs the act by clicking the real button, so the existing
// handlers (the #241 New-case form, the file picker, the call)
// do the work exactly as they always have. The composer mic is
// REMOVED on the phone surface altogether — not in the bar, not
// a row in this list (the bar's #chat-mic is display:none at
// phone widths and no row stands for it): on a phone, voice is
// the call card's own Push to talk while a call is live, and
// that control (data-ptt='call') belongs to the call surface
// and is never touched here. Desktop never sees any of it.
var chatToolsOpen = false;
function toolsMenuHtml() {
  // The list's shells ride the composer HTML so a re-render
  // keeps the open state chatToolsOpen carries; the rows are
  // (re)built by paintToolsMenu() from the live DOM.
  return "<div id='tools-shade' style='display:" + (chatToolsOpen ? "block" : "none") + "'></div><div id='tools-menu' role='menu' aria-label='Chat tools' style='display:" + (chatToolsOpen ? "block" : "none") + "'><div id='tools-rows'></div></div>";
}
function toolsRowsHtml() {
  var rows = "";
  if (document.getElementById("chat-newcase")) rows += "<button type='button' class='tools-row' data-tools-act='newcase'><span class='tools-ic'>" + caseSvg() + "</span><span class='tools-label'>New case</span></button>";
  if (document.getElementById("chat-upload")) rows += "<button type='button' class='tools-row' data-tools-act='upload'><span class='tools-ic'>" + uploadSvg() + "</span><span class='tools-label'>Attach</span></button>";
  if (document.getElementById("chat-call")) rows += "<button type='button' class='tools-row' data-tools-act='call'><span class='tools-ic'>" + phoneSvg() + "</span><span class='tools-label'>Call</span></button>";
  return rows;
}
function paintToolsMenu() {
  var holder = document.getElementById("tools-rows");
  if (!holder) return;
  holder.innerHTML = toolsRowsHtml();
  holder.querySelectorAll("[data-tools-act]").forEach(function (row) {
    row.onclick = function () {
      var act = row.getAttribute("data-tools-act");
      var targetId = act === "newcase" ? "chat-newcase" : act === "upload" ? "chat-upload" : "chat-call";
      closeChatTools();
      var btn = document.getElementById(targetId);
      if (btn) btn.click();
    };
  });
}
function openChatTools() {
  chatToolsOpen = true;
  paintToolsMenu();
  var shade = document.getElementById("tools-shade");
  var menu = document.getElementById("tools-menu");
  if (shade) shade.style.display = "block";
  if (menu) menu.style.display = "block";
  var btn = document.getElementById("chat-tools");
  if (btn) btn.setAttribute("aria-expanded", "true");
}
function closeChatTools() {
  chatToolsOpen = false;
  var shade = document.getElementById("tools-shade");
  var menu = document.getElementById("tools-menu");
  if (shade) shade.style.display = "none";
  if (menu) menu.style.display = "none";
  var btn = document.getElementById("chat-tools");
  if (btn) btn.setAttribute("aria-expanded", "false");
}
function ensureToolsEsc() {
  if (document._toolsEscBound) return;
  document._toolsEscBound = true;
  document.addEventListener("keydown", function (e) { if (e.key === "Escape" && chatToolsOpen) closeChatTools(); });
}
function updaterStatusHtml() {
  if (CONFIG.backend !== "live") {
    return "<span style='color:var(--dim)'>Update checks run over the live bridge — this window is in Mock backend mode (Settings → System), so nothing is checked from here. Switch to Live and the check below asks GitHub for the project's latest release.</span>";
  }
  if (updater.state === "checking") return "<span style='color:var(--dim)'>Checking GitHub for the latest release…</span>";
  if (updater.state === "applying") return "<span style='color:var(--dim)'>Updating now — the step it is on is below. Apex will restart itself when the install finishes.</span>";
  if (updater.state === "failed") return "<span style='color:#f5b544'>The last check didn't complete: " + esc(updater.error) + "</span><br><span style='color:var(--dim)'>That is all the check can honestly say — it never guesses that you are up to date.</span>";
  if (updater.state === "applyfailed") return "<span style='color:#f5b544'>The update didn't finish: " + esc(updater.applyError) + "</span>" + (updater.stagedDir ? "<br><span style='color:var(--dim)'>The staged package is still in place — the manual fallback is below.</span>" : "");
  if (updater.state === "updated") return "<span style='color:#34d399'>Updated <b class='mono'>" + esc(updater.fromLabel) + "</b> → <b class='mono'>" + esc(updater.toLabel) + "</b></span><br><span style='color:var(--dim)'>Confirmed from the VERSION file after the restart — the version above is the one now running.</span>";
  if (updater.state === "uptodate") {
    // Build 2: an Edge build checked on the Stable channel must
    // not be told Stable's newest is "the same as yours" — the
    // running preview is not on that channel at all. The line
    // names the channel the check followed instead.
    if (CONFIG.versionStatus === "edge" && updater.channel === "stable" && updater.latest) return "<span style='color:var(--dim)'>That check followed the <b>Stable</b> channel — its newest is <b class='mono'>v" + esc(updater.latest.replace(/^v/i, "")) + "</b>. You're running an Edge build (v" + esc(CONFIG.version) + esc(verTag(CONFIG.versionStatus)) + "), which the Stable channel doesn't carry — there is nothing on this channel to offer you.</span>";
    return "<span style='color:var(--dim)'>You're on the latest release" + (updater.latest ? " — GitHub's newest is <b class='mono'>v" + esc(updater.latest.replace(/^v/i, "")) + "</b>, the same as yours" : "") + ".</span>";
  }
  if (updater.state === "available") {
    var buildTxt = (updater.buildSeq && updaterSameNumbers()) ? " · Build " + updater.buildSeq + " — a refresh of the preview you are running" : "";
    var when = updater.publishedAt ? new Date(updater.publishedAt) : null;
    var whenTxt = when && !isNaN(when.getTime()) ? " · published " + when.toLocaleDateString(undefined, { year: "numeric", month: "short", day: "numeric" }) : "";
    var sizeTxt = updater.assetSize ? " · " + (updater.assetSize / 1048576).toFixed(1) + " MB package" : "";
    return "<span style='color:var(--text)'>Available: <b class='mono'>v" + esc(updater.latest.replace(/^v/i, "")) + "</b>" + esc(buildTxt) + "</span><span style='color:var(--dim)'>" + esc(whenTxt + sizeTxt) + "</span>" +
      (updater.releaseUrl ? "<br><a class='ghost' href='" + esc(updater.releaseUrl) + "' target='_blank' rel='noreferrer' style='margin-top:6px'>Release notes on GitHub</a>" : "") +
      (updater.stageError ? "<br><span style='color:#f5b544'>" + esc(updater.stageError) + "</span>" : "");
  }
  return "<span style='color:var(--dim)'>Not checked yet this session. A check asks GitHub for the project's latest release and compares its tag with the version above.</span>";
}
function setUpdaterChannel(chan) {
  // Release channels (build .83): the switch persists through
  // the bridge (config updater block), so checks and applies
  // after a restart follow the same channel — and a fresh check
  // runs on the new channel at once, since the last verdict
  // belonged to the old one.
  if (chan !== "stable" && chan !== "edge") return;
  if (updater.channel === chan && updater.state !== "idle") return;
  updater.channel = chan;
  CONFIG.updaterChannel = chan;
  updaterChannelExplicit = true;
  updater.state = "idle";
  updater.latest = "";
  updater.notes = "";
  render();
  if (CONFIG.backend === "live") {
    // The fresh check runs once the choice is persisted (Build
    // 2): the bridge treats a stamped choice as authoritative
    // when it resolves the check's channel, so the save landing
    // first is what makes the verdict belong to the new channel.
    fetch("/api/settings/updater-channel", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ channel: chan }) })
      .then(function () { checkForUpdates(); })
      .catch(function () { checkForUpdates(); });
  }
}
function updaterSameNumbers() {
  var t = String(updater.latest || "").replace(/^v/i, "").replace(/[eb]$/i, "");
  return !!t && t === String(CONFIG.version || "");
}
function updaterEdgeNoticeHtml() {
  // The Edge notices (build .86): shown while an update is on
  // offer and the offer (or the running build) is on the preview
  // line. The AMBER preview notice is calmer than the red Sudo
  // warning; both can show, stacked in the warnings seat above
  // the notes. Graduation gets a positive GREEN notice instead
  // of a warning. Ordinary full releases on the Stable channel
  // get no notice at all.
  if (updater.state !== "available") return "";
  var curEdge = CONFIG.versionStatus === "edge";
  var candEdge = !!updater.prerelease || /e$/i.test(String(updater.latest || "").replace(/^v/i, ""));
  var carry = "Your Soul, Memory, chips and packs carry over, as always.";
  if (curEdge && !candEdge) {
    return "<div class='card' style='margin-top:10px;border-color:#34d399'><b style='color:#34d399'>The full release is out.</b><br><span style='color:var(--dim)'>Updating leaves the preview line: v" + esc(String(updater.latest || "").replace(/^v/i, "")) + " is the finished release of the version you have been previewing. " + carry + "</span></div>";
  }
  if (!candEdge) return "";
  if (curEdge && updaterSameNumbers() && updater.buildSeq) {
    return "<div class='card' style='margin-top:10px;border-color:#f5b544'><b style='color:#f5b544'>A refreshed build of v" + esc(String(updater.latest || "").replace(/^v/i, "")) + ".</b><br><span style='color:var(--dim)'>Build " + updater.buildSeq + " replaces Build " + updater.currentBuildSeq + " — the same preview version, rebuilt; not a new version. Preview contents can change between builds. " + carry + " And the next full release supersedes the preview automatically.</span></div>";
  }
  return "<div class='card' style='margin-top:10px;border-color:#f5b544'><b style='color:#f5b544'>A newer preview from the Edge line.</b><br><span style='color:var(--dim)'>This machine is on the preview line: Edge builds arrive before they are finished, and preview contents can change between builds. " + carry + " And the next full release supersedes the preview automatically.</span></div>";
}
function updaterChannelNoticeHtml() {
  // The channel-mismatch notice (Build 2): this build is an Edge
  // preview and the operator has EXPLICITLY set the Updater to
  // the Stable channel — a real choice with a real cost, stated
  // plainly in the warnings seat: Edge builds are never published
  // to Stable, so no Edge update or refresh of this preview will
  // ever be offered while the channel stands. Calm amber, the
  // Edge preview notice's family — a heads-up, not an alarm. An
  // Edge build with no explicit choice never lands here (its
  // effective channel IS Edge), and neither does a Stable build.
  // The status column's single "Up to date" / "Update available"
  // voice is not touched.
  if (CONFIG.backend !== "live") return "";
  if (CONFIG.versionStatus !== "edge" || updater.channel !== "stable" || !updaterChannelExplicit) return "";
  return "<div class='card' style='margin-top:10px;border-color:#f5b544'><b style='color:#f5b544'>An Edge preview on the Stable channel.</b><br><span style='color:var(--dim)'>You're running an Edge build (v" + esc(CONFIG.version) + esc(verTag(CONFIG.versionStatus)) + "), and the Release channel below is set to Stable by your choice. Edge builds and refreshes of this preview are published only on the Edge channel, so none will be offered here. Switch the channel to Edge (pre-release) to keep following the preview line — the next full release supersedes the preview on either channel.</span></div>";
}
function updaterChannelHtml() {
  if (CONFIG.backend !== "live") return "";
  var btn = function (chan, label) {
    var on = updater.channel === chan;
    return "<button type='button' class='" + (on ? "act" : "ghost") + "' data-upchannel='" + chan + "' aria-pressed='" + on + "' style='min-height:44px'>" + label + "</button>";
  };
  return "<div class='card' style='margin-top:10px'><b>Release channel</b><div class='tabbar' style='margin-top:8px'>" +
    btn("stable", "Stable") + btn("edge", "Edge (pre-release)") + "</div>" +
    "<p style='color:var(--dim);margin:8px 0 0'>" + (updater.channel === "edge"
      ? "Edge: the newest build as soon as it is cut — pre-releases from the dev branch, tagged like v0.1.8e. Newer and less settled. Switching back to Stable never downgrades you; it waits for the next full release to pass what you run."
      : "Stable: full releases only, cut from main and tagged with the bare number, like v0.1.7. Edge pre-releases never appear on this channel.") + "</p></div>";
}
var updaterFoldOpen = false;
function settingsUpdaterHtml() {
  // 0.2.0e Build 10 (#196): the whole Updater, folded into
  // Settings → System as its own collapsible dropdown — check,
  // download, apply, status and the apply trace all travel; no
  // updater UI is left anywhere else. The panel order inside
  // the fold is the Updater view's own definitive order (Alex,
  // 2026-11-09): status card, the warnings seat, the channel
  // notices, the channel card, the notes surface, the actions,
  // then the progress chain and the outcome cards. The Current
  // Version Notes are NOT in the fold: they close the System
  // menu, below it. The fold's open state is tracked
  // (updaterFoldOpen) so the apply poll's re-renders never
  // slam it shut mid-apply.
  var h = "<details class='card' id='settings-updater'" + (updaterFoldOpen ? " open" : "") + " style='margin-top:10px'><summary style='cursor:pointer'><b>Updater</b> <span style='color:var(--dim)'>(check, download, install — and the record of the last apply)</span></summary>";
  h += "<div class='card' style='margin-top:10px'>Current: <b class='mono'>v" + esc(CONFIG.version) + esc(verTag(CONFIG.versionStatus)) + "</b> <span style='color:var(--dim)'>(" + esc(CONFIG.versionStatus) + ")" + (CONFIG.versionStatus === "edge" ? " · Build " + updater.currentBuildSeq : "") + "</span><br>" + updaterStatusHtml() + "</div>";
  if (updater.state === "available") {
    h += CONFIG.sudo
      ? "<p style='color:#34d399;font-size:13px;margin:8px 0 0'>Sudo Mode is on — ready to update.</p>"
      : "<p style='color:#f87171;font-size:13px;margin:8px 0 0'>⚠ <b>Enable Sudo Mode before updating.</b> The installer's system step needs root. Without it, an update can stall or finish without its system packages.</p>";
  }
  h += updaterChannelNoticeHtml();
  h += updaterEdgeNoticeHtml();
  h += updaterChannelHtml();
  h += updaterNotesHtml();
  h += "<div style='margin-top:10px'><button class='act' id='up-check'" + (CONFIG.backend !== "live" || updater.state === "checking" || updater.state === "applying" ? " disabled" : "") + " style='min-height:44px'>" + (updater.state === "checking" ? "Checking…" : "Check for updates") + "</button> ";
  if (updater.state === "available") h += "<button class='ghost' id='up-apply' style='min-height:44px'>Download &amp; install v" + esc(updater.latest.replace(/^v/i, "")) + "</button>";
  h += "</div>";
  if (updater.state === "applying") h += updaterProgressHtml();
  if (updater.state === "updated") h += updaterUpdatedHtml();
  if (updater.state === "applyfailed" && updater.stagedDir) h += updaterFallbackHtml();
  return h + "</details>";
}
function wireUpdaterFold() {
  var f = document.getElementById("settings-updater");
  if (f) f.addEventListener("toggle", function () { updaterFoldOpen = f.open; });
  var upCheck = document.getElementById("up-check");
  if (upCheck) upCheck.onclick = function () { checkForUpdates(); };
  var upApply = document.getElementById("up-apply");
  if (upApply) upApply.onclick = function () { updaterFoldOpen = true; applyUpdate(); };
  mainEl.querySelectorAll("[data-upchannel]").forEach(function (b) { b.onclick = function () { setUpdaterChannel(b.getAttribute("data-upchannel")); }; });
}
function updaterCurrentHtml() {
  // "About this release" (build .83): the current version's own
  // release page as a dropdown — everything the page offers:
  // name, tag, full/Edge badge, published date, the link, the
  // package asset, and the full patch notes rendered in place.
  var cur = updater.currentRelease;
  var head = "About this release — v" + esc(CONFIG.version) + esc(verTag(CONFIG.versionStatus));
  if (!cur) {
    return "<details class='card' style='margin-top:10px'><summary style='cursor:pointer'><b>" + head + "</b> <span style='color:var(--dim)'>(patch notes for the version you are running)</span></summary>" +
      "<p style='color:var(--dim)'>Not fetched yet — run Check for updates and this fills in from the release page on GitHub. If GitHub carries no release for this version, it says so rather than guessing.</p></details>";
  }
  var when = cur.publishedAt ? new Date(cur.publishedAt) : null;
  var whenTxt = when && !isNaN(when.getTime()) ? " · published " + when.toLocaleDateString(undefined, { year: "numeric", month: "short", day: "numeric" }) : "";
  var assetTxt = cur.asset ? " · package " + esc(cur.asset.name) + (cur.asset.size ? " (" + (cur.asset.size / 1048576).toFixed(1) + " MB)" : "") : "";
  var src = cur.notesSource === "changelog" ? "<br><span style='color:var(--dim);font-size:12px'>The release page itself carried no notes, so these are the changelog's section for this version.</span>" : "";
  return "<details class='card' style='margin-top:10px'><summary style='cursor:pointer'><b>" + head + "</b> <span style='color:var(--dim)'>(patch notes for the version you are running)</span></summary>" +
    "<p style='margin:8px 0 0'><a href='" + esc(cur.url) + "' target='_blank' rel='noreferrer'><b>" + esc(cur.name || cur.tag) + "</b></a> <span class='mono' style='color:var(--dim);font-size:12px'>(" + esc(cur.tag) + ")</span></p>" +
    "<p style='color:var(--dim);margin:4px 0 0'>" + (cur.prerelease ? "Edge pre-release — cut from the dev branch." : "Full release — cut from main.") + esc(whenTxt + assetTxt) + "</p>" + src +
    (cur.notes ? "<div class='md' style='margin-top:6px'>" + mdToHtml(stripLeadH1(cur.notes, { latest: cur.tag })) + "</div>" : "<p style='color:var(--dim)'>The release page carries no notes of its own.</p>") +
    "</details>";
}
function updaterNotesHtml() {
  // The release-notes panel (issue #47): shown only while an
  // update is on offer (or being applied) and only when the
  // check actually carried notes — a failed check never reaches
  // this state, and an up-to-date check shows no panel at all.
  // Rendered with the app's own markdown, in the theme's colors.
  if (!updater.notes || (updater.state !== "available" && updater.state !== "applying")) return "";
  var src = updater.notesSource === "changelog"
    ? "<br><span style='color:var(--dim);font-size:12px'>The release page itself carried no notes, so these are the changelog's section for this version — the same words, from CHANGELOG.md.</span>" : "";
  return "<div class='card' style='margin-top:10px'><b>What's in v" + esc(updater.latest.replace(/^v/i, "")) + "</b>" + src +
    "<div class='md' style='margin-top:6px'>" + mdToHtml(stripLeadH1(updater.notes, updater)) + "</div></div>";
}
function updaterUpdatedHtml() {
  var log = "### Update log\n\n" +
    "- **Downloaded** the package and verified it against the release.\n" +
    "- **Staged and installed** it — everything that's yours (chips, knowledge packs and their indexes, the Vault, the model file, Soul and Memory, her voice, your GitHub settings) rode the keep-list across.\n" +
    "- **DHD prerequisites** — " + (updater.dhdPrereqs || "installed by the update's installer run as their own named step") + ".\n" +
    "- **Restarted and confirmed** from the VERSION file: " + updater.fromLabel + " → " + updater.toLabel + ".\n";
  var tailNote = updater.tailFailure ? "<br><span style='color:#f5b544'><b>Installed, with a tail failure in the " + esc(updater.tailFailure) + " step.</b> The installer exited nonzero after the new build was already in place — the version confirmed below is the target build, so this is not the manual-fallback failure. The installer's own output for that step is in its log (apply-install.log, beside the staged package).</span>" : "";
  return "<div class='card' style='margin-top:10px'><b>Updated " + esc(updater.fromLabel) + " → " + esc(updater.toLabel) + "</b><br>" +
    "<span style='color:var(--dim)'>The update downloaded, installed, and restarted itself — and the version now running is the one the update promised, read from the VERSION file after the restart.</span>" + tailNote +
    "<div class='md' style='margin-top:6px'>" + mdToHtml(log) + "</div></div>";
}
// Issue #17 — the first-launch race: the Chat view reads
// llama.status as it renders (a stopped model renders the
// CLI-only state), but a status change learned by the poll
// used to repaint only the status column, so an open Chat
// stayed in whichever state it first painted with until the
// operator left the view and came back. On a transition —
// either direction — an open Chat re-renders once, so it
// heals the moment the model arrives and degrades honestly
// if the model drops. The composer's half-typed line and its
// focus are carried across by hand: render() rebuilds the
// input, and a poll must never eat what the operator is
// typing. (The placeholder follows on its own —
// chatPlaceholder() reads the same status, so the fresh
// render words it right.) Returns whether it re-rendered.
function llamaTransitionHeal(prevStatus) {
  if (prevStatus === llama.status || view !== "Chat" || naviThinking || (call && callState === "thinking")) return false;
  var before = document.getElementById("chat-in");
  var typed = before ? before.value : "";
  var focused = !!before && document.activeElement === before;
  render();
  var after = document.getElementById("chat-in");
  if (after && typed) {
    after.value = typed;
    if (focused) { after.focus(); try { after.setSelectionRange(typed.length, typed.length); } catch (e) {} }
  }
  return true;
}
function serversSummaryDots() {
  // The collapsed Servers header still reports honestly (build
  // .79): one dot per row, colored exactly as that row's own dot
  // is colored below - collapsing the group never hides a problem.
  var ps = publicState || CONFIG.publicServer || { enabled: false, activePublic: false, pending: false };
  var nodeLive = !!ps.activePublic && (CONFIG.backend !== "live" || publicReportLive);
  var nodeDot = ps.pending ? "#f5b544" : nodeLive ? "#34d399" : "#9fb3c8";
  var d = dhdViewState();
  var dhdDot = d.status === "healthy" ? "#34d399" : (d.status === "degraded" || d.status === "starting") ? "#f5b544" : "#9fb3c8";
  var llamaDot = llama.status === "running" ? "#34d399" : "#9fb3c8";
  var meshDot = meshDotColor(meshViewState());
  function dot(c) { return "<span class='dot' style='background:" + c + ";margin-left:6px'></span>"; }
  // Build 2 (#204): the header's dots obey the same law as
  // the rows — a feature that is off leaves no dot behind.
  // The engine (Navi) has no off switch and always reports.
  var out = dot(llamaDot);
  if (ps.enabled || ps.activePublic || ps.pending) out += dot(nodeDot);
  if (meshViewState().enabled) out += dot(meshDot);
  if (d.active) out += dot(dhdDot);
  return out;
}
function wireStatusTabs(root) {
  if (!root) return;
  root.querySelectorAll("[data-statustab]").forEach(function (t) {
    t.onclick = function () { statusPaneTab = t.getAttribute("data-statustab"); render(); };
    t.onkeydown = function (e) {
      if (e.key !== "ArrowLeft" && e.key !== "ArrowRight") return;
      e.preventDefault();
      statusPaneTab = statusPaneTab === "status" ? "chats" : "status";
      render();
      var nt = document.getElementById("status-tab-" + statusPaneTab);
      if (nt) nt.focus();
    };
  });
}
// Choosing a conversation lands the operator ON it (Build 6,
// #108): the Chat view opens on the chosen thread, and on
// mobile the dock closes so the thread is what they land on —
// whichever view they were in when they picked it.
function landOnChat() { dockOpen = false; view = "Chat"; render(); }
// Build 3 (#177): chat deletion is a confirmed act that goes
// through the server — /api/chats/delete existed since the store
// moved server-side, but no control ever called it; the pane's
// old Delete only filtered the local array and let the write-
// through figure it out, with no confirm at all. Armed state and
// the last refusal live here; ids compare as STRINGS throughout
// (server-era ids are strings — the old Number() coercion turned
// them into NaN and silently deleted nothing).
var chatDeleteArmId = null;
var chatDeleteError = "";
function wireChatsPanel(root) {
  if (!root) return;
  root.querySelectorAll("[data-switch]").forEach(function (b) { b.onclick = function () { var id = b.getAttribute("data-switch"); var c = chats.filter(function (x) { return String(x.id) === String(id); })[0]; if (!c) return; activeChatId = c.id; messages = c.messages; persist(); landOnChat(); }; });
  root.querySelectorAll("[data-delchat]").forEach(function (b) { b.onclick = function () { chatDeleteArmId = b.getAttribute("data-delchat"); chatDeleteError = ""; render(); }; });
  root.querySelectorAll("[data-delchat-no]").forEach(function (b) { b.onclick = function () { chatDeleteArmId = null; chatDeleteError = ""; render(); }; });
  root.querySelectorAll("[data-delchat-yes]").forEach(function (b) { b.onclick = function () {
    var id = b.getAttribute("data-delchat-yes");
    var finish = function () {
      chats = chats.filter(function (x) { return String(x.id) !== String(id); });
      if (!chats.length) seedChats();
      if (!chats.some(function (x) { return String(x.id) === String(activeChatId); })) { activeChatId = chats[0].id; messages = chats[0].messages; }
      chatDeleteArmId = null; chatDeleteError = "";
      persist(); render();
    };
    if (CONFIG.backend === "live") {
      fetch("/api/chats/delete", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ id: id }) }).then(function (r) { return r.json(); }).then(function (j) {
        if (j && j.ok === false) { chatDeleteError = j.error || "The bridge refused that delete."; render(); return; }
        finish();
      }).catch(function () { chatDeleteError = "Couldn't reach the bridge — the chat is untouched."; render(); });
    } else { finish(); }
  }; });
  var newBtn = root.querySelector("#chat-new");
  if (newBtn) newBtn.onclick = function () { var t = root.querySelector("#chat-title"); var title = t && t.value.trim() ? t.value.trim() : "Chat " + (chats.length + 1); var c = { id: Date.now(), title: title, messages: [] }; chats.push(c); activeChatId = c.id; messages = c.messages; persist(); landOnChat(); };
  root.querySelectorAll("[data-delupload]").forEach(function (b) { b.onclick = function () { uploads.splice(Number(b.getAttribute("data-delupload")), 1); persistUploads(); render(); }; });
}

function chatsPanelHtml() {
  // The Chats pane (0.1.9e Build 6, issues #107/#108): what the
  // retired Memory view held for conversations — the saved chats
  // (switch, new, delete) and the Uploaded files section — as
  // the status pane's second tab. Same data, same storage keys,
  // same handlers (wireChatsPanel binds what the Memory wiring
  // bound); only the home changed, from Build 4's disclosure —
  // whose panel opened at the column's foot — to a pane that
  // renders directly under the tab bar.
  var h = "";
  h += "<p style='color:var(--dim);font-size:12px;margin:2px 0 8px'>Every conversation is saved locally. Switch here and the thread picks that conversation back up.</p>";
  if (chatDeleteError) h += "<p style='color:#f87171;font-size:12px;margin:2px 0 8px'>" + esc(chatDeleteError) + "</p>";
  h += chats.map(function (c) {
    var on = String(c.id) === String(activeChatId);
    var armed = chatDeleteArmId !== null && String(chatDeleteArmId) === String(c.id);
    var foot;
    if (armed && !c.case) {
      foot = "<div style='margin-top:8px;border-top:1px solid var(--line);padding-top:8px'><span style='color:#f5b544;font-size:12px'>Delete <b>" + esc(c.title) + "</b>? Its " + c.messages.length + " messages go with it — and the deletion travels with Sync, so it won't come back from your other machine. Copies inside backups already taken stay in those backups.</span><div style='margin-top:6px;display:flex;gap:6px'><button class='ghost' data-delchat-yes='" + esc(String(c.id)) + "' style='min-height:44px;color:#f87171;border-color:#f87171'>Delete it</button><button class='ghost' data-delchat-no='1' style='min-height:44px'>Keep it</button></div></div>";
    } else {
      foot = "<div style='margin-top:6px;display:flex;gap:6px'>" +
        (on ? "<span class='mono' style='color:var(--cy);font-size:11px;align-self:center'>ACTIVE</span>" : "<button class='ghost' data-switch='" + c.id + "' style='min-height:40px'>Switch</button>") +
        (String(c.id) === "1" ? "<span style='color:var(--dim);font-size:11px;align-self:center'>The main chat can't be deleted — /clear or /cls empties it.</span>" : c.case ? "<span style='color:var(--dim);font-size:11px;align-self:center'>Bound to its case — it deletes through the case: deleting the case deletes its chat.</span>" : (chats.length > 1 ? "<button class='ghost' data-delchat='" + esc(String(c.id)) + "' style='min-height:44px'>Delete</button>" : "")) + "</div>";
    }
    return "<div class='card' style='margin-bottom:8px;padding:10px;" + (on ? "border-color:var(--cy)" : "") + "'><b>" + esc(c.title) + "</b>" + (c.case ? " <span class='mono' style='font-size:11px;color:#34d399'>CASE</span>" : "") + " <span style='color:var(--dim);font-size:12px'>" + c.messages.length + " messages</span>" + foot + "</div>";
  }).join("");
  h += "<div class='row'><input class='grow' id='chat-title' type='text' placeholder='New chat title (optional)' aria-label='New chat title'><button class='act' id='chat-new' style='min-height:44px'>New chat</button></div>";
  h += "<h3 style='margin-top:14px'>Uploaded files</h3>" + (uploads.length ? uploads.map(function (u, i) { return "<div class='card' style='margin-bottom:8px;padding:10px'>" + esc(u.name) + " <span style='color:var(--dim);font-size:12px'>" + esc(u.kind) + (u.chat ? " · in " + esc(u.chat) : "") + (u.at ? " · " + esc(String(u.at).slice(0, 10)) : "") + "</span> <button class='ghost' data-delupload='" + i + "' aria-label='Delete uploaded file " + esc(u.name) + "' style='min-height:40px'>Delete</button></div>"; }).join("") : "<p style='color:var(--dim);font-size:12px'>Nothing uploaded yet — attach a picture or a text file from the chat composer and it is indexed here.</p>");
  return h;
}

// The status column's federation list lives in its OWN
// variable (#212): the Federation page's federationState is
// the state OBJECT from POST /api/federation, this one is
// the status payload's [{name, state}] array. They shared a
// global once — the boot-time status poll overwrote the
// page's object with the array, federationHtml() threw on
// knownDefault, and the Federation view never painted.
var fedStatusList = null;
function renderFederationStatus(live) {
  // Build 2 (#204): the FediNet sections — under the Servers
  // list, above Sudo Mode, exactly where the decree puts
  // them. One line per fedinet this node belongs to, the name
  // colored by the federation machinery's own verdict
  // (bridge-side, in the status payload): green when the
  // fedinet is reachable, red when it cannot connect, amber
  // for the named middle states — a roster past its 72-hour
  // freshness window (pairing-only until a sync lands), or a
  // gateway no probe has checked yet. Not a member of any
  // fedinet: the one line the decree names. Mock mode has no
  // federation truth to read, so the section stays absent
  // there rather than inventing one.
  if (CONFIG.backend !== "live") return "";
  if (live && Array.isArray(live.federation)) fedStatusList = live.federation;
  if (!fedStatusList) return "";
  var feds = fedStatusList;
  if (!feds.length) {
    return "<div class='stat'><span style='color:var(--dim)'>You are not federated.</span></div>";
  }
  return feds.map(function (f) {
    var color = f.state === "ok" ? "var(--green)" : f.state === "down" ? "var(--red)" : "var(--am)";
    var note = f.state === "stale" ? " title='Roster past its 72-hour freshness window — pairing-only until a roster sync lands'"
      : f.state === "unknown" ? " title='The gateway has not been checked yet'"
      : f.state === "down" ? " title='The fedinet\u2019s gateway cannot be reached'"
      : " title='Reachable'";
    return "<div class='stat'><span style='color:var(--dim)'>FediNet:</span> <span class='v mono' style='font-weight:700;color:" + color + "'" + note + ">" + esc(f.name) + "</span></div>";
  }).join("");
}
function renderStatus(live) {
  var prevLlamaStatus = llama.status;
  if (CONFIG.backend === "live" && live) llama.status = live.llamafile_reachable ? "running" : "stopped";
  // The Updater line (owner's spec): directly under Navi's
  // status, bracketed by the column's separators, and nothing
  // else — one line that flips state. Green "Up to date" when a
  // check has run clean; yellow "Update available" when one
  // found a newer release, and the line then opens the Updater.
  // No label above it, no subtext beneath it; the honest
  // in-between states keep their own single plain words.
  var updHtml;
  if (updater.state === "available" || updater.state === "applying") {
    updHtml = "<button id='updater-warn' style='display:block;width:100%;text-align:left;background:none;border:0;color:#f5b544;cursor:pointer;font:inherit;font-weight:700;padding:2px 0' aria-label='Update available — open the Updater tab to apply it'>Update available</button>";
  } else if (updater.state === "applyfailed") {
    updHtml = "<button id='updater-warn' style='display:block;width:100%;text-align:left;background:none;border:0;color:#f5b544;cursor:pointer;font:inherit;font-weight:700;padding:2px 0' aria-label='Update didn't finish — open the Updater tab'>Update didn't finish</button>";
  } else if (updater.state === "failed") {
    updHtml = "<div class='mono' style='font-weight:700;color:var(--dim)'>Couldn't check for updates</div>";
  } else if (updater.state === "checking") {
    updHtml = "<div class='mono' style='font-weight:700;color:var(--dim)'>Checking for updates\u2026</div>";
  } else if (updater.state === "uptodate" || updater.state === "updated") {
    updHtml = "<div class='mono' style='font-weight:700;color:#34d399'>Up to date</div>";
  } else {
    updHtml = "<div class='mono' style='font-weight:700;color:var(--dim)'>" + (CONFIG.backend === "live" ? "Not checked yet" : "Live bridge only") + "</div>";
  }
  // v0.4.10e (#250 + #254, as ruled): the pane opens on the
  // sync bar. The STATUS header and its lone green dot are
  // retired — the dot was the presence row's, left orphaned
  // when #248 took the presence line out of the pane; her
  // portrait, name, and dot live at the top of the chat,
  // untouched — and the bar takes their seat: "System Synced"
  // at rest, the progress bar while a round runs, the receipt
  // hanging off it on landing. Then the sections, unchanged.
  var statusBody =
    renderSyncBar(live) +
    (bridgeStale && CONFIG.backend === "live"
      ? "<div class='stat' style='border:1px solid #f87171;color:#f87171'><b>BRIDGE OUT OF DATE</b><br>An older bridge.py is still answering on port " + BRIDGE_PORT + ". This dashboard is the new build, but the bridge process isn't — so chat returns the llamafile's 400 and sudo authentication 404s. Fully quit the app and relaunch it, or re-run the installer (it now stops the old bridge for you).</div>"
      : "") +
    (pageStale && CONFIG.backend === "live"
      ? "<div class='stat' style='border:1px solid #f5b544;color:#f5b544'><b>THIS PAGE IS FROM THE PREVIOUS BUILD</b><br>The bridge answering on port " + BRIDGE_PORT + " is running Build " + bridgeBuildSeq + ", but this page is Build " + SHIPPED_BUILD_SEQ + " — the update landed while this dashboard was open, and the page itself is the stale half now. Reload to land on the new build.<br><button id='page-stale-reload' style='margin-top:8px;min-height:44px;padding:8px 16px;background:none;border:1px solid #f5b544;border-radius:8px;color:#f5b544;cursor:pointer;font:inherit;font-weight:700' aria-label='Reload the dashboard onto the new build'>Reload the dashboard</button></div>"
      : "") +
    "<div class='stat' style='border-bottom:1px solid var(--line);padding:8px 0 10px'>" + updHtml + "</div>" +
    // The Servers block carries the standard .stat boundary
    // (build .80, issue #67): the #66 parent was a bare <details>
    // with no bottom spacing, so in the expanded default state
    // Sudo rendered flush against the DHD row. Wrapped in .stat,
    // the block gets the same margin, padding, and divider as
    // every other status block — expanded and collapsed alike,
    // since the wrapper renders in both states.
    "<div class='stat'><details id='servers-details'" + (serversOpen ? " open" : "") + "><summary aria-label='Servers - Navi, Apex | Node, Apex | Mesh and Apex | DHD' style='color:var(--dim);font-size:11px;letter-spacing:.14em;font-weight:700;cursor:pointer;padding:2px 0'>SERVERS" + serversSummaryDots() + "</summary>" +
    renderLlama() +
    renderLocalNetwork() +
    renderMeshStatus() +
    renderDhdStatus() +
    renderBootStatus(live && live.boot) +
    "</details></div>" +
    renderFederationStatus(live) +
    renderSudo(live) +
    renderVoiceStatus() +
    stat("CASE", openCaseState ? openCaseState.name : "No case open", openCaseState ? "#34d399" : "#9fb3c8") +
    stat("MEMORY", memoryStatText(live), "#34d399") +
    stat("RAG", CONFIG.backend === "live" ? (ragPacksState ? ragPacksState.filter(function (p) { return p.active; }).length + " of " + ragPacksState.length + " packs active" : "…") : "simulated (mock mode)", "#34d399") +
    backupStatHtml(githubState);
  // The tabbed pane composes here (Build 6, #108): the bar is
  // always the column's first element, and the selected pane —
  // Status (statusBody above) or Chats (chatsPanelHtml) — swaps
  // in directly beneath it. While Chats is selected, the Status
  // tab carries an amber dot for any standing summons, so the
  // tab never buries one.
  var statusAttn = (updater.state === "available" || updater.state === "applyfailed" || (bridgeStale && CONFIG.backend === "live") || (pageStale && CONFIG.backend === "live"));
  var statusTabsHtml = "<div class='status-tabs' role='tablist' aria-label='Status pane'>"
    + "<button type='button' role='tab' id='status-tab-status' data-statustab='status' class='status-tab" + (statusPaneTab === "status" ? " on" : "") + "' aria-selected='" + (statusPaneTab === "status" ? "true" : "false") + "' aria-controls='status-pane-status'>Status" + (statusPaneTab === "chats" && statusAttn ? "<span class='status-attn' role='img' aria-label='Status needs attention'></span>" : "") + "</button>"
    + "<button type='button' role='tab' id='status-tab-chats' data-statustab='chats' class='status-tab" + (statusPaneTab === "chats" ? " on" : "") + "' aria-selected='" + (statusPaneTab === "chats" ? "true" : "false") + "' aria-controls='status-pane-chats'>Chats<span class='chats-count mono'>" + chats.length + "</span></button></div>";
  statusEl.innerHTML = statusTabsHtml + "<div role='tabpanel' id='status-pane-" + statusPaneTab + "' class='status-pane-in' aria-labelledby='status-tab-" + statusPaneTab + "'>" + (statusPaneTab === "chats" ? chatsPanelHtml() : statusBody) + "</div>";
  // No threat-level element: the operator took it out — it was a
  // decorative selector with no mechanics behind it, and this
  // column carries only statuses that mean something.
  wireSudo();
  wireLlama();
  wireLocalNetwork();
  wireDhdStatus();
  wireMeshStatus();
  var sdet = document.getElementById("servers-details");
  if (sdet) sdet.addEventListener("toggle", function () { serversOpen = sdet.open; });
  wireStatusTabs(statusEl);
  var uw = document.getElementById("updater-warn");
  if (uw) uw.onclick = function () { view = "Settings"; settingsTab = "system"; updaterFoldOpen = true; dockOpen = false; render(); };
  wireChatsPanel(statusEl);
  var psr = document.getElementById("page-stale-reload");
  if (psr) psr.onclick = function () { location.reload(); };
  var bab = document.getElementById("boot-action");
  if (bab) bab.onclick = function () { bootAction(); };
  if (CONFIG.backend === "live" && !voiceState && !voiceFetching && !naviThinking) refreshVoice(false);
  if (CONFIG.backend === "live" && !ragPacksState && !ragPacksFetching && !naviThinking) refreshRagPacks(false);
  if (!githubState && !githubFetching) fetchGithub(false);
  // Issue #17: a model status transition heals (or honestly
  // degrades) an open Chat view — see llamaTransitionHeal.
  if (live) llamaTransitionHeal(prevLlamaStatus);
  function stat(k, v, c) { return "<div class='stat'><span class='dot' style='background:" + c + "'></span><span style='color:var(--dim)'>" + k + "</span><br><span class='v mono'>" + esc(v) + "</span></div>"; }
}
function memoryStatText(live) {
  // v0.5.1e (#256): the pane reports the operator class's true
  // state — the entry count and the sync clock the bar reads,
  // in the bar's own terms. "(local)" is retired: since this
  // build the list the pane counts is the bridge's converged
  // record, not a browser's private copy. Mock mode has no
  // bridge and says so plainly, as it always has.
  var n = memory.length;
  var noun = n === 1 ? " entry" : " entries";
  if (CONFIG.backend !== "live") return n + noun + " (local)";
  var sync = (live && live.sync) || null;
  if (sync && sync.lastSyncEpoch) return n + noun + " · synced " + syncAgoText(sync.lastSyncEpoch);
  return n + noun + " · not synced yet";
}
function ghWhen(iso) {
  if (!iso) return "";
  var d = new Date(iso);
  return isNaN(d.getTime()) ? "" : d.toLocaleString(undefined, { year: "numeric", month: "short", day: "numeric", hour: "numeric", minute: "2-digit" });
}
// The status column's backup readout (v0.5.1e, #258, the
// operator's words from the field): line one says whether a
// backup has ever landed and how long ago, in the pane's
// green when one has — "Backed up today", "Backed up 1 day
// ago", "Backed up N days ago" — and reads "Never backed up"
// in the muted color when none has; it must not glow like a
// healthy state. Line two, smaller, is the schedule: "Next in
// N days" from the same record and the same arithmetic the
// bridge uses to decide a backup is due (lastBackupAt +
// intervalDays), "Next today" when it is due — and when no
// backup has ever landed there is no anchor for a countdown,
// so it names the cadence instead: "Runs every N days". The
// file count leaves the pane; it stays on Settings → GitHub &
// Backup, whose LAST BACKUP readout reads the same record.
function ghBackupReadout(gh) {
  if (!gh) return { line: "…", color: "#9fb3c8", sub: "" };
  var iv = (typeof gh.intervalDays === "number" && gh.intervalDays > 0) ? gh.intervalDays : 1;
  var then = gh.lastBackupAt ? new Date(gh.lastBackupAt).getTime() : NaN;
  if (!gh.lastBackupAt || isNaN(then)) return { line: "Never backed up", color: "#9fb3c8", sub: "Runs every " + iv + " day" + (iv === 1 ? "" : "s") };
  var days = Math.floor((Date.now() - then) / 86400000);
  if (days < 0) days = 0;
  var line = days === 0 ? "Backed up today" : "Backed up " + days + " day" + (days === 1 ? "" : "s") + " ago";
  var left = iv - days;
  if (left < 0) left = 0;
  var sub = left === 0 ? "Next today" : "Next in " + left + " day" + (left === 1 ? "" : "s");
  return { line: line, color: "#34d399", sub: sub };
}
function backupStatHtml(gh) {
  var b = ghBackupReadout(gh);
  return "<div class='stat'><span class='dot' style='background:" + b.color + "'></span><span style='color:var(--dim)'>BACKUP</span><br><span class='v mono'>" + esc(b.line) + "</span>" +
    (b.sub ? "<div style='color:var(--dim);font-size:12px;margin-top:2px'>" + esc(b.sub) + "</div>" : "") + "</div>";
}
// The GitHub view: Issues (outside her window, no account to look),
// optional Device-Flow sign-in (never a gate on Navi herself), and
// backup/import of the ready-to-fly essentials.
// Issue #25: the GitHub section's entries are cards, in the
// same frame family as the rest of the app — the flip-face
// frame (panel background, one line border, 8px radius) the
// chips and pack cards wear. The whole card is the button, and
// the click is issue #24's: the shell opens the issue's own
// window, a browser opens a tab.
function githubIssueRowsHtml(list) {
  if (!list.length) return "";
  return list.map(function (i) {
    var labels = (i.labels || []).map(function (l) { return "<span class='mono' style='font-size:10px;border:1px solid var(--line);border-radius:999px;padding:1px 7px;color:var(--dim)'>" + esc(l) + "</span>"; }).join(" ");
    return "<button type='button' data-gh-issue='" + esc(i.url) + "' style='display:block;width:100%;text-align:left;background:var(--panel);border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-top:8px;cursor:pointer;color:var(--text);font:inherit' aria-label='Open issue #" + i.number + " on GitHub'>" +
      "<span class='mono' style='color:var(--cy);font-size:12px'>#" + i.number + "</span> <b>" + esc(i.title) + "</b><br>" +
      "<span style='color:var(--dim);font-size:12px'>" + esc(i.state) + (i.author ? " · @" + esc(i.author) : "") + (i.updatedAt ? " · updated " + esc(ghWhen(i.updatedAt)) : "") + (i.comments ? " · " + i.comments + " comment" + (i.comments === 1 ? "" : "s") : "") + "</span>" +
      (labels ? "<br><span style='display:inline-flex;gap:4px;flex-wrap:wrap;margin-top:4px'>" + labels + "</span>" : "") +
      "</button>";
  }).join("");
}
// The quiet footer each GitHub tab (Overview excepted — it stays
// exactly as it is) carries at its bottom: one small link, blended
// into the UI, that leads to the matching place in the repository
// on GitHub — file an issue from Issues, the board from Roadmap,
// a new discussion from Discussions.
function ghFooterHtml(label, url) {
  return "<p class='mono' style='border-top:1px solid var(--line);margin-top:14px;padding-top:9px;font-size:11px;color:var(--dim)'>ON GITHUB · <button type='button' data-gh-open='" + url + "' style='background:none;border:none;padding:0;color:var(--cy);cursor:pointer;font:inherit;text-decoration:underline dotted' aria-label='" + label + " — opens on GitHub'>" + label + " ↗</button></p>";
}
// The Roadmap and Discussions tabs. Inside the standalone shell
// the page itself lives in its own native webview window (opened
// by the tab's wiring through openGithubPageWindow); this body is
// the tab's home base — what the page is, a way to raise the
// window again, and the honest fallback: outside the shell there
// is no webview to give it and GitHub refuses to be framed inside
// another page, so the tab says so and links out rather than
// showing a dead blank frame.
function githubWebviewTabHtml(page) {
  var isRoadmap = page === "roadmap";
  var title = isRoadmap ? "Roadmap" : "Discussions";
  var url = isRoadmap ? NAVI_ROADMAP_URL : NAVI_REPO_URL + "/discussions";
  var what = isRoadmap
    ? "The Roadmap is Alex's project board for Apex — where the work is laid out and tracked."
    : "Discussions are the project's open forum on GitHub — questions, ideas, and conversation about Apex.";
  var h = "<div class='card'><h3>" + title + "</h3><p style='color:var(--dim)'>" + what + "</p>";
  if (docsInShell()) {
    h += "<p style='color:var(--dim)'>It lives in its own Apex window — a real webview, opened natively by the shell, because GitHub will not be framed inside another page. The window opens when you pick this tab; if it has been closed or buried, bring it up again:</p>";
    h += "<button class='act' id='gh-webview-open'>Open the " + title + " window</button> <button class='ghost' data-gh-open='" + url + "'>Open on GitHub instead ↗</button>";
    h += "<p style='color:var(--dim);font-size:12px;margin-top:8px'>GitHub may ask you to sign in inside that window for anything beyond reading. That is GitHub's door — it just happens to be in her window instead of your browser.</p>";
  } else {
    h += "<p style='color:var(--dim)'>In her desktop window this tab opens as a webview of its own. Out here in a plain browser there is no window to give it — and GitHub refuses to be framed inside another page — so this tab cannot embed it. It lives on GitHub:</p>";
    h += "<button class='act' data-gh-open='" + url + "'>Open the " + (isRoadmap ? "Roadmap board" : "Discussions") + " on GitHub ↗</button>";
  }
  h += ghFooterHtml(isRoadmap ? "Open the Roadmap board on GitHub" : "Start a new discussion", isRoadmap ? NAVI_ROADMAP_URL : NAVI_REPO_URL + "/discussions/new");
  h += "</div>";
  return h;
}
// One catalog chip's flip card on the NUR tab — shared by the
// full render and the Community search's live re-render of just
// the list, so the two can never drift apart. Fixed size: the
// front carries the install state and actions; the back repeats
// the basics and scrolls the repo's README inside the card,
// with links to the repo's Issues and, when it has a wiki, its
// Documentation. README bodies load lazily on first flip
// (loadNurReadme) and are cached in nurReadmes.
var nurReadmes = {};
function nurCardHtml(c) {
  var badge = c.installed ? (c.updateAvailable ? "<span class='mono' style='color:#f5b544;font-size:11px'>UPDATE AVAILABLE</span>" : "<span class='mono' style='color:#34d399;font-size:11px'>INSTALLED" + (c.installedRef ? " · " + esc(c.installedRef) : "") + "</span>") : (c.inCatalog ? "<span class='mono' style='color:var(--dim);font-size:11px'>NOT INSTALLED</span>" : "<span class='mono' style='color:var(--dim);font-size:11px'>LOCAL CHIP</span>");
  if (c.locallyEdited) badge += " <span class='mono' style='color:#f5b544;font-size:11px'>EDITED LOCALLY</span>";
  var rel = c.latestRelease ? "latest release " + esc(c.latestRelease.tag) + " · " + esc(String(c.latestRelease.publishedAt || "").slice(0, 10)) : "no releases yet — installs track the default branch";
  // The catalog card is a discovery surface only (issue #21):
  // the state badge, Install, and the repo destinations — every
  // management action for a chip you already have lives on its
  // Local Chips card, and an installed chip's card says so in
  // plain text. Two columns, as ever: the destinations in the
  // flexible left column, and the right column — anchored to the
  // card's right side — carries Install on top with the README
  // button directly beneath it.
  var links = "";
  if (c.htmlUrl) links += "<a class='ghost' href='" + esc(c.htmlUrl) + "' target='_blank' rel='noreferrer'>View on GitHub</a>";
  if (c.latestRelease && c.latestRelease.url) links += " <a class='ghost' href='" + esc(c.latestRelease.url) + "' target='_blank' rel='noreferrer' aria-label='Open release " + esc(c.latestRelease.tag) + " of " + esc(c.name) + " on GitHub'>Release " + esc(c.latestRelease.tag) + "</a>";
  if (c.installed) links += " <span style='color:var(--dim);font-size:11px'>Manage it under Local Chips.</span>";
  var main = "";
  if (!c.installed && c.inCatalog) main += "<button class='act' data-cinstall='" + esc(c.fullName) + "'>Install</button>";
  // Issue #22: an installed chip — this repo's own install, or a
  // slug match (the bridge reconciles by slug, forged chips
  // included) — shows its Install grayed out and dead, never a
  // second live install over the copy already on this machine.
  else if (c.installed) main += "<button class='act' disabled aria-label='" + esc(c.name) + " is already installed' style='opacity:.45;cursor:default'>Install</button>";
  if (c.installed && c.inCatalog) main += "<button class='ghost' data-cuninstall='" + esc(c.fullName) + "' aria-label='Uninstall chip " + esc(c.name) + "' style='margin-top:6px'>Uninstall</button>";
  var head = "<div style='display:flex;gap:8px;align-items:flex-start;flex:none'><div style='flex:1;min-width:0'><b style='display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.35'>" + esc(c.name) + "</b><span class='mono' style='color:var(--dim);font-size:11px;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.4'>" + esc(c.fullName) + "</span></div><span style='flex:0 0 auto'>" + badge + "</span></div>";
  var front = "<div class='flip-face'>" + head
    + (c.description ? "<p style='color:var(--dim);font-size:12px;line-height:1.45;margin:4px 0 0;flex:none;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden'>" + esc(c.description) + "</p>" : "")
    + "<p class='mono' style='color:var(--dim);font-size:11px;line-height:1.4;margin:2px 0 0;flex:none'>★ " + c.stars + " · forks " + c.forks + " · " + esc(c.language) + " · " + c.openIssues + " open issues</p>"
    + "<p class='mono' style='color:var(--dim);font-size:11px;line-height:1.4;margin:2px 0 0;flex:none;white-space:nowrap;overflow:hidden;text-overflow:ellipsis'>" + rel + "</p>"
    + "<div class='flip-actions'><div class='flip-side'>" + links + "</div>"
    + "<div class='flip-main'>" + main + "<button class='ghost' data-flipbtn aria-label='Read the README for " + esc(c.name) + "' style='color:var(--cy)'>README →</button></div></div></div>";
  var backHead = "<div style='display:flex;gap:8px;align-items:flex-start;flex:none'><div style='flex:1;min-width:0'><b style='display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.35'>" + esc(c.name) + "</b><span class='mono' style='color:var(--dim);font-size:11px;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.4'>" + esc(c.fullName) + " · ★ " + c.stars + " · " + esc(c.language) + " · " + c.openIssues + " open issues</span></div><span style='flex:0 0 auto'>" + badge + "</span></div>";
  var back = "<div class='flip-face flip-back'>" + backHead
    + "<div class='flip-scroll md-root' data-readme='" + esc(c.fullName) + "'><p style='color:var(--dim)'>Loading the README from GitHub…</p></div>"
    + "<div class='flip-foot'><a class='ghost' href='" + esc(c.htmlUrl) + "/issues' target='_blank' rel='noreferrer'>Issues ↗</a>"
    + (c.hasWiki ? " <a class='ghost' href='" + esc(c.htmlUrl) + "/wiki' target='_blank' rel='noreferrer'>Documentation ↗</a>" : "")
    + " <button class='ghost' data-flipbtn aria-label='Flip " + esc(c.name) + " back to the front' style='margin-left:auto;color:var(--cy)'>← Front</button></div></div>";
  return "<div class='flip-card' data-flipcard='" + esc(c.fullName) + "'><div class='flip-inner'>" + front + back + "</div></div>";
}
function paintNurReadme(full) {
  var rec = nurReadmes[full];
  document.querySelectorAll("[data-readme]").forEach(function (el) {
    if (el.getAttribute("data-readme") !== full) return;
    if (!rec || rec.state === "loading") el.innerHTML = "<p style='color:var(--dim)'>Loading the README from GitHub…</p>";
    else if (rec.state === "ok") el.innerHTML = mdToHtml(rec.markdown || "");
    else el.innerHTML = "<p style='color:var(--dim)'>" + esc(rec.error || "The README could not be loaded.") + "</p>";
  });
}
function loadNurReadme(full) {
  if (!full) return;
  if (nurReadmes[full]) { paintNurReadme(full); return; }
  nurReadmes[full] = { state: "loading" };
  paintNurReadme(full);
  fetch("/api/chips/readme?repo=" + encodeURIComponent(full)).then(function (r) { return r.json(); }).then(function (j) {
    nurReadmes[full] = j && j.ok ? { state: "ok", markdown: j.markdown || "" } : { state: "error", error: (j && j.error) || "The README could not be loaded." };
    paintNurReadme(full);
  }).catch(function () {
    nurReadmes[full] = { state: "error", error: "GitHub could not be reached — the README could not be loaded." };
    paintNurReadme(full);
  });
}
// One catalog pack's flip card (issue #23): the RAG catalog
// wears the same card as the NUR chips — fixed size, the front
// carries the pack basics and its actions in the same two
// columns (destinations and pack controls left, the primary
// action right with the README button beneath it), and the back
// repeats the basics and scrolls the repo's README inside the
// card, loaded lazily on first flip from /api/rag/readme and
// cached in ragReadmes. Issue #22 lands here too: the bridge
// reconciles installs by slug (forged packs included), so an
// installed pack's card reads INSTALLED and its Install renders
// grayed and dead; an available Update stays live. The pack
// controls that address the install record itself (Check for
// updates, Delete, the Active toggle) ride along only for a
// pack installed from this repo — a slug-matched copy is
// managed on its card under On this machine, and its catalog
// card says so.
var ragReadmes = {};
function ragCardHtml(c) {
  var badge = c.installed ? (c.updateAvailable ? "<span class='mono' style='color:#f5b544;font-size:11px'>UPDATE AVAILABLE</span>" : "<span class='mono' style='color:#34d399;font-size:11px'>INSTALLED" + (c.installedRef ? " · " + esc(c.installedRef) : "") + "</span>") : "<span class='mono' style='color:var(--dim);font-size:11px'>NOT INSTALLED</span>";
  if (c.locallyEdited) badge += " <span class='mono' style='color:#f5b544;font-size:11px'>EDITED LOCALLY</span>";
  var rel = c.latestRelease ? "latest release " + esc(c.latestRelease.tag) + " · " + esc(String(c.latestRelease.publishedAt || "").slice(0, 10)) : "no releases yet — installs track the default branch";
  var exact = c.installed && !c.slugMatched;
  var links = "";
  if (c.htmlUrl) links += "<a class='ghost' href='" + esc(c.htmlUrl) + "' target='_blank' rel='noreferrer'>View on GitHub</a>";
  if (c.latestRelease && c.latestRelease.url) links += " <a class='ghost' href='" + esc(c.latestRelease.url) + "' target='_blank' rel='noreferrer' aria-label='Open release " + esc(c.latestRelease.tag) + " of " + esc(c.name) + " on GitHub'>Release " + esc(c.latestRelease.tag) + "</a>";
  if (exact && !c.updateAvailable) links += " <button class='ghost' data-ragupdate='" + esc(c.fullName) + "'>Check for updates</button>";
  if (exact) links += " <button class='ghost' data-ragdelete='" + esc(c.fullName) + "'>Delete</button>";
  if (exact) links += " <label style='font-size:12px;align-self:center'><input type='checkbox' data-ragtoggle='" + esc(c.fullName) + "'" + (c.active ? " checked" : "") + "> Active — feeds her answers</label>";
  if (c.slugMatched) links += " <span style='color:var(--dim);font-size:11px'>Manage it under On this machine.</span>";
  var main = "";
  if (!c.installed) main += "<button class='act' data-raginstall='" + esc(c.fullName) + "'>Install</button>";
  else if (c.updateAvailable) main += "<button class='act' data-ragupdate='" + esc(c.fullName) + "'>Update" + (c.latestRelease ? " to " + esc(c.latestRelease.tag) : "") + "</button>";
  else main += "<button class='act' disabled aria-label='" + esc(c.name) + " is already installed' style='opacity:.45;cursor:default'>Install</button>";
  var head = "<div style='display:flex;gap:8px;align-items:flex-start;flex:none'><div style='flex:1;min-width:0'><b style='display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.35'>" + esc(c.name) + "</b><span class='mono' style='color:var(--dim);font-size:11px;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.4'>" + esc(c.fullName) + "</span></div><span style='flex:0 0 auto'>" + badge + "</span></div>";
  var front = "<div class='flip-face'>" + head
    + (c.description ? "<p style='color:var(--dim);font-size:12px;line-height:1.45;margin:4px 0 0;flex:none;display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;overflow:hidden'>" + esc(c.description) + "</p>" : "")
    + "<p class='mono' style='color:var(--dim);font-size:11px;line-height:1.4;margin:2px 0 0;flex:none'>★ " + c.stars + " · forks " + c.forks + " · " + esc(c.language) + " · " + c.openIssues + " open issues</p>"
    + "<p class='mono' style='color:var(--dim);font-size:11px;line-height:1.4;margin:2px 0 0;flex:none;white-space:nowrap;overflow:hidden;text-overflow:ellipsis'>" + rel + (c.installed && c.docCount != null ? " · " + c.docCount + " documents on this machine" : "") + "</p>"
    + "<div class='flip-actions'><div class='flip-side'>" + links + "</div>"
    + "<div class='flip-main'>" + main + "<button class='ghost' data-flipbtn aria-label='Read the README for " + esc(c.name) + "' style='color:var(--cy)'>README →</button></div></div></div>";
  var backHead = "<div style='display:flex;gap:8px;align-items:flex-start;flex:none'><div style='flex:1;min-width:0'><b style='display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.35'>" + esc(c.name) + "</b><span class='mono' style='color:var(--dim);font-size:11px;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;line-height:1.4'>" + esc(c.fullName) + " · ★ " + c.stars + " · " + esc(c.language) + " · " + c.openIssues + " open issues</span></div><span style='flex:0 0 auto'>" + badge + "</span></div>";
  var back = "<div class='flip-face flip-back'>" + backHead
    + "<div class='flip-scroll md-root' data-packreadme='" + esc(c.fullName) + "'><p style='color:var(--dim)'>Loading the README from GitHub…</p></div>"
    + "<div class='flip-foot'><a class='ghost' href='" + esc(c.htmlUrl) + "/issues' target='_blank' rel='noreferrer'>Issues ↗</a>"
    + (c.hasWiki ? " <a class='ghost' href='" + esc(c.htmlUrl) + "/wiki' target='_blank' rel='noreferrer'>Documentation ↗</a>" : "")
    + " <button class='ghost' data-flipbtn aria-label='Flip " + esc(c.name) + " back to the front' style='margin-left:auto;color:var(--cy)'>← Front</button></div></div>";
  return "<div class='flip-card' data-ragflipcard='" + esc(c.fullName) + "'><div class='flip-inner'>" + front + back + "</div></div>";
}
function paintRagReadme(full) {
  var rec = ragReadmes[full];
  document.querySelectorAll("[data-packreadme]").forEach(function (el) {
    if (el.getAttribute("data-packreadme") !== full) return;
    if (!rec || rec.state === "loading") el.innerHTML = "<p style='color:var(--dim)'>Loading the README from GitHub…</p>";
    else if (rec.state === "ok") el.innerHTML = mdToHtml(rec.markdown || "");
    else el.innerHTML = "<p style='color:var(--dim)'>" + esc(rec.error || "The README could not be loaded.") + "</p>";
  });
}
function loadRagReadme(full) {
  if (!full) return;
  if (ragReadmes[full]) { paintRagReadme(full); return; }
  ragReadmes[full] = { state: "loading" };
  paintRagReadme(full);
  fetch("/api/rag/readme?repo=" + encodeURIComponent(full)).then(function (r) { return r.json(); }).then(function (j) {
    ragReadmes[full] = j && j.ok ? { state: "ok", markdown: j.markdown || "" } : { state: "error", error: (j && j.error) || "The README could not be loaded." };
    paintRagReadme(full);
  }).catch(function () {
    ragReadmes[full] = { state: "error", error: "GitHub could not be reached — the README could not be loaded." };
    paintRagReadme(full);
  });
}
function nurIsSaints(c) { return String(c.fullName || "").split("/")[0].toLowerCase() === "saintssec"; }
function nurMatches(c, q) { return (String(c.name || "") + " " + String(c.fullName || "") + " " + String(c.description || "")).toLowerCase().indexOf(q) !== -1; }
// The NUR list for the active topic tab: Saints Sec Chips is
// SaintsSec's own shelf; Community Chips is everyone else's —
// filtered by the search text while a search is entered, the
// whole general list otherwise.
// Load one chip from the folder into Chip Studio — the single
// path behind the Studio's own "Open a chip…" picker and the
// Edit in Studio buttons on chip cards, so the draft is built
// exactly the same way from either door: the source read, the
// files list (root.py synthesized when the chip carries a
// single file), and the snapshot that marks the draft clean.
// onOk runs once the draft is in (the cards use it to land on
// the Studio tab); onErr receives the failure in words — where
// it is shown is the caller's choice, and it is never swallowed.
function loadChipIntoStudio(file, onOk, onErr) {
  fetch("/api/chips/source?file=" + encodeURIComponent(file)).then(function (r) { return r.json(); }).then(function (j) {
    if (j.ok) {
      var fs = (j.files && j.files.length) ? j.files : [{ path: "root.py", content: j.code }];
      studioDraft = { file: j.file, name: j.name, description: j.description, code: j.code, output: j.output, files: fs, active: "root.py", folds: [], snapshot: "", icon: (j.icon && j.icon.length) ? j.icon.join("\n") : "" };
      studioDraft.snapshot = studioSnapshot(studioDraft);
      studioMsg = "Editing " + j.name + " — saving re-forges its chip with the new code inside."; studioMade = null; studioExtras = [];
      if (onOk) onOk(j);
    }
    else if (onErr) onErr(j.error || "That chip could not be read.");
    render();
  }).catch(function () { if (onErr) onErr("The bridge could not be reached."); render(); });
}
// One chip from the local folder (chips/ on this machine) as a
// card — shared by the Local Chips tab's two groups. This card is
// the chip's single management home (issue #21): every action on
// it is a button; the .png download is a button-styled link.
// Same two-column split as the catalog cards: the management
// controls sit in the left column, and the right column —
// anchored to the card's right side — carries the primary action
// (Run) on top with Download .png directly beneath it.
function localChipCardHtml(c) {
  var stem = String(c.file || "").replace(/.png$/i, "");
  var exp = chipExp[c.file];
  var pub = chipPub[c.file];
  var red = chipReadmeEd[c.file];
  // The chip's catalog record, when the catalog has been searched:
  // the installed entry whose local file is this chip's file. It
  // carries the update state for GitHub installs, and its full
  // name is a forged chip's delete address (the bridge keeps a
  // local/… record for forged chips); without one, a forged chip
  // deletes by "local/" + its file stem, which the bridge
  // resolves the same way.
  var cat = null;
  if (chipCatalog && chipCatalog.chips) {
    for (var ci = 0; ci < chipCatalog.chips.length; ci++) {
      if (chipCatalog.chips[ci].localFile && chipCatalog.chips[ci].localFile === c.file) { cat = chipCatalog.chips[ci]; break; }
    }
  }
  var delName = c.sourceKey ? c.sourceKey : (cat ? cat.fullName : "local/" + stem);
  var reqLine = (c.requirements && c.requirements.length)
    ? "<div style='color:var(--dim);font-size:12px;margin-top:4px'>Declares dependencies: " + esc(c.requirements.join(", ")) + " — not installed automatically</div>"
    : "";
  if (c.unreadable) {
    // Issue #79 (build .86): a cartridge that does not parse is
    // shown honestly — it can't run or be edited, it is skipped
    // by the pill row and the runners, but it is still here, and
    // still deletable, instead of silently absent.
    return "<div class='card' style='flex:0 0 auto'><b>" + esc(c.name) + "</b> <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(c.file) + "</span>" +
      "<p style='color:#f5b544;font-size:12px'>This chip's cartridge couldn't be read — the .png is in the chips folder, but the code inside it doesn't parse, so it can't run. It still shows here so it can be deleted.</p>" +
      reqLine +
      "<div style='margin-top:6px'><button class='ghost' data-cdelete='" + esc(delName) + "' aria-label='Delete chip " + esc(c.name) + "'>Delete</button></div></div>";
  }
  // Update actions belong to GitHub-installed chips only — a
  // forged chip has no repo to check against. The button rides
  // the same data-cupdate path the catalog card used: the update
  // endpoint checks live, so "Check for updates" is right even
  // when the catalog has not returned this chip's repo.
  var upd = "";
  if (c.sourceKey) {
    upd = (cat && cat.updateAvailable)
      ? "<button class='ghost' data-cupdate='" + esc(c.sourceKey) + "' aria-label='Update chip " + esc(c.name) + "'>Update" + (cat.latestRelease ? " to " + esc(cat.latestRelease.tag) : "") + "</button>"
      : "<button class='ghost' data-cupdate='" + esc(c.sourceKey) + "' aria-label='Check chip " + esc(c.name) + " for updates'>Check for updates</button>";
  }
  return "<div class='card'><b>" + esc(c.name) + "</b> <span class='mono' style='color:var(--dim);font-size:11px'>" + esc(c.file) + "</span>" + (c.locallyEdited ? " <span class='mono' style='color:#f5b544;font-size:11px'>EDITED LOCALLY" + (c.sourceKey ? " · DIVERGES FROM " + esc(c.sourceKey) : "") + "</span>" : (c.sourceKey ? " <span class='mono' style='color:var(--dim);font-size:11px'>INSTALLED FROM " + esc(c.sourceKey) + "</span>" : ""))
  + "<div class='lchip-actions'><div class='lchip-side'><label style='color:var(--dim);font-size:12px'>Version <input id='chipver-" + esc(stem) + "' type='text' value='0.1.0' style='width:80px' aria-label='Export version for chip " + esc(c.name) + "'></label><button class='ghost' data-chipexport='" + esc(c.file) + "' aria-label='Export chip " + esc(c.name) + " for GitHub'>Export for GitHub</button><button class='ghost' data-chippub='" + esc(c.file) + "' aria-label='Publish chip " + esc(c.name) + " to GitHub'>Publish to GitHub</button><button class='ghost' data-chipreadme='" + esc(c.file) + "' aria-label='Edit README for chip " + esc(c.name) + "'>Edit README</button><button class='ghost' data-chipedit='" + esc(c.file) + "' aria-label='Edit chip " + esc(c.name) + " in Chip Studio'>Edit in Studio</button>"
  + upd + "<button class='ghost' data-cdelete='" + esc(delName) + "' aria-label='Delete chip " + esc(c.name) + "'>Delete</button>"
  + reqLine
  + (exp ? (exp.error ? "<span style='color:#f5b544;font-size:12px'>" + esc(exp.error) + "</span>" : "<span style='font-size:12px'>Repo ZIP ready: <a class='ghost' href='/api/chips/export-file?name=" + encodeURIComponent(exp.file) + "' download='" + esc(exp.file) + "'>" + esc(exp.file) + "</a></span><span class='mono' style='color:var(--dim);font-size:11px'>" + exp.files.map(function (f) { return esc(f); }).join(" · ") + "</span>") : "")
  + "</div><div class='lchip-main'><button class='act' data-lchip='" + esc(stem) + "'>Run</button><a class='ghost' href='/api/chips/file?name=" + encodeURIComponent(c.file) + "' download='" + esc(c.file) + "'>Download .png</a></div></div>"
  + pubPanelHtml(pub, "chip", c.file) + readmeEdHtml(red, "chip", c.file)
  + "<pre class='out mono' id='lchip-" + esc(stem) + "' style='display:none'></pre></div>";
}
// The Local Chips tab: what's on this machine, in two groups —
// Installed (pulled from the catalogs) and Forged (made here).
function nurLocalHtml() {
  if (!wfData) return "<p style='color:var(--dim)'>Loading the chips on this machine…</p>";
  var chips = wfData.chips || [];
  var installed = chips.filter(function (c) { return c.source === "github"; });
  var forged = chips.filter(function (c) { return c.source !== "github"; });
  var h = "<p style='color:var(--dim)'>Every chip on this machine is a single .png with its whole codebase embedded inside the image. Download a chip, share the image, load it back: the code travels with the picture. This tab is the one place to manage the chips you have — each card carries every management action — while the Saints Sec and Community tabs are for finding and installing new ones.</p>";
  h += "<h3 style='margin-top:14px'>Installed · " + installed.length + "</h3>";
  h += installed.length ? installed.map(localChipCardHtml).join("") : "<p style='color:var(--dim)'>Nothing installed yet — install a chip from the Saints Sec or Community tabs and it lands here.</p>";
  h += "<h3 style='margin-top:16px'>Forged · " + forged.length + "</h3>";
  h += forged.length ? forged.map(localChipCardHtml).join("") : "<p style='color:var(--dim)'>Nothing forged yet — forge a chip in Chip Studio, or drop a chip .png into the chips folder, and it lands here.</p>";
  return h;
}
function nurCardsHtml() {
  if (nurTab === "local") return nurLocalHtml();
  if (!chipCatalog || !chipCatalog.chips) return "";
  // Only true catalog entries render in the grid (issue #21):
  // the payload's appended install-summary entries (inCatalog
  // false) are the Local Chips tab's business now, never a
  // second card here — an installed chip shows once, as its
  // repo entry in its installed state.
  var list = chipCatalog.chips.filter(function (c) { return c.inCatalog && (nurTab === "saintssec" ? nurIsSaints(c) : !nurIsSaints(c)); });
  if (nurTab === "community") {
    var q = nurQuery.trim().toLowerCase();
    if (q) list = list.filter(function (c) { return nurMatches(c, q); });
  }
  if (!list.length) {
    if (!chipCatalog.chips.some(function (c) { return c.inCatalog; })) return "<p style='color:var(--dim)'>No repos carry the navi-chips topic right now.</p>";
    if (nurTab === "community" && nurQuery.trim()) return "<p style='color:var(--dim)'>No community chips match “" + esc(nurQuery.trim()) + "” — clear the search to see the whole list.</p>";
    return nurTab === "saintssec"
      ? "<p style='color:var(--dim)'>No SaintsSec chips in the catalog yet — Saints Sec publishes under the SaintsSec name on GitHub.</p>"
      : "<p style='color:var(--dim)'>No community chips in the catalog yet — every navi-chips repo outside SaintsSec lands here.</p>";
  }
  return "<div class='nur-grid'>" + list.map(nurCardHtml).join("") + "</div>";
}
// The "N of M community chips match …" line under the Community
// search — empty unless a search is actually entered.
function nurMetaText() {
  if (nurTab !== "community" || !nurQuery.trim() || !chipCatalog || !chipCatalog.chips) return "";
  var q = nurQuery.trim().toLowerCase();
  var comm = chipCatalog.chips.filter(function (c) { return c.inCatalog && !nurIsSaints(c); });
  var n = comm.filter(function (c) { return nurMatches(c, q); }).length;
  return n + " of " + comm.length + " community chips match “" + nurQuery.trim() + "”.";
}

function githubViewHtml() {
  // The GitHub section is her pulse window: a peek into the project
  // itself - AlexKollar/Apex, live from the GitHub API through
  // the bridge. Four tabs: Overview (the pulse), Issues (the live
  // open-issues list), Roadmap (Alex's project board, a webview
  // window in the shell) and Discussions (the project's forum,
  // likewise). Account and backup live in Settings -> GitHub &
  // Backup; this section is for looking at the project.
  var h = "<h3>GitHub</h3>";
  h += "<div role='tablist' aria-label='GitHub section tabs' class='tabbar' style='margin-bottom:10px'>" +
    "<button type='button' role='tab' aria-selected='" + (githubTab === "overview") + "' class='" + (githubTab === "overview" ? "act" : "ghost") + "' id='gh-tab-overview'>Overview</button>" +
    "<button type='button' role='tab' aria-selected='" + (githubTab === "issues") + "' class='" + (githubTab === "issues" ? "act" : "ghost") + "' id='gh-tab-issues'>Issues</button>" +
    "<button type='button' role='tab' aria-selected='" + (githubTab === "roadmap") + "' class='" + (githubTab === "roadmap" ? "act" : "ghost") + "' id='gh-tab-roadmap'>Roadmap</button>" +
    "<button type='button' role='tab' aria-selected='" + (githubTab === "discussions") + "' class='" + (githubTab === "discussions" ? "act" : "ghost") + "' id='gh-tab-discussions'>Discussions</button></div>";
  if (githubTab === "roadmap" || githubTab === "discussions") {
    h += githubWebviewTabHtml(githubTab);
    return h;
  }
  if (githubTab === "issues") {
    h += "<div class='card'><h3>Issues — AlexKollar/Apex</h3><p style='color:var(--dim)'>The repo's current open issues, live from GitHub. Looking needs <b>no account at all</b> — and neither does this list. Clicking an issue opens it on GitHub, outside her window; filing happens there, with your own account.</p>";
    h += "<button class='ghost' id='gh-issues-refresh'>Refresh</button> <button class='ghost' id='gh-issues-page'>Open the GitHub Issues page ↗</button>";
    if (githubIssuesFetching && !githubIssues) h += "<p style='color:var(--dim);margin-top:8px'>Loading issues…</p>";
    else if (githubIssuesError && !githubIssues) h += "<div class='card' style='border-color:#f87171;margin-top:8px'><b style='color:#f87171'>Couldn't load the issues list.</b><br><span style='color:var(--dim)'>" + esc(githubIssuesError) + "</span></div>";
    else if (githubIssues) {
      if (githubIssuesError) h += "<p style='color:#f5b544;font-size:12px;margin-top:6px'>Refresh failed — showing the last list she loaded. " + esc(githubIssuesError) + "</p>";
      h += githubIssues.length ? githubIssueRowsHtml(githubIssues) : "<p style='color:var(--dim);margin-top:8px'>No open issues right now — the queue is clear.</p>";
      h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:8px'>" + githubIssues.length + " open · fetched " + esc(ghWhen(githubIssuesFetchedAt) || "just now") + "</p>";
    }
    h += ghFooterHtml("File an issue", NAVI_REPO_URL + "/issues/new");
    h += "</div>";
    return h;
  }
  var pu = githubPulse;
  h += "<div class='card'><div style='display:flex;align-items:center;gap:8px;flex-wrap:wrap'><h3 style='margin:0'>Project pulse</h3><button class='ghost' id='gh-pulse-refresh' style='margin-left:auto'>Refresh</button></div>";
  if (githubPulseFetching && !pu) h += "<p style='color:var(--dim)'>Reading the repo…</p>";
  else if (githubPulseError && !pu) h += "<div class='card' style='border-color:#f87171;margin-top:8px'><b style='color:#f87171'>Couldn't read the repo.</b><br><span style='color:var(--dim)'>" + esc(githubPulseError) + " Public GitHub allows a modest number of unauthenticated reads per hour — if this is a rate limit, it clears on its own.</span></div>";
  else if (pu) {
    if (githubPulseError) h += "<p style='color:#f5b544;font-size:12px'>Refresh failed — showing the last pulse she read. " + esc(githubPulseError) + "</p>";
    if (pu.description) h += "<p style='color:var(--text);margin-top:6px'>" + esc(pu.description) + "</p>";
    h += "<div style='display:flex;gap:14px;flex-wrap:wrap;margin-top:8px'>" +
      "<span><b>" + pu.stars + "</b> <span style='color:var(--dim)'>stars</span></span>" +
      "<span><b>" + pu.forks + "</b> <span style='color:var(--dim)'>forks</span></span>" +
      "<span><b>" + pu.watchers + "</b> <span style='color:var(--dim)'>watchers</span></span>" +
      "<span><b>" + pu.openIssuesCount + "</b> <span style='color:var(--dim)'>open issues</span></span>" +
      (pu.commitCount !== null && pu.commitCount !== undefined ? "<span><b>" + pu.commitCount + "</b> <span style='color:var(--dim)'>commits</span></span>" : "") +
      "<span><b>" + esc(pu.language) + "</b> <span style='color:var(--dim)'>language</span></span>" +
      "<span><b>" + esc(pu.defaultBranch) + "</b> <span style='color:var(--dim)'>default branch</span></span></div>";
    h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:8px'>LAST PUSH · " + esc(ghWhen(pu.pushedAt) || "—") + "</p>";
    // Issue #25: the pulse's own entries are cards too, in the
    // issue cards' frame. They are not issues, so their clicks
    // keep the guarded openExternal route (their URLs never
    // match the #24 issue shape) — only the shape changes.
    if (pu.latestRelease) h += "<button type='button' data-gh-issue='" + esc(pu.latestRelease.url) + "' style='display:block;width:100%;text-align:left;background:var(--panel);border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-top:8px;cursor:pointer;color:var(--text);font:inherit' aria-label='Open release " + esc(pu.latestRelease.tag) + " on GitHub'><span class='mono' style='color:var(--dim);font-size:11px'>LATEST RELEASE</span><br><b>" + esc(pu.latestRelease.tag) + (pu.latestRelease.name ? " — " + esc(pu.latestRelease.name) : "") + "</b> <span style='color:var(--dim);font-size:12px'>" + esc(ghWhen(pu.latestRelease.publishedAt)) + "</span></button>";
    else h += "<p style='color:var(--dim);margin-top:6px'>No releases yet.</p>";
    if (pu.commits && pu.commits.length) {
      h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:10px'>RECENT COMMITS</p>" + pu.commits.map(function (c) {
        return "<button type='button' data-gh-issue='" + esc(c.url) + "' style='display:block;width:100%;text-align:left;background:var(--panel);border:1px solid var(--line);border-radius:8px;padding:10px 12px;margin-top:8px;cursor:pointer;color:var(--text);font:inherit' aria-label='Open commit " + esc(c.sha) + " on GitHub'><span class='mono' style='color:var(--cy);font-size:12px'>" + esc(c.sha) + "</span> <span>" + esc(c.message) + "</span><br><span style='color:var(--dim);font-size:12px'>" + esc(c.author) + (c.date ? " · " + esc(ghWhen(c.date)) : "") + "</span></button>";
      }).join("");
    }
    if (pu.issues && pu.issues.length) {
      h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:10px'>OPEN ISSUES</p>" + githubIssueRowsHtml(pu.issues);
    }
    h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:8px'>Fetched " + esc(ghWhen(githubPulseFetchedAt) || "just now") + " · live from the GitHub API</p>";
  }
  h += "</div>";
  h += "<div class='card'><h3>The repo</h3><p style='color:var(--dim)'>Apex lives at <span class='mono'>AlexKollar/Apex</span> — her fresh home, started for this rebuild. The original Navi she grew from stays at SaintsSec/Navi as the legacy archive. Account things — sign-in, backup, import — live in <b>Settings → GitHub &amp; Backup</b>; this section is for looking at the project.</p><button class='act' id='gh-repo'>The repo itself ↗</button> <button class='ghost' id='gh-issues-page'>GitHub Issues page ↗</button> <button class='ghost' id='gh-legacy'>Original Navi (legacy) ↗</button></div>";
  return h;
}

function githubSettingsHtml() {
  // Settings -> GitHub & Backup: sign-in and the whole backup/import
  // block live here now (moved from the GitHub view, which is the
  // project pulse). Same ids, same wiring - one home.
  var gh = githubState;
  var h = "";
  if (!gh) { return "<div class='card'>Loading GitHub state…</div>"; }
  if (gh.liveError) h += "<div class='card' style='border-color:#f5b544;color:#f5b544'>" + esc(gh.liveError) + "</div>";
  // Sign-in card.
  h += "<div class='card'><h3>Sign-in — optional, always</h3><p style='color:var(--dim)'>Navi <b>never</b> requires a GitHub sign-in: she runs, chats, works offline, and uses her chips and packs with no account at all. Sign-in exists only for the GitHub-backed actions — publishing, backup, and import. It uses GitHub's <b>Device Flow</b>: GitHub gives you a short code, you enter it in your own browser, and GitHub tells the app you're in. No password ever passes through the app, and no client secret is embedded in it." + (CONFIG.backend === "live" ? " The token is kept only on this machine (config/github-auth.json, readable only by you) — never in the backup, never logged." : " In mock mode the flow is simulated; the live bridge runs it against GitHub for real.") + "</p><p style='color:var(--dim)'><b>What signing in turns on:</b></p><ul style='color:var(--dim);margin:4px 0 4px 18px'><li><b>Publish to GitHub</b> for your RAG packs and chips — Navi does the whole chain for you — repo, navi-rag / navi-chips topic, files, tag, and the published release with the picture attached.</li><li><b>Automatic backups</b> to a repo under your own account, on the schedule you pick below.</li><li><b>Taking part</b> in the GitHub section's live issues, roadmap, and discussions — looking never needed an account; joining in does.</li></ul><p style='color:var(--dim)'>And the standing truth: none of it is required. <b>Export for GitHub</b> — packaging your packs and chips as a repo-ready ZIP — is always available, signed in or not. Sign-in only adds the upload route; it never gates exporting or publishing by hand.</p>";
  if (gh.signedIn) {
    h += "<p class='mono' style='font-size:13px;color:var(--green)'>Signed in as @" + esc(gh.login) + (CONFIG.backend !== "live" ? " <span style='color:var(--dim);font-size:11px'>(simulated account — mock mode has no real GitHub behind it)</span>" : "") + "</p><button class='ghost' id='gh-signout'>Sign out</button>";
  } else if (githubDevice) {
    h += "<p style='color:var(--dim)'>Enter this code at <a href='https://github.com/login/device' target='_blank' rel='noreferrer'>github.com/login/device</a>, then " + (CONFIG.backend === "live" ? "wait here — this page checks with GitHub on its own and signs you in when GitHub confirms." : "finish here:") + "</p>";
    h += "<p class='mono' style='font-size:22px;font-weight:700;letter-spacing:.3em;color:var(--cy);border:1px solid var(--cy);display:inline-block;padding:6px 14px;border-radius:6px'>" + esc(githubDevice.userCode) + "</p><br>";
    if (CONFIG.backend !== "live") h += "<button class='act' id='gh-complete'>I've entered the code — finish sign-in</button> ";
    h += "<button class='ghost' id='gh-cancel'>Cancel</button>";
    if (CONFIG.backend !== "live") h += "<p style='color:var(--dim);font-size:12px'>Simulated: the button stands in for GitHub's answer. The live bridge polls GitHub itself until it confirms, the code expires, or you cancel.</p>";
  } else if (!gh.clientIdConfigured) {
    h += "<div class='card' style='border-color:var(--am);color:var(--am)'>No GitHub OAuth client id is configured, so sign-in can't start. Apex ships with the project's client id in <span class='mono'>config/github-signin.json</span> — if that file is missing or its client_id is blank, set one there (an OAuth App with Device Flow enabled; the client id is a public identifier, not a secret). The GitHub section's pulse and Issues tabs work regardless.</div>";
  } else {
    h += "<button class='act' id='gh-signin'>Sign in with GitHub</button> <span class='mono' style='color:var(--dim);font-size:11px'>one sign-in for the whole product — no tokens to create or paste · client id: " + esc(gh.clientId) + "</span>";
  }
  h += "</div>";
  // Identities card (Build 2, #173; Build 6, #173 phase 3):
  // ONE credential now — the operator's own GitHub sign-in.
  // Build 2 split the app's reads onto a separate Runtime PAT
  // the operator had to create and paste; Build 6 retired it:
  // sign in once, with your account, and the reads ride the same
  // token as backup and import. Release Automation still holds
  // no credential in Apex, by design. Failures anywhere GitHub
  // is read keep their names: auth, rate (with its reset), or
  // abuse (with its wait) — never a bare HTTP 403.
  var ids = gh.identities || null;
  var idOp = ids ? ids.operator : { signedIn: gh.signedIn, login: gh.login, scopesNote: "Acts as you — the only GitHub credential Apex holds. Used by backup and import, and by the app's own reads (Updater, NUR, the GitHub section).", buckets: { core: null, search: null } };
  var idAu = ids ? ids.automation : { note: "Releases are published from the operator's signed-in browser session and the local git remote. Apex holds no release credential — by design." };
  h += "<div class='card'><h3>Identities — who Apex speaks to GitHub as</h3><p style='color:var(--dim)'>One credential, honestly labelled. Earlier builds split Apex's speech three ways — and asked you to create and paste a separate token for the app's own reads. That token is retired: <b>your sign-in above is the only GitHub credential Apex holds</b>, and everything rides it. Signed out, the app's reads go unauthenticated and share the small public allowance (60 requests an hour per network address, 10 a minute for search) — and their failures say so, in words.</p>";
  h += "<p style='margin-top:10px'><b>1 · Operator</b> — your sign-in. <span style='color:var(--dim)'>" + esc(idOp.scopesNote || "") + "</span></p><p class='mono' style='font-size:12px;color:" + (idOp.signedIn ? "var(--green)" : "var(--dim)") + "'>" + (idOp.signedIn ? "Signed in as @" + esc(idOp.login) : "Not signed in — GitHub-backed actions will ask you to sign in; everything local works regardless") + "</p>";
  if (idOp.signedIn) {
    var bk = idOp.buckets || { core: null, search: null };
    h += ghBucketLine("CORE", bk.core);
    h += ghBucketLine("SEARCH", bk.search);
  }
  h += "<p style='margin-top:10px'><b>2 · Release Automation</b> — <span style='color:var(--dim)'>" + esc(idAu.note || "") + "</span></p>";
  h += "<p style='color:var(--dim);font-size:12px'>And failures have names: <b>auth</b> (the credential itself is refused), <b>rate</b> (a bucket is spent — the message carries its reset time), <b>abuse</b> (GitHub's secondary safeguard — the message carries the wait it asked for, and Apex stops rather than retrying). Wherever GitHub results are shown — the Updater, the NUR, this section — the name is what you will read.</p></div>";
  // Backup card.
  var unticked = GH_ITEMS.filter(function (it) { return gh.backupItems.indexOf(it.id) === -1; });
  h += "<div class='card'><h3>Backup — the ready-to-fly essentials</h3><p style='color:var(--dim)'>A backup carries <b>only what a fresh Navi needs to be fully herself</b>: fresh install + import = ready to fly. Chats ride on their own tick now — they live on the server (Build 12), and a restore <i>converges</i> them instead of overwriting: newest wins per thread, and a thread both machines moved on is kept whole as a dated sibling. Deliberately never backed up: call logs, caches and logs, the hardware profile (a new machine gets profiled fresh at its own install), and her voice model files (they reinstall — her voice <i>settings</i> are in the backup). Backups run while Navi is running: the bridge checks on launch and while she's up whether one is due, on your interval, and runs it when she's signed in. Nothing backs up while the machine is off.</p>";
  h += "<label>Backup repository (under your account)<br><input class='mono' id='gh-repo' type='text' value='" + esc(gh.backupRepo) + "' aria-label='Backup repository name'></label>";
  h += "<label>Visibility<br><select id='gh-vis' aria-label='Backup repository visibility'><option value='private'" + (gh.backupPrivate ? " selected" : "") + ">Private — only you can see it (recommended)</option><option value='public'" + (!gh.backupPrivate ? " selected" : "") + ">Public — anyone can read it</option></select></label>";
  h += "<div id='gh-pubwarn'></div>";
  h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:10px'>THE ESSENTIALS CHECKLIST</p>";
  h += "<p style='color:var(--dim);font-size:12px'>Soul, Memory, and Settings &amp; config are ticked automatically — every backup carries them. The rest are your choice.</p>";
  GH_ITEMS.forEach(function (it) {
    if (it.core) {
      h += "<label style='display:block;margin:4px 0'><input type='checkbox' id='gh-item-" + it.id + "' checked disabled> <b>" + esc(it.label) + "</b> <span style='color:var(--dim)'>— " + esc(it.blurb) + "</span> <span class='mono' style='color:var(--dim);font-size:11px'>ALWAYS BACKED UP</span></label>";
    } else {
      h += "<label style='display:block;margin:4px 0'><input type='checkbox' id='gh-item-" + it.id + "'" + (gh.backupItems.indexOf(it.id) !== -1 ? " checked" : "") + "> <b>" + esc(it.label) + "</b> <span style='color:var(--dim)'>— " + esc(it.blurb) + "</span></label>";
    }
  });
  if (unticked.length) h += "<div class='card' style='border-color:var(--am);color:var(--am)'>Unticked: " + unticked.map(function (u) { return esc(u.label); }).join(", ") + ". You can back up without them — but an import from that backup will <b>not</b> be fully ready-to-fly: the unticked essentials stay behind.</div>";
  h += "<label>Back up automatically<br><select id='gh-interval' aria-label='Automatic backup interval'><option value='1'" + (gh.intervalDays === 1 ? " selected" : "") + ">Every day</option><option value='3'" + (gh.intervalDays === 3 ? " selected" : "") + ">Every 3 days</option><option value='5'" + (gh.intervalDays === 5 ? " selected" : "") + ">Every 5 days</option><option value='7'" + (gh.intervalDays === 7 ? " selected" : "") + ">Every 7 days</option></select></label>";
  h += "<div style='margin-top:8px'><button class='ghost' id='gh-settings-save'>Save backup settings</button> <button class='act' id='gh-backup'" + (gh.signedIn ? "" : " disabled title='Sign in above first'") + ">Back up now</button></div>";
  if (!gh.signedIn) h += "<p style='color:var(--dim);font-size:12px'>Back up now needs a sign-in — writing to GitHub is one of the actions that does.</p>";
  h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:8px'>LAST BACKUP · " + (gh.lastBackupAt ? ghWhen(gh.lastBackupAt) + " — " + esc(gh.lastBackupResult || "") : "never yet") + "</p></div>";
  // The sync override's setting (v0.6.2e, #303): the switch
  // renders from the bridge's stored choice alone; a press
  // asks the bridge to store the new value and the re-render
  // reads the answer — flip, stored state and reload are one
  // truth, and a refusal stands on this card, switch unmoved.
  if (CONFIG.backend === "live" && syncChoiceState === null && !syncChoiceFetching) fetchSyncChoice();
  var scOn = syncChoiceState !== false;
  h += "<div class='card'><h3>The Sync button — Send or Receive</h3><p style='color:var(--dim)'>The <b>Sync</b> control in Utilities, with this on, offers a choice when you press it: <b>Send a sync</b> (publish this seat now, pull nothing) or <b>Receive a sync</b> (pull the converged set now, publish nothing) — a manual override, so a sync happens when you say, not when the cadence gets to it. Turn it off and a press runs the full round at once, as it always did. The automatic rounds — app start, DHD sessions, Mesh contact, push-on-write — never consult this setting.</p>";
  h += "<div style='display:flex;align-items:center;gap:8px;margin-top:8px'><button class='switch' id='gh-sync-choice' role='switch' aria-checked='" + (scOn ? "true" : "false") + "' aria-label='Sync button offers Send or Receive'><span class='knob'></span></button><span><b>" + (scOn ? "On — a press offers Send / Receive" : "Off — a press runs the full round") + "</b></span></div>";
  if (syncChoiceMsg) h += "<p style='color:var(--am);font-size:12px;margin-top:6px'>" + esc(syncChoiceMsg) + "</p>";
  h += "</div>";
  // Import card.
  h += "<div class='card'><h3>Import — bring a Navi back</h3><p style='color:var(--dim)'>Import pulls the backup down and shows you, essential by essential, what would change — <b>nothing is overwritten until you confirm</b>. Restored packs come back with their indexes and active states; restored chips come back whole and runnable. What's not in the backup (call logs, the old machine's hardware profile, voice models) simply starts fresh, the way a new install should.</p>";
  h += "<button class='ghost' id='gh-preview'" + (gh.signedIn ? "" : " disabled title='Sign in above first'") + ">Preview what would change</button>";
  if (githubPreview && githubPreview.ok && !githubApplied) h += " <button class='act' id='gh-apply'>Confirm import — overwrite as shown</button>";
  if (githubPreview && !githubPreview.ok && githubPreview.error) h += "<p style='color:#f87171'>" + esc(githubPreview.error) + "</p>";
  var changes = githubApplied && githubApplied.ok ? null : (githubPreview && githubPreview.ok ? githubPreview.changes : null);
  if (changes) {
    h += "<div style='margin-top:8px'>" + changes.map(function (c) {
      return "<div class='card'><b>" + esc(c.label) + "</b> <span class='mono' style='font-size:11px;color:" + (c.willChange ? "var(--am)" : "var(--dim)") + "'>" + (c.willChange ? "WOULD CHANGE" : "NO CHANGE") + "</span><br><span style='color:var(--dim)'>" + esc(c.detail) + "</span></div>";
    }).join("") + "</div>";
  }
  if (githubApplied && githubApplied.ok) h += "<p style='color:var(--green)'>" + esc(githubApplied.result || "Import complete.") + "</p>";
  h += "<p class='mono' style='color:var(--dim);font-size:11px;margin-top:8px'>LAST IMPORT · " + (gh.lastImportAt ? ghWhen(gh.lastImportAt) + " — " + esc(gh.lastImportResult || "") : "never yet") + "</p></div>";
  // Account sync card (Build 12, #136; Build 13, #136/#150):
  // truthful per-state labels, the bootstrap path in writing,
  // the same real file slot the Operator room has (one shared
  // intake, one badge state), and a sync that goes to Chat at
  // press time under a timeout instead of spinning here. The
  // Operator Badge's
  // functional home is here, with the account machinery — slotting
  // it pulls the latest backup into a preview on a live operation
  // card in Chat, and nothing lands until the operator confirms
  // there. The badge authenticates nothing; the sign-in is the key.
  h += "<div class='card'><h3>Account sync — slot your Operator Badge</h3><p style='color:var(--dim)'>Your <b>Operator Badge</b> is a pointer, never a key: slotting it tells Navi <i>whose</i> backup to pull, and the GitHub sign-in above remains the only thing that can unlock it. Slotting the badge pulls the latest backup and lays its per-item preview on a <b>live operation card in Chat</b> — pressing the button takes you to Chat at once, where the card runs: Navi speaks before it and after it, nothing is generated while it runs, and <b>nothing lands until you confirm on the card</b>. A pull that does not answer within 45 seconds says so in plain words — this card never spins forever. Chats converge rather than overwrite, and her node keypair and Mesh certificate never ride a backup, so they can never ride a sync.</p>";
  var syncBtnDis = (!gh.signedIn || badgeSyncBusy) ? " disabled" + (!gh.signedIn ? " title='Sign in above first'" : "") : "";
  var syncBtnLabel = badgeSyncBusy ? "Pulling the latest backup…" : (CONFIG.operatorBadge ? "Slot the badge — pull the latest backup" : "Pull the latest backup — no badge slotted");
  if (CONFIG.operatorBadge) {
    h += "<div style='display:flex;gap:14px;align-items:center;flex-wrap:wrap'><img src='" + esc(CONFIG.operatorBadge) + "' alt='Your Operator Badge' style='width:88px;border-radius:8px'><div><button class='act' id='gh-badge-sync' style='min-height:44px'" + syncBtnDis + ">" + syncBtnLabel + "</button>" + (gh.signedIn ? "" : "<p style='color:var(--dim);font-size:12px'>Signed out, slotting the badge stops at the sign-in step and fetches nothing — sign in above first.</p>") + "</div></div>";
  } else {
    h += "<p style='color:var(--dim)'>No badge slotted on this machine yet. Two ways in: <b>mint one here</b> — in Settings → Operator — then <b>Back up now</b> above, and the badge rides that backup to your other machines; or <b>slot a badge picture</b> minted on another machine — right here, or in Settings → Operator; the badge file alone is enough, and you can also pull a backup that already carries one. The sync itself needs only the sign-in; the badge is how you point it.</p><button class='act' id='gh-badge-sync' style='min-height:44px'" + syncBtnDis + ">" + syncBtnLabel + "</button>";
  }
  h += "<div class='row' style='margin-top:8px'><button class='ghost' id='gh-badge-slot'>Slot a badge from a file…</button><input id='gh-badge-file' type='file' accept='image/png' style='display:none' aria-label='Choose an operator badge picture'></div>";
  h += "</div>";
  if (githubMsg) h += "<p style='font-size:13px;color:" + (githubError ? "#f87171" : "var(--green)") + "'>" + esc(githubMsg) + "</p>";
  return h;
}

// The Voice home (Build 6, #113): the Voice VIEW is retired
// and everything it held lives on this one Settings tab, in
// this order — the call panel, the pipeline strip, her voice
// settings card, the engines-not-installed guidance, the
// desktop-app microphone known-issue card (moved VERBATIM,
// per-session mic line and all), the mock-mode notice, and
// the past-calls list. The sections below are the view's own
// markup, moved whole; the call object is global, so a call
// started here (or from Chat) survives tab switches and shows
// live whenever this tab renders.
function voiceHomeHtml() {
  var h = "";
    if (CONFIG.backend === "live" && !voiceState && !voiceFetching) refreshVoice(true);
    if (CONFIG.backend === "live" && voiceCalls === null) refreshVoiceCalls();
    // Warm her engines as the Voice view opens, so pressing Call
    // doesn't pay the model load inside the pickup.
    warmVoice();
    var vc = voiceState ? voiceState.config : { stt_model: "small.en", tts_engine: "kokoro", tts_voice: "bf_isabella", rate: 1, pitch: 1 };
    var vEnabled = voiceState ? voiceState.enabled : true;
    var sttOk = CONFIG.backend !== "live" || (voiceState && voiceState.stt.available);
    var ttsOk = CONFIG.backend !== "live" || (voiceState && voiceState.tts.available);
    h += "<p style='color:var(--dim)'>She hears you and answers out loud — the whole loop on this machine, nothing sent anywhere. You speak; <b>faster-whisper</b> turns it into text; her model answers with her persona, CLI access, and chips intact; <b>" + (vc.tts_engine === "piper" ? "Piper" : "Kokoro") + "</b> speaks it back, sentence by sentence as it arrives.</p>";
    if (voiceMsg) h += "<div class='card' style='border-color:#f5b544;color:#f5b544'>" + esc(voiceMsg) + "</div>";
    // The call panel.
    h += "<div class='card'>";
    if (!call) {
      var callBlocked = CONFIG.backend === "live" && voiceState && (!vEnabled || !sttOk || !ttsOk);
      h += "<div style='display:flex;gap:16px;align-items:center;flex-wrap:wrap'><button class='callbtn' id='call-start' " + (callBlocked ? "disabled" : "") + " aria-label='Call Navi'><span style='display:flex;flex-direction:column;align-items:center;justify-content:center;gap:3px;height:100%'>" + phoneSvg(26) + "<span style='font-size:12px'>Call</span></span></button><div style='flex:1;min-width:220px'><b>Call Navi</b><br><span style='color:var(--dim)'>She picks up and greets you; then it's a real conversation — talk, she answers out loud, and she can run commands and chips while you talk. Ended calls are recorded in <span class='mono'>voice/calls.json</span> and filed into the active chat. Push-to-talk is also sitting beside the chat composer for one-off spoken messages — and the Call button beside it starts this same call right in the chat window.</span>" +
        (callBlocked ? "<br><span style='color:#f5b544'>" + (!vEnabled ? "Voice is turned off - switch it on in Settings → Voice." : "Her speech engines aren't installed yet — see the note above.") + "</span>" : "") +
        (llama.status !== "running" ? "<br><span style='color:var(--dim)'>Note: navi.llamafile is disconnected, so on the call she'll answer like command-line mode — running what you say as commands and reading the output back.</span>" : "") + "</div></div>";
      h += "<div style='margin-top:10px'><button class='ghost' id='voice-test' " + (CONFIG.backend === "live" && voiceState && !ttsOk ? "disabled" : "") + ">Hear her voice</button> <span style='color:var(--dim);font-size:12px'>One line, synthesized locally, so you can check the voice before calling.</span></div>";
    } else {
      h += "<div style='display:flex;gap:14px;align-items:center;flex-wrap:wrap'><span class='wave " + (callState === "speaking" ? "speaking" : callState === "listening" ? "" : "idle") + "' aria-hidden='true'><i></i><i></i><i></i><i></i><i></i></span><div><b class='mono' id='call-clock' style='font-size:18px'>" + fmtCallTime(callSeconds) + "</b><br><span style='color:var(--dim);font-size:12px'>" + (callState === "thinking" ? "Navi is thinking…" : callState === "speaking" ? "Navi is speaking…" : "Listening — push to talk, or type below") + "</span></div><button class='callbtn end' id='call-end' style='width:60px;height:60px;margin-left:auto' aria-label='End the call'>End</button></div>";
      h += "<div style='margin-top:10px;max-height:300px;overflow-y:auto' aria-label='Call transcript'>" + call.turns.map(function (t) {
        return callTurnHtml(t);
      }).join("") + "</div>";
      h += "<div style='margin-top:8px;display:flex;gap:8px;flex-wrap:wrap;align-items:center'><button class='ptt' id='call-ptt' data-ptt='call' aria-label='Push to talk: click to start talking, click again to send'>Push to talk</button><span style='color:var(--dim);font-size:12px'>or type what you'd say:</span></div>";
      h += "<div class='row' style='margin-top:8px'><input class='grow' id='call-type' type='text' placeholder='Show me the contents of /home/name/' aria-label='Type what you would say on the call'><button class='act' id='call-say'>Speak</button></div>";
    }
    h += "</div>";
    // The pipeline, with each stage's real state.
    var pipe = [["Microphone", "your voice, in the room", true], ["faster-whisper · " + vc.stt_model, "speech → text, CPU int8", !!sttOk], ["Navi · llamafile", "her model answers", llama.status === "running"], [vc.tts_engine === "piper" ? "Piper" : "Kokoro-82M", "text → her voice", !!ttsOk], ["Speaker", "her voice, in the room", true]];
    h += "<div style='display:flex;flex-wrap:wrap;gap:6px;align-items:stretch;margin-bottom:12px'>" + pipe.map(function (n, i) {
      return (i ? "<span style='align-self:center;color:var(--dim)'>→</span>" : "") + "<div class='card' style='margin:0;padding:8px 10px;border-color:" + (n[2] ? "#34d399" : "#f87171") + "'><span class='dot' style='background:" + (n[2] ? "#34d399" : "#f87171") + "'></span><b style='font-size:12px'>" + esc(n[0]) + "</b><br><span style='color:var(--dim);font-size:11px'>" + esc(n[1]) + "</span></div>";
    }).join("") + "</div>";
    // v0.4.1e (#229): the device verdicts, in the open — the
    // strip above says the engines are installed; this says
    // which hardware speech actually runs on, proven by
    // execution at warmup, and a fallen-back stage names its
    // reason in the operator's words (never a raw CUDA error).
    if (CONFIG.backend === "live" && voiceState && voiceState.tts && voiceState.tts.device) {
      var tdev = voiceState.tts.device;
      if (tdev.state === "fallen-back") h += "<div class='card' style='border-color:#f5b544'><b>Speech is on the CPU — the graphics card cannot be used.</b><br><span style='color:var(--dim)'>" + esc(tdev.detail || "") + "</span></div>";
      else if (tdev.state === "executing") h += "<p style='color:var(--dim);font-size:12px'>Speech device: <b style='color:var(--text)'>" + esc(tdev.name || "graphics card") + "</b>" + (tdev.capability ? " (compute capability " + esc(tdev.capability) + ")" : "") + " — executing, proven by a real operation at warmup.</p>";